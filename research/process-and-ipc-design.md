# FreeUnit on native Windows: process model and IPC design study

Scope: how FreeUnit's main, discovery, controller, router, prototype and worker processes, its port IPC, descriptor passing, shared memory, sender identity checks, signals and daemon mode can work in a native Windows build, and what a first version should leave out.

Evidence base, all checked on 2026-10-07:

- FreeUnit source at commit 872bf041 (origin/master, https://github.com/freeunitorg/freeunit/tree/872bf041). Code claims are cited as `path:line@872bf041`.
- Precedent source at release tags: PostgreSQL `REL_18_6`, Apache httpd `2.4.69`, nginx `release-1.31.6`, libuv `v1.53.0`, php-src `php-8.5.11`. Cited as GitHub URLs pinned to the tag.
- Microsoft documentation on learn.microsoft.com. The quoted sentences were checked against the public MicrosoftDocs sources of the same pages.

Findings carry a source. Design choices and interpretations are in the Inferences sections only.

## 1. What FreeUnit does today: the verified process and IPC map

### Takeaway

FreeUnit is a fork tree of six process types that talk over message-oriented AF_UNIX socket pairs. Descriptors cross between processes that are not parent and child at runtime: router and app workers exchange shared-memory segments, ports and request-body files, and the controller sends config blobs to its sibling, the router. Every privileged handler authorizes the sender by a kernel-supplied pid. The prototype does real pre-fork work only for PHP (module startup) and WASM (module compile); Python, Perl and Ruby set up nothing before fork.

### Cited Findings

#### 1.1 Process types, who starts them, restart policy

Types: `NXT_PROCESS_MAIN = 0`, `DISCOVERY`, `CONTROLLER`, `ROUTER`, `PROTOTYPE`, `APP` (src/nxt_process_type.h:12-17@872bf041).

| type | started by | where | restarted by main when it dies |
|---|---|---|---|
| main | the user or init | daemonizes unless `--no-daemon`: nxt_process_daemon src/nxt_process.c:1371@872bf041 (fork :1385, setsid :1413), called from src/nxt_runtime.c:399-401@872bf041; flag parsed at src/nxt_runtime.c:1334-1335@872bf041 | n/a |
| discovery | main, first | nxt_discovery_start (src/nxt_application.c:194@872bf041) loads every module (dlopen and dlsym at :413, :420), sends MODULES to main (:223) and exits on main's reply (nxt_discovery_quit :562) | no |
| controller | main, after MODULES | nxt_main_port_modules_handler src/nxt_main_process.c:2013@872bf041, nxt_process_init_start at :2174 | yes (`init.restart`, src/nxt_main_process.c:1599-1612@872bf041) |
| router | main, after MODULES | same handler, :2176 | yes |
| prototype (one per app) | main, on START_PROCESS from the router when the app has no prototype | router side src/nxt_router.c:4546-4567@872bf041; main side nxt_main_start_process_handler src/nxt_main_process.c:554@872bf041 | no |
| app worker | the app's prototype, on START_PROCESS from the router | router side src/nxt_router.c:4546, :4573-4574@872bf041; prototype side nxt_proto_start_process_handler src/nxt_application.c:742@872bf041, fork through nxt_process_start at :843 | no (the router starts new ones) |
| state store child (not a process type) | main, to write config state files off the main loop | nxt_main_store_fork src/nxt_main_process.c:2557@872bf041, fork() at :2561 | n/a |

Main writes the pid file (src/nxt_runtime.c:423@872bf041, nxt_runtime_pid_file_create at :1615) and binds the control socket itself before any child exists (nxt_runtime_controller_socket src/nxt_controller.c:748@872bf041, nxt_listen_socket_create at :798, called from src/nxt_runtime.c:1069-1071@872bf041). The controller inherits the listening socket through fork.

#### 1.2 How a process is created

- nxt_process_start (src/nxt_process.c:243@872bf041) creates the child's own port and socket pair in the parent before fork: nxt_port_new (:253), nxt_port_socket_init (:260). It runs the `prefork` hook, calls nxt_process_create (:279), and in the parent closes the read end and enables writes (:300-301).
- nxt_process_create (src/nxt_process.c:687@872bf041): fork() at :704. Child: nxt_process_unshare (:720), nxt_process_child_fixup (:727), nxt_process_setup (:733). Parent: moves the child into its cgroup (:756); if that fails it tells the child to exit with a -1 status byte (:765), or sends SIGTERM in builds without Linux namespaces (:769). It reads the real pid from a pipe when a PID namespace is used (:782) and registers the process (:817).
- PID namespace: unshare() at :574 and a second fork() at :588 inside the child; PR_SET_CHILD_SUBREAPER in the parent at :672-673 (src/nxt_process.c@872bf041).
- nxt_process_setup (src/nxt_process.c:825@872bf041) sets the title "unit: <name>" (:838), installs signals and swaps the engine. A direct child of main outside a PID namespace then enters nxt_process_do_start (:896). Every other child, including every app worker, first sends WHOAMI to main and continues from the reply (:882-885, nxt_process_whoami_ok :1037). nxt_process_do_start runs the `setup` callback. A setup that leaves state CREATED sends PROCESS_CREATED to main (nxt_process_send_created :1053); READY sends PROCESS_READY to the parent (nxt_process_send_ready :1316).

#### 1.3 What a child keeps after fork

nxt_process_child_fixup (src/nxt_process.c:326@872bf041) closes every inherited port whose type the keep matrix rejects (the parent's port is always kept, :374-375), the ports of every process that is not READY (:384-390) and the ports of the siblings in `init->siblings` (:396-400). It also destroys inherited incoming mmaps. The matrices are at src/nxt_process.c:80-105@872bf041. Rows are the type of the child (keep) or of the receiving process (send); columns are the other process type.

| row | keep matrix: ports kept | send matrix: NEW_PORT types received from broadcasts |
|---|---|---|
| main | all | all |
| discovery | main | main |
| controller | main, router | main, router |
| router | main, controller, router, prototype, app | main, controller, router, prototype, app |
| prototype | main, router | main |
| app | main, router | main |

The send matrix drives nxt_port_send_new_port (src/nxt_port.c:467@872bf041). A third matrix, `nxt_proc_remove_notify_matrix`, selects REMOVE_PID receivers.

#### 1.4 The port channel

- Socket type: `socketpair(AF_UNIX, SOCK_SEQPACKET)` when `NXT_HAVE_AF_UNIX_SOCK_SEQPACKET`, else `SOCK_DGRAM` (src/nxt_socketpair.c:16-20, :26@872bf041). Both ends non-blocking and FD_CLOEXEC (:33-47). SO_PASSCRED on both ends where available (:49-65).
- Framing: one sendmsg is one recvmsg. Records are limited to `port->max_size = min(16 KiB, SO_SNDBUF)` (src/nxt_port_socket.c:154-155, :187@872bf041); `max_share` is 64 KiB (:188). Larger payloads are split into fragments or carried in shared memory.
- One reader, many writers: the owner keeps the read end; every writer holds a duplicate of the write end `pair[1]`, delivered by NEW_PORT (src/nxt_port.c:540@872bf041).
- The raw calls are nxt_sendmsg and nxt_recvmsg (src/nxt_socket_msg.c:32, :53@872bf041). The daemon and libunit both use them.
- Kernel credentials: SCM_CREDENTIALS with `struct ucred` (Linux) or SCM_CREDS with `struct cmsgcred` (FreeBSD) (src/nxt_socket_msg.h:13-25@872bf041). Where supported, a message without credentials is rejected (src/nxt_socket_msg.h:331-336@872bf041). `NXT_USE_CMSG_PID` exists only on those platforms (src/nxt_port.h:293-295@872bf041); elsewhere, for example macOS, `nxt_recv_msg_cmsg_pid()` falls back to the self-declared `msg->port_msg.pid` (src/nxt_port.h:327-332@872bf041).

#### 1.5 Every message that carries a descriptor

Method: a scripted scan of every `nxt_port_socket_write()` and `nxt_port_socket_write2()` call whose descriptor argument is not -1, plus every libunit send with ancillary data. "Non-parent" marks pairs that are not parent and child; on Windows these are the hard cases.

| # | message | sender -> receiver | descriptors | created by | send site | pair |
|---|---|---|---|---|---|---|
| 1 | NEW_PORT | main -> all (broadcast); prototype -> main and router (its workers); router -> app (reply to GET_PORT) | port write end; port queue section | creator's socketpair; nxt_shm_open | nxt_port_send_port src/nxt_port.c:515, :540; broadcast :467; GET_PORT reply src/nxt_router.c:9283 | prototype->router and router->app are non-parent |
| 2 | NEW_PORT (libunit) | app worker -> router | per-thread context port write end; its queue section | src/nxt_unit.c:6782 (in nxt_unit_create_port :6773); queue at :6639 | nxt_unit_send_port src/nxt_unit.c:6844 | non-parent |
| 3 | PROCESS_READY (libunit) | app worker -> prototype | worker's port queue section | src/nxt_unit.c:644 | nxt_unit_ready src/nxt_unit.c:1098 (sendmsg :1115); receiver maps it at src/nxt_port.c:808-830 | parent |
| 4 | WHOAMI | app worker -> main | the worker's own port write end (sent only when the parent is not main) | nxt_process_start | src/nxt_process.c:981-985 | non-parent (grandchild to main) |
| 5 | START_PROCESS | router -> main | app shared port read end; app queue section | src/nxt_router.c:3325-3342, :3968 | src/nxt_router.c:721-722, :746-747; :4564-4567, :4604 | non-parent (siblings' parent) |
| 6 | START_PROCESS | router -> prototype | none (set to -1 at src/nxt_router.c:4573-4574) | n/a | the same calls as row 5 with `dport = app->proto_port`: src/nxt_router.c:746-747 (dport :684-688) and :4604 (dport :4546) | non-parent |
| 7 | MMAP | app worker -> router | 10 MiB + 4 KiB data segment | nxt_unit_new_mmap src/nxt_unit.c:4895 | nxt_unit_send_mmap src/nxt_unit.c:5031 | non-parent |
| 8 | MMAP (reply to GET_MMAP) | router -> app worker | router-created data segment | nxt_port_new_port_mmap src/nxt_port_memory.c:442 | nxt_router_get_mmap_handler src/nxt_router.c:9171, write :9239 | non-parent |
| 9 | REQ_BODY | router -> app worker | request body temporary file | src/nxt_router.c:8267 | nxt_router_req_headers_ack_handler src/nxt_router.c:6398, write :6474-6484 | non-parent |
| 10 | SOCKET (reply) | main -> router | bound listening socket | main binds | nxt_main_port_socket_handler src/nxt_main_process.c:1663, write :1735 | parent |
| 11 | ACCESS_LOG (reply) | main -> router | access log file | main opens | src/nxt_main_process.c:3167, write :3215 | parent |
| 12 | CHANGE_FILE | main -> every process | reopened log file | SIGUSR1 handler src/nxt_main_process.c:1371, :1433 | nxt_port_change_log_file src/nxt_port.c:1133, write :1158 | parent, also to grandchildren |
| 13 | CERT_GET reply / CERT_STORE | main -> router or controller / controller -> main | certificate file / blob section | src/nxt_cert.c:1608 (blob) | src/nxt_cert.c:1568; :1642 | parent / child |
| 14 | SCRIPT_GET reply / SCRIPT_STORE | as for certificates | njs script file / blob section | src/nxt_script.c:708 (blob) | src/nxt_script.c:667; :742 | parent / child |
| 15 | CONF_STORE | controller -> main | config blob section | src/nxt_controller.c:3083 | src/nxt_controller.c:3106 | child |
| 16 | DATA (config push) | controller -> router | config blob section | src/nxt_controller.c:702 | nxt_controller_conf_send src/nxt_controller.c:673, write :727 | non-parent (siblings) |

All rows cite @872bf041.

#### 1.6 Shared memory segments

- Creation: nxt_shm_open (src/nxt_port_memory.c:493@872bf041) uses memfd_create (:509), or shm_open(SHM_ANON) (:521), or shm_open of the name `unit.<pid>.<random>` followed at once by shm_unlink (:501, :533-545), then ftruncate (:555). libunit has its own copy, nxt_unit_shm_open (src/nxt_unit.c:4958@872bf041); its shm_open name is `unit.<pid>.<thread id>` with no random part (:4967-4968).
- Kinds and sizes, computed from the definitions:
  - Data segment: 4 KiB header plus 10 MiB data = 10,489,856 bytes, 16 KiB chunks, 640 chunks (src/nxt_port_memory_int.h:24-32@872bf041). Created by whichever side sends a large payload: the router (src/nxt_port_memory.c:442@872bf041) or an app worker (src/nxt_unit.c:4895@872bf041).
  - Port queue, `nxt_port_queue_t`: 4 + 2 x 65,544 + 16,384 x 32 = 655,380 bytes (src/nxt_port_queue.h:17-30, src/nxt_nncq.h:12, :22-29@872bf041). Created by the port owner: router engine ports (src/nxt_router.c:3996@872bf041) and libunit contexts (src/nxt_unit.c:644, :6639@872bf041).
  - App queue, `nxt_app_queue_t`: 4 + 2 x 524,296 + 131,072 x 36 = 5,767,188 bytes (src/nxt_app_queue.h:18-32, src/nxt_app_nncq.h:12-24@872bf041). One per app, created by the router (src/nxt_router.c:3968@872bf041).
  - Blobs: config, certificate and script data sent from the controller (rows 13 to 16 above).
- Validation on receipt: the object must be at least `PORT_MMAP_SIZE` (src/nxt_port_memory.c:300-316, src/nxt_unit.c:5159@872bf041). This was relaxed from an exact match after macOS 16 KiB pages made every segment fail, issue https://github.com/freeunitorg/freeunit/issues/595 (opened 2026-10-06, closed 2026-10-07). Header fields `src_pid`, `dst_pid` and `id` are copied once and checked (src/nxt_port_memory.c:335-337@872bf041).
- Position independence: request data uses `nxt_unit_sptr_t`, a `uint32_t` offset relative to the field's own address (src/nxt_unit_sptr.h:20-32@872bf041). Queues hold indices and inline bytes. MMAP messages carry `{mmap_id, chunk_id, size}` (src/nxt_port_memory_int.h:154-156@872bf041). No shared structure needs the same base address in two processes.

#### 1.7 Lock-free queues and atomics

- `src/nxt_atomic.h` has only one implementation, the GCC builtins branch `NXT_HAVE_GCC_ATOMIC`: `__sync_bool_compare_and_swap`, `__sync_lock_test_and_set`, `__sync_fetch_and_add`, `__sync_lock_release`, `__sync_or_and_fetch`, `__sync_and_and_fetch` (src/nxt_atomic.h:17-54@872bf041). `nxt_cpu_pause()` is GNU inline assembly, `pause` on x86 and `isb` on arm64 (:57-67). The macOS atomic(3) branch is a comment only.
- `nxt_atomic_int_t` is `intptr_t` and `nxt_atomic_uint_t` is `uintptr_t` (src/nxt_atomic.h:19-21@872bf041), so they stay 64-bit on Windows x64 (LLP64 changes `long`, not `intptr_t`). The ring counters are `uint32_t` (src/nxt_nncq.h:22, src/nxt_app_nncq.h:17@872bf041). The mmap header holds `nxt_pid_t src_pid, dst_pid` and an `nxt_atomic_t oosm` (src/nxt_port_memory_int.h:94-99@872bf041).
- The rings are multi-producer and multi-consumer: compare-and-swap on head, tail and entries (src/nxt_nncq.h:49, :130, :163@872bf041); the port queue counter uses fetch-add (src/nxt_port_queue.h:84, :124@872bf041); the app queue uses CAS on `notified` and on per-item `tracking` (src/nxt_app_queue.h:84, :109@872bf041).

#### 1.8 Sender identity: PR 91 and the gates built on it

- https://github.com/freeunitorg/freeunit/pull/91, "fix(port): authorize privileged IPC senders by kernel-validated PID", merged 2026-07-07. Its body lists the privileged handlers in main (bind listener, unlink socket, open access log, accept modules, store config, release certificates and scripts, write uid_map/gid_map) and states that each was "gated on `msg->port_msg.pid`, a value the sender writes itself".
- At 872bf041 the checks are: main start process, router only (src/nxt_main_process.c:590); process created (:890); whoami (:1059); remove child pid (:1212); listener bind (:1681); socket unlink (:1913); modules, discovery only (:2040); config store, controller only (:2231); access log, router only (:3188-3189); certificates (src/nxt_cert.c:1516, :1734); scripts (src/nxt_script.c:607, :789, :929); all @872bf041.
- The prototype accepts START_PROCESS only from the router's kernel pid, or, inside a PID namespace, from any sender outside it, which the kernel reports as pid 0 (nxt_proto_start_process_sender_ok src/nxt_application.c:719-735, the pid 0 case :724-726@872bf041). The port layer accepts PROCESS_READY only from the process's own pid (src/nxt_port.c:751-752@872bf041).
- The router has a per-message-type sender gate table with classes MAIN, CONTROLLER, MAIN_OR_PROTO, NOT_APP, SELF and ANY (src/nxt_router.c:487, :513, nxt_router_msg_sender_ok :1718@872bf041).
- The REMOVE_CHILD_PID handler is registered only where kernel credentials exist (src/nxt_main_process.c:946-948@872bf041).
- Control socket: on accept, AF_UNIX peers are checked with SO_PEERCRED and SO_PEERGROUPS on Linux or getpeereid on the BSDs and macOS (src/nxt_controller.c:891-962@872bf041). Allowed: uid 0, unitd's euid, `--control-user`, `--control-group` (:817-839). Non-AF_UNIX peers (TCP) return OK without a check (:903).

#### 1.9 Signals, reaping, restart and shutdown

- Delivery: "Signals are handled only via a main thread event engine work queue", through kqueue or epoll signalfd, a sigwait() thread, or a handler that posts to the engine (src/nxt_signal.c:8-20@872bf041; sigwait at :167).
- Main (src/nxt_main_process.c:152-157@872bf041): SIGHUP to a generic handler; SIGINT and SIGTERM to fast quit; SIGQUIT to graceful quit; SIGCHLD to the reaper; SIGUSR1 to log reopen. The SIGUSR1 handler (src/nxt_main_process.c:1371@872bf041) tells the router to reopen access logs (:1389), reopens `rt->log_files`, and sends the new descriptors to every process with CHANGE_FILE (:1433).
- Other processes (src/nxt_signal_handlers.c:20-26@872bf041): SIGINT and SIGTERM quit; SIGQUIT quits gracefully; SIGHUP, SIGCHLD, SIGUSR1 and SIGUSR2 go to a generic handler.
- Reaper in main (src/nxt_main_process.c:1469@872bf041): a `waitpid(-1, WNOHANG)` loop (:1487); isolation cleanup (:1548); QUIT to the dead process's children (:1570); REMOVE_PID to the others (:1585); restart when `init.restart` is set (:1599-1612). The state store child is handled first (:1511, :1541).
- Prototypes reap their own workers (waitpid at src/nxt_application.c:1153@872bf041, in nxt_proto_sigchld_handler :1138) and report each exit to main with REMOVE_CHILD_PID (src/nxt_application.c:1466, :1489-1490@872bf041), handled at src/nxt_main_process.c:1186@872bf041.
- There is no PR_SET_PDEATHSIG in src (search at 872bf041). A worker leaves its loop on any port error: nxt_unit_run quits with NXT_QUIT_NORMAL on NXT_UNIT_ERROR (src/nxt_unit.c:5812, :5827@872bf041).

#### 1.10 How the app configuration is applied in the child

In the prototype, nxt_proto_setup (src/nxt_application.c:569@872bf041) runs in this order: load the module (:593), set `environment` (:599), run the module `setup` (:606-607), prepare and change the root (:616, :622), `chdir(working_directory)` (:632), set state CREATED (:642). Main then writes uid/gid maps on PROCESS_CREATED, and nxt_process_created_ok (src/nxt_process.c:1094@872bf041) calls nxt_process_apply_creds (:1257): setgroups, setgid and setuid through src/nxt_credential.c:224-348@872bf041, then PR_SET_NO_NEW_PRIVS (src/nxt_process.c:1286-1288@872bf041). So module setup runs before the credential drop and before chroot. Workers inherit all of it through fork. The worker's `setup`, nxt_app_setup, calls the module `start` directly (src/nxt_application.c:1661@872bf041). Config keys are validated in src/nxt_conf_validation.c:1342-1357@872bf041 (`user`, `group`, `working_directory`, `environment`).

#### 1.11 What the prototype does before fork, per language

| module | `setup` in the prototype | `start` in the worker |
|---|---|---|
| PHP | ZTS TSRM startup, `sapi_startup`, then `php_module_startup` through nxt_php_startup (src/nxt_php_sapi.c:393-445, :1400-1405@872bf041) | nxt_unit_init and nxt_unit_run (src/nxt_php_sapi.c:476, :530, :537@872bf041) |
| WASM (wasmtime) | creates engine and store and compiles the module with `wasmtime_module_new` (src/wasm/nxt_wasm.c:389-444, src/wasm/nxt_rt_wasmtime.c:490-527@872bf041) | runs requests |
| WASM component | stores a global config (src/wasm-wasi-component/src/lib.rs:70-111@872bf041) | runs requests |
| Java | resolves the jar directory only (nxt_java_setup src/nxt_java.c:78-141, realpath at :122@872bf041) | builds the class path and creates the JVM |
| Python, Perl, Ruby | `setup` is NULL (src/python/nxt_python.c:49-57, src/perl/nxt_perl_psgi.c:111-119, src/ruby/nxt_ruby.c:93-101@872bf041) | initialises the interpreter after fork |
| external (Go, Node) | built into unitd (src/nxt_external.c:18@872bf041) | execs the user's binary |

#### 1.12 External applications: an existing exec path with inherited descriptors

nxt_external_start (src/nxt_external.c:60@872bf041) clears FD_CLOEXEC on six descriptors: the prototype port write end, the router port write end, both ends of the worker's own port, the shared port and the shared queue (:89-114). It then sets `NXT_UNIT_INIT` to `VERSION;stream;proto_pid,proto_id,proto_fd;router_pid,router_id,router_fd;my_pid,my_id,my_in_fd,my_out_fd;shared_port_fd,shared_queue_fd;2,shm_limit,request_limit` (:121-134, setenv :144) and calls execve (:201). libunit reads it with getenv (src/nxt_unit.c:1012@872bf041).

#### 1.13 The module contract and what a module imports from unitd

- `struct nxt_app_module_s` has `compat_length`, `compat`, `type`, `version`, `mounts`, `nmounts`, `setup`, `start` (src/nxt_application.h:177@872bf041). Each module exports `NXT_EXPORT nxt_app_module_t nxt_app_module` (for example src/nxt_php_sapi.c:369@872bf041).
- Loading: discovery uses `dlopen(name, RTLD_GLOBAL | RTLD_NOW)` and `dlsym(dl, "nxt_app_module")` (src/nxt_application.c:413, :420@872bf041); the prototype uses `RTLD_GLOBAL | RTLD_LAZY` (:1519, :1528). There is no src/nxt_dyld.c at 872bf041.
- Linking: on Linux unitd links with `-Wl,-E` ("exports symbols of executable file", auto/os/conf:30-31@872bf041) and modules with `-shared` (:28); on macOS modules use `-dynamiclib -undefined dynamic_lookup` (:113). The daemon compiles with `-fvisibility=hidden` (auto/cc/test:66@872bf041) and `NXT_EXPORT` is `visibility("default")` (src/nxt_clang.h:123@872bf041), so only NXT_EXPORT symbols are visible to modules.
- `libunit.a` holds nxt_lvlhsh.o, nxt_murmur_hash.o, nxt_socket_msg.o, nxt_websocket.o and nxt_unit.o (auto/make:120-130, :166-170@872bf041). The PHP module links only nxt_unit.o (auto/modules/php:604, :645-646@872bf041).
- Measured on a local debug build of 2026-09-15 (older than 872bf041, so treat as indicative): `php.unit.so` has 18 undefined `nxt_` symbols, all resolved from unitd at load time: `nxt_conf_get_object_member`, `nxt_conf_get_string`, `nxt_conf_next_object_member`, `nxt_conf_object_members_count`, `nxt_lvlhsh_delete`, `nxt_lvlhsh_find`, `nxt_lvlhsh_insert`, `nxt_lvlhsh_retrieve`, `nxt_malloc`, `nxt_zalloc`, `nxt_murmur_hash2`, `nxt_recvmsg`, `nxt_sendmsg`, `nxt_server`, `nxt_unit_default_init`, `nxt_websocket_frame_header_size`, `nxt_websocket_frame_init`, `nxt_websocket_frame_payload_len` (output of `nm -D --undefined-only`). The unitd of that build exported 291 `nxt_` symbols.

#### 1.14 libunit's waiting model and event-loop integrations

- nxt_unit_read_buf (src/nxt_unit.c:5899@872bf041) reads in this order: the context port's socket when the context waits for items or is not ready (:5937-5938), the context port's own queue (:5949), the app queue (:5971), and only then poll() on the context read port and the shared port (:5991). Every worker process polls the same shared port: the shared port is a competing-consumer doorbell.
- The Node.js module waits with `uv_poll_init(loop, &data->poll, port->in_fd)` (src/nodejs/unit-http/unit.cpp:586@872bf041). Python ASGI uses asyncio `add_reader` on port descriptors (src/python/nxt_python_asgi.c:42, :271, :1035@872bf041).

#### 1.15 Upstream position on Windows

- nginx/unit#604 "Native Windows build" (opened 2021-11-26, closed 2022-06-19). The Unit maintainer VBart replied on 2021-11-27: "Unit doesn't support native Windows interfaces and never will. It has architecture based on POSIX principles and highly tuned for *nix kernels. There's just no way to run it on Windows without WSL2." (https://github.com/nginx/unit/issues/604)
- nginx/unit#1008 (open since 2023-11-23) reports a failure to cross-compile a Linux Go application on a Windows host with `CGO_ENABLED=1 GOOS=linux` (error in runtime/cgo `linux_syscall.c`). It is not about running Unit on Windows (https://github.com/nginx/unit/issues/1008).

#### 1.16 Briefing corrections

- There is no src/nxt_dyld.c. Modules are loaded with dlopen in src/nxt_application.c:413 and :1519@872bf041.
- "The prototype loads the module once, runs module setup such as php_module_startup, then forks workers": true for PHP and WASM. For Python, Perl and Ruby `setup` is NULL, and Java's setup only builds a class path (1.11).
- "Main reaps and restarts children": main restarts only the controller and the router. Prototypes reap their own workers and report with REMOVE_CHILD_PID (1.9). Main also forks a state store child that the briefing did not mention (1.1).
- Modules are not fully self-contained. They link only nxt_unit.o and resolve lvlhsh, murmur hash, socket-message, websocket, allocator and config-API symbols from unitd at load time (1.13). This decides the Windows module link model.
- Message boundaries come from SOCK_SEQPACKET or SOCK_DGRAM, never a stream (1.4).
- The incoming segment size check is now "at least PORT_MMAP_SIZE", not "equal" (1.6).
- nginx/unit#1008 is a cross-compilation report, not a Windows port request (1.15).
- Stale in the August 2026 internal notes: the port dispatcher now closes every descriptor a handler leaves set; a handler that keeps one sets its slot to -1 (src/nxt_port.h:340-346@872bf041).

### Inferences

Current topology, as verified above:

```
main (root; binds listeners and the control socket; opens logs; writes state)
 |-- discovery        dlopen every module, report MODULES, exit
 |-- controller       control API; config blobs -> main (store) and -> router (push)
 |-- router           N engine threads, one port each; app queues; data segments
 |-- prototype "app"  module setup (PHP startup, WASM compile), fork per worker
 |     |-- worker 1   module start; libunit contexts; MMAP <-> router
 |     |-- worker n
 |-- state store child (short-lived)

shared write ends:  every peer of a port writes into the same socket
identity:           kernel pid on each message (SCM_CREDENTIALS / SCM_CREDS)
descriptor passing: SCM_RIGHTS, including router <-> worker and controller -> router
```

- fork() is not the main obstacle. The obstacles are: shared write ends that need per-message kernel identity; descriptor passing between non-parent processes at runtime (rows 1, 2, 4, 5, 7, 8, 9 and 16 of 1.5); and readiness-based waiting in both the daemon engines and libunit (1.14).
- Losing fork costs little for Python, Perl, Ruby and Java, because their real start-up already happens after fork. PHP loses its pre-forked module startup and WASM loses its pre-compiled module.
- FreeUnit's shared memory is position-independent (1.6), so it avoids the fixed-address reattach problem that PostgreSQL, nginx and PHP opcache have on Windows (section 3).
- The import surface from unitd into a module is small and already marked by NXT_EXPORT. A Windows export list can be generated from the same annotation.

### Gaps

- Imports of the Python, Perl, Ruby, Java and WASM modules from unitd were not measured; no build artifacts were available.
- The remove-notify matrix was not re-derived row by row.

## 2. Windows architecture proposal: one decision per mechanism

### Takeaway

Re-execute `unitd.exe` once per role, with an explicit list of inherited handles, and keep FreeUnit's process tree including the prototype. Replace each socket pair with a message-mode named pipe that has one instance per writer, an unguessable name and a server identity check; the instance's kernel-reported client pid replaces SCM_CREDENTIALS, and queue markers name the writer. Pass shared memory as named sections in a per-instance private namespace and the remaining files and sockets by DuplicateHandle or WSADuplicateSocket, always performed by the more trusted process. Supervise children with per-process waits posted to the engine and a kill-on-close job. Map signals to console, service and control-pipe events. Ship a user-level console process first.

### Cited Findings

#### 2.1 Process creation and handle inheritance

- CreateProcessW: "If this parameter is TRUE, each inheritable handle in the calling process is inherited by the new process." and "inherited handles have the same value and access rights as the original handles." (https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessw)
- Handle Inheritance: "An inherited handle refers to the same object in the child process as it does in the parent process. It also has the same value and access privileges." The parent usually tells the child the values "through its command line, environment block, or some form of interprocess communication" (https://learn.microsoft.com/en-us/windows/win32/procthread/inheritance).
- PROC_THREAD_ATTRIBUTE_HANDLE_LIST: "a list of handles to be inherited by the child process. These handles must be created as inheritable handles and must not include pseudo handles", and with it "pass in a value of TRUE for the bInheritHandles parameter". UpdateProcThreadAttribute requires Windows Vista or Windows Server 2008. PROC_THREAD_ATTRIBUTE_JOB_LIST is "Supported in Windows 10 and newer and Windows Server 2016 and newer". PROC_THREAD_ATTRIBUTE_SECURITY_CAPABILITIES creates an AppContainer process, Windows 8 and Windows Server 2012 and newer (https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-updateprocthreadattribute).
- Raymond Chen, "Programmatically controlling which handles are inherited by new processes in Win32", 2011-12-16 (https://devblogs.microsoft.com/oldnewthing/20111216-00/?p=8873).
- Process creation flags: CREATE_NEW_PROCESS_GROUP: "If this flag is specified, CTRL+C signals will be disabled for all processes within the new process group." DETACHED_PROCESS: "the new process does not inherit its parent's console" (https://learn.microsoft.com/en-us/windows/win32/procthread/process-creation-flags).

#### 2.2 Duplicating handles and sockets

- DuplicateHandle: both the source and the target process handles "must have the PROCESS_DUP_HANDLE access right". It "can be called by either the source process or the target process". "If the process that calls DuplicateHandle is not also the target process, the source process must use interprocess communication to pass the value of the duplicate handle to the target process." File mappings, events, semaphores, pipes, files, jobs and processes can be duplicated. Sockets must not be: "To duplicate a socket handle, use the WSADuplicateSocket function". I/O completion ports cannot be duplicated: "No error is returned, but the duplicate handle cannot be used". A duplicated handle can be closed remotely with DUPLICATE_CLOSE_SOURCE. Minimum Windows 2000 (https://learn.microsoft.com/en-us/windows/win32/api/handleapi/nf-handleapi-duplicatehandle).
- Process Security and Access Rights: "if process A has a handle to process B with PROCESS_DUP_HANDLE access, it can duplicate the pseudo handle for process B. This creates a handle that has maximum access to process B." Also: "The handle returned by the CreateProcess function has PROCESS_ALL_ACCESS access to the process object", and the default process DACL comes "from the primary or impersonation token of the creator" (https://learn.microsoft.com/en-us/windows/win32/procthread/process-security-and-access-rights).
- WSADuplicateSocketW takes the "Process identifier of the target process in which the duplicated socket will be used". The source passes the WSAPROTOCOL_INFO by IPC to the target, which calls WSASocket. "The special WSAPROTOCOL_INFO structure can only be used once by the target process." (https://learn.microsoft.com/en-us/windows/win32/api/winsock2/nf-winsock2-wsaduplicatesocketw)

#### 2.3 Job objects and waiting for exit

- JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE (0x00002000) "Causes all processes associated with the job to terminate when the last handle to the job is closed" (https://learn.microsoft.com/en-us/windows/win32/api/winnt/ns-winnt-jobobject_basic_limit_information).
- Children join the job automatically: "After a process is associated with a job, by default any child processes it creates using CreateProcess are also associated with the job", unless BREAKAWAY_OK plus CREATE_BREAKAWAY_FROM_JOB, or SILENT_BREAKAWAY_OK, is set (https://learn.microsoft.com/en-us/windows/win32/procthread/job-objects).
- "A process can be associated with more than one job starting in Windows 8 and Windows Server 2012." (https://learn.microsoft.com/en-us/windows/win32/api/jobapi2/nf-jobapi2-assignprocesstojobobject)
- Job completion-port messages: "except for limits set with the JobObjectNotificationLimitInformation information class, messages are intended only as notifications and their delivery to the completion port is not guaranteed." For messages carrying a pid, "you cannot guarantee that this process is still active or that the identifier has not been recycled" (https://learn.microsoft.com/en-us/windows/win32/api/winnt/ns-winnt-jobobject_associate_completion_port).
- RegisterWaitForSingleObject "Directs a wait thread in the thread pool to wait on the object" and queues a callback when it is signaled (https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-registerwaitforsingleobject).
- WaitOnAddress wakes only on a change made by "another thread in the same process"; minimum Windows 8 (https://learn.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitonaddress). Cross-process wake-ups need kernel objects or messages.

#### 2.4 Named pipes

- PIPE_TYPE_MESSAGE: "The pipe treats the bytes written during each write operation as a message unit. The GetLastError function returns ERROR_MORE_DATA when a message is not read completely." The client end "starts out in byte mode, even if the server side is in message mode", so a client that reads must switch to message read mode. nMaxInstances is 1 to PIPE_UNLIMITED_INSTANCES (255); 255 means limited only by resources. FILE_FLAG_FIRST_PIPE_INSTANCE: "creation of the first instance succeeds, but creation of the next instance fails with ERROR_ACCESS_DENIED". PIPE_REJECT_REMOTE_CLIENTS: "Connections from remote clients are automatically rejected." Default security: "The ACLs in the default security descriptor for a named pipe grant full control to the LocalSystem account, administrators, and the creator owner. They also grant read access to members of the Everyone group and the anonymous account." (https://learn.microsoft.com/en-us/windows/win32/api/namedpipeapi/nf-namedpipeapi-createnamedpipew)
- GetNamedPipeClientProcessId "Retrieves the client process identifier for the specified named pipe"; minimum Windows Vista and Windows Server 2008 (https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-getnamedpipeclientprocessid).
- Clients: "If CreateFile opens the client end of a named pipe, the function uses any instance of the named pipe that is in the listening state." Unless the client sets quality-of-service flags, the server may impersonate it: SECURITY_IMPERSONATION "is the default behavior if no other flags are specified along with the SECURITY_SQOS_PRESENT flag." (https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilew)
- Per-session namespaces exist for "events, semaphores, mutexes, waitable timers, file-mapping objects, job objects, and symbolic link objects" (https://learn.microsoft.com/en-us/windows/win32/termserv/kernel-object-namespaces). Named pipes are not in that list.
- Pipe names: the name part "can include any character other than a backslash", and "The entire pipe name string can be up to 256 characters long" (https://learn.microsoft.com/en-us/windows/win32/ipc/pipe-names).
- Anonymous pipes: "Asynchronous (overlapped) read and write operations are not supported by anonymous pipes." and "Anonymous pipes are implemented using a named pipe with a unique name." (https://learn.microsoft.com/en-us/windows/win32/ipc/anonymous-pipe-operations)

#### 2.5 AF_UNIX on Windows

- Announced 2017-12-19: "Beginning in Insider Build 17063". Only SOCK_STREAM. Unsupported: "AF_UNIX datagram (SOCK_DGRAM) or sequence packet (SOCK_SEQPACKET) socket type", ancillary data including SCM_RIGHTS and SCM_CREDENTIALS, autobind, and "socketpair socket API is not supported in Winsock 2.0". Permissions: creating the socket file needs write permission on the directory, and "for connecting to a stream socket, the connecting process should have write permission on the socket." (https://devblogs.microsoft.com/commandline/af_unix-comes-to-windows/)
- `afunix.h` (mingw-w64 v14.x copy of the SDK header): `UNIX_PATH_MAX 108`; `SIO_AF_UNIX_GETPEERPID _WSAIOR(IOC_VENDOR, 256)`; `SIO_AF_UNIX_SETBINDPARENTPATH` (257); `SIO_AF_UNIX_SETCONNPARENTPATH` (258) (https://mingw.googlesource.com/mingw-w64/+/refs/heads/v14.x/mingw-w64-headers/include/afunix.h). A Wine bug filed on 2026-08-20 describes the peer-pid ioctl as "roughly equivalent to getsockopt(SO_PEERCRED)" (https://list.winehq.org/hyperkitty/list/wine-bugs@list.winehq.org/thread/D5XYEW4MKQX2RCOLQVOMH6I2HBCXGDGY/). It is a third-party feature request and the only description found; it does not establish which pid Windows reports after the socket is passed to another process.
- libuv does not use AF_UNIX on Windows; "AF_UNIX on Windows" has been an open issue since 2019-11-10 (https://github.com/libuv/libuv/issues/2537).
- libuv: "On windows only sockets can be polled with poll handles." (https://docs.libuv.org/en/v1.x/poll.html). asyncio on Windows: SelectorEventLoop `add_reader` "only accept[s] socket handles", while ProactorEventLoop, the default, does not support `add_reader` at all (https://docs.python.org/3/library/asyncio-platforms.html).

#### 2.6 Shared memory and object namespaces

- CreateFileMapping with INVALID_HANDLE_VALUE creates an object "backed by the system paging file". "If the object exists before the function call, the function returns a handle to the existing object (with its current size, not the specified size), and GetLastError returns ERROR_ALREADY_EXISTS." "Creating a file mapping object in the global namespace from a session other than session zero requires the SeCreateGlobalPrivilege privilege." "Mapped views of a file mapping object maintain internal references to the object, and a file mapping object does not close until all references to it are released." (https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-createfilemappingw)
- MapViewOfFile: the offset "must be a multiple of the VirtualAlloc allocation granularity", read from GetSystemInfo; for the view size "All bytes must be within the maximum size specified by CreateFileMapping" (https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-mapviewoffile).
- File mappings: "The ACLs in the default security descriptor for a file-mapping object come from the primary or impersonation token of the creator." (https://learn.microsoft.com/en-us/windows/win32/memory/file-mapping-security-and-access-rights)
- Namespaces: "each session has a separate namespace for these objects"; the `Global\` prefix selects the global namespace; "Service applications use the global namespace by default." (https://learn.microsoft.com/en-us/windows/win32/termserv/kernel-object-namespaces)
- Private namespaces need a boundary descriptor (CreateBoundaryDescriptor, AddSIDToBoundaryDescriptor). "The caller must be within the specified boundary for the create operation to succeed." "The system supports multiple private namespaces with the same name, as long as they specify different boundaries." (https://learn.microsoft.com/en-us/windows/win32/sync/object-namespaces). CreatePrivateNamespaceW requires Windows Vista or Windows Server 2008 (https://learn.microsoft.com/en-us/windows/win32/api/namespaceapi/nf-namespaceapi-createprivatenamespacew).
- Windows ASLR: "each DLL or EXE image gets assigned a random load address by the kernel the first time it is used, and as additional instances of the DLL or EXE are loaded, they receive the same load address. If all instances of an image are unloaded and that image is subsequently loaded again, the image may or may not receive the same base address; see Fact 4. Only rebooting can guarantee fresh base addresses for all images systemwide." (Mandiant, 2020-03-17, https://cloud.google.com/blog/topics/threat-intelligence/six-facts-about-address-space-layout-randomization-on-windows)

#### 2.7 Console events, CRT signals and services

- CRT `signal()`: "The SIGILL and SIGTERM signals aren't generated under Windows." (https://learn.microsoft.com/en-us/cpp/c-runtime-library/reference/signal)
- Console handlers: CTRL_CLOSE_EVENT is sent "to all processes attached to a console when the user closes the console". The Timeouts table gives it `SPI_GETHUNGAPPTIMEOUT`, 5000 ms, after which the system ends the process (https://learn.microsoft.com/en-us/windows/console/handlerroutine). "By default, these signals are passed to all console processes that are attached to the console." and "CTRL+BREAK is always treated as a signal" (https://learn.microsoft.com/en-us/windows/console/ctrl-c-and-ctrl-break-signals).
- GenerateConsoleCtrlEvent with CTRL_C_EVENT: "This signal cannot be limited to a specific process group." (https://learn.microsoft.com/en-us/windows/console/generateconsolectrlevent)
- Services: StartServiceCtrlDispatcher must be called "as soon as possible after it starts up (within 30 seconds)"; as a console program it fails with ERROR_FAILED_SERVICE_CONTROLLER_CONNECT (https://learn.microsoft.com/en-us/windows/win32/api/winsvc/nf-winsvc-startservicectrldispatcherw). The control handler must return within 30 seconds. SERVICE_CONTROL_SHUTDOWN gets "about 20 seconds". SERVICE_CONTROL_PRESHUTDOWN blocks shutdown up to a configured time-out. Control codes 128 to 255 are service-defined (https://learn.microsoft.com/en-us/windows/win32/api/winsvc/nc-winsvc-lphandler_function_ex).

#### 2.8 Identity and isolation

- CreateProcessAsUser: "Typically, the process that calls the CreateProcessAsUser function must have the SE_INCREASE_QUOTA_NAME privilege and may require the SE_ASSIGNPRIMARYTOKEN_NAME privilege if the token is not assignable." But "If hToken is a restricted version of the caller's primary token, the SE_ASSIGNPRIMARYTOKEN_NAME privilege is not required." (https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessasuserw)
- AppContainer (UWP) processes do not receive TCP/IP loopback traffic by default; `CheckNetIsolation.exe LoopbackExempt` adds exemptions for outbound (`-a`) and inbound server (`-is`) cases (https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/troubleshooting-uwp-firewall). The rule concerns network traffic; named pipes and sections are outside it.

#### 2.9 Language runtimes and toolchains

- PHP for Windows 8.5 is built with Visual Studio 2022 (VS17). Each build ships as Zip, Debug Pack and "Development package (SDK to develop PHP extensions)". Guidance: "NTS builds are for single-threaded use cases, typically PHP running via FastCGI or on the CLI. TS builds support multithreaded SAPIs and are intended for PHP loaded as a web server module." (https://www.php.net/downloads.php?os=windows)
- The embed SAPI is off by default on Windows and builds a static `php<version>embed.lib` (`ARG_ENABLE('embed', 'Embedded SAPI library', 'no')`, https://github.com/php/php-src/blob/php-8.5.11/sapi/embed/config.w32#L3-L8). FreeUnit's PHP module never calls the embed API; it defines its own SAPI (no "embed" in src/nxt_php_sapi.c@872bf041). The Unix build only uses the embed configure option to obtain libphp (auto/modules/php:142@872bf041).
- opcache on Windows: `opcache.mmap_base` exists because "All PHP processes have to map shared memory into the same address space". `opcache.file_cache_fallback` (default 1, Windows only) "Implies opcache.file_cache_only=1 for a certain process that failed to reattach to shared memory". `opcache.cache_id` (Windows only, since PHP 7.4.0): "all processes running the same PHP SAPI under the same user account having the same cache ID share a single OPcache instance" (https://www.php.net/manual/en/opcache.configuration.php).
- Python: "On Windows, extensions that use a Stable ABI should be linked against python3.dll rather than a version-specific library such as python39.dll." (https://docs.python.org/3/c-api/stable.html)
- node-gyp's delay-load hook exists so that an addon linked against the host executable "work[s] when the host executable is renamed" (https://github.com/nodejs/node-gyp/blob/main/src/win_delay_load_hook.cc).
- SQLite extensions receive a `const sqlite3_api_routines *pApi` table and bind it with `SQLITE_EXTENSION_INIT2(pApi)` (https://www.sqlite.org/loadext.html).
- wasmtime: `x86_64-pc-windows-msvc` is Tier 1, `x86_64-pc-windows-gnu` Tier 2, `aarch64-pc-windows-msvc` Tier 3 (https://docs.wasmtime.dev/stability-tiers.html).
- MSVC C11 atomics arrived in Visual Studio 2022 17.5 Preview 2 behind `/experimental:c11atomics`, and `__STDC_NO_ATOMICS__` stays defined until locking atomics are implemented (https://devblogs.microsoft.com/cppblog/c11-atomics-in-visual-studio-2022-version-17-5-preview-2/). In November 2025, PostgreSQL developers still needed the flag on Visual Studio 2022 (pgsql-hackers thread "Trying out <stdatomic.h>", mirrored at https://hackorum.dev/topics/52632).

### Inferences

#### 2.A Target and shape of v1

- Target Windows 10 version 1809 and Windows Server 2019 or later, x64 only. Every API chosen below is available there (Vista-era pipe and namespace APIs; nested jobs from Windows 8; PROC_THREAD_ATTRIBUTE_JOB_LIST from Windows 10; AF_UNIX from build 17063, used only for the control socket).
- First deliverable: a user-level console process for local development. It needs no administrator rights and no SCM registration. Service mode is designed here but ships later.
- Trust model for v1: all FreeUnit processes run as the same user at the same integrity level, so a worker can open the router process anyway (the default process DACL comes from the creator token, 2.2). v1 therefore protects against other users and other sessions, not against a compromised worker. Pipe names are not per-session (2.4), so that protection needs unguessable names, client-side impersonation limits and a server identity check (D3, D9). One rule keeps a later hardening possible: **only the more trusted process ever duplicates a handle into or out of a less trusted one.** No worker ever needs PROCESS_DUP_HANDLE on the router, the prototype or main.

#### 2.B Diagrams

Process tree on Windows:

```
unitd.exe                      main: console (v1) or service (later)
|                              creates job J (KILL_ON_JOB_CLOSE, handle not inheritable)
|                              and private namespace "FreeUnit-<instance>"
|-- unitd.exe --role discovery          LoadLibrary each *.unit.dll, report MODULES, exit
|-- unitd.exe --role controller         control API on an AF_UNIX socket file
|-- unitd.exe --role router             listeners, HTTP engines, app queues
|-- unitd.exe --role prototype --app A  loads module A, runs setup, keeps it resident,
|     |                                 spawns and reaps workers of A
|     |-- unitd.exe --role app --app A  loads module A, runs setup, then start
|     |-- unitd.exe --role app --app A
|-- (state store: a thread in main, was a forked child)

every process: CreateProcessW(unitd.exe, "--role ...", bInheritHandles=TRUE,
               STARTUPINFOEXW + PROC_THREAD_ATTRIBUTE_HANDLE_LIST, CREATE_SUSPENDED)
               -> write bootstrap block -> ResumeThread
               main assigns its own children to job J; workers join J automatically
```

Channels and objects:

```
port P of process X  =  named pipe server  \\.\pipe\fu-<secret>-<pid>-<port id>
                        <secret>: at least 128 random bits from the bootstrap block
                        message mode, overlapped, one instance per writer,
                        DACL: unitd user + SYSTEM only, PIPE_REJECT_REMOTE_CLIENTS
writer Y -> port P   =  Y opens its own client instance; X records
                        GetNamedPipeClientProcessId() for that instance = sender pid;
                        Y checks GetNamedPipeServerProcessId() = X before writing;
                        a READ_SOCKET marker in X's queue names Y's connection

router  --requests-->  app queue section (unchanged ring)  --+
router  --doorbell-->  semaphore S_app (one per app)  --------+--> workers of the app compete
worker  --response-->  client instance on router engine pipe R_j
worker <--> router     data segments: sections in the private namespace,
                       name "seg-<src pid>-<dst pid>-<id>-<random>", sent in the MMAP record
port queues            sections in the private namespace; READ_QUEUE stays a pipe record
files and sockets      DuplicateHandle or WSADuplicateSocket, done by main, the router
                       or the prototype, never by a worker
```

#### 2.C Decisions

**D1. Spawning: re-execute `unitd.exe` with a role argument.**
- Decision: CreateProcessW on our own image with `--role <type>` and, for apps, `--app <name>`. Use STARTUPINFOEXW with PROC_THREAD_ATTRIBUTE_HANDLE_LIST naming exactly the handles the child needs, `bInheritHandles = TRUE`, and CREATE_SUSPENDED. Main assigns its own children to job J (or passes PROC_THREAD_ATTRIBUTE_JOB_LIST); workers, created by a prototype that holds no handle to J, join J automatically (2.3). Then the parent writes the bootstrap block and calls ResumeThread. Set `lpCurrentDirectory` from `working_directory` and `lpEnvironment` from `environment` (CREATE_UNICODE_ENVIRONMENT).
- Bootstrap block: an inheritable unnamed section whose handle value is printed on the command line. It holds the role, name, stream, the parent's pid, the serialized parts of `nxt_process_init_t`, the app config JSON, and the handle values or object names of the channels. This is PostgreSQL's pattern (section 3).
- External apps (`type: external`): Windows has no exec. A unitd shim cannot turn into the user's binary under the same pid, and a pipe instance opened by the shim would keep reporting the shim's pid (2.4), while every identity gate keys on the pid registered at spawn (1.8). So the prototype starts the user's binary directly with CreateProcessW, and `NXT_UNIT_INIT` carries pipe and section names, plus inherited handle values only for unnamed objects such as the D4 semaphore. Those values stay valid because "inherited handles have the same value" (2.1).
- Alternatives: inheriting every inheritable handle without a list, which is racy and was fixed in libuv only in 2026 (section 3); an environment variable only, as nginx does, which is too small for app configs; DuplicateHandle into the suspended child, which PostgreSQL uses for single handles and is kept for handles that must never be inheritable.
- Risk: process creation cost. No credible measurement was found (section 5).

**D2. Process tree and the prototype: keep the prototype as a module-resident spawner (option a).**
- Decision: the prototype is re-executed, loads the module and runs `setup` exactly as on Unix (1.10, 1.11), then spawns each worker with D1. Each worker loads the module, runs `setup` again and then `start`.
- Why: 49 lines of src/nxt_router.c@872bf041 match `NXT_PROCESS_PROTOTYPE|proto_port|nxt_proto`, libunit's ready port is the prototype's port (1.12), and worker exits already flow through the prototype (1.9). Keeping the prototype leaves all of that unchanged. A resident prototype also keeps `php8*.dll` loaded, so its randomized base stays fixed; an image unloaded everywhere "may or may not receive the same base address" when loaded again (2.6). PHP opcache reattaches only when `execute_ex` sits at the creator's address and the stored base range is free in the new process (section 3). The prototype secures the first condition, not the second, and it keeps the opcache mapping alive between worker restarts. Python, Perl and Ruby lose nothing, because their `setup` is NULL.
- Cost: every PHP worker pays `php_module_startup`; every WASM worker compiles its module again (mitigation later: wasmtime module serialization, not verified here).
- Rejected (b), one worker with N threads on a ZTS PHP: FreeUnit's PHP module runs one context per worker (src/nxt_php_sapi.c:530-537@872bf041); php.net reserves TS builds for "PHP loaded as a web server module" (2.9). It would be a different concurrency model with its own module work. Keep it as a later memory option.
- Rejected (c), main spawns workers directly: it changes the router's start path and the REMOVE_CHILD_PID flow, for no gain beyond one process per app.

**D3. Port channel transport: message-mode named pipes, one instance per writer.**
- Decision: each port owner creates a pipe server named `\\.\pipe\fu-<secret>-<pid>-<port id>`, where `<secret>` is at least 128 random bits generated by main and handed down in the bootstrap block. Flags: PIPE_TYPE_MESSAGE, PIPE_READMODE_MESSAGE, FILE_FLAG_OVERLAPPED, PIPE_REJECT_REMOTE_CLIENTS, FILE_FLAG_FIRST_PIPE_INSTANCE on the first instance, and an explicit DACL (the unitd user SID and SYSTEM). Each writer process opens its own client instance with SECURITY_SQOS_PRESENT | SECURITY_IDENTIFICATION and checks GetNamedPipeServerProcessId against the pid announced in NEW_PORT before its first write; a client that also reads switches to message read mode. Records keep the current layout and the 16 KiB `max_size`; the fragment layer still handles larger messages. ERROR_MORE_DATA on a read is a protocol error. The owner keeps one spare instance pending in ConnectNamedPipe, like a listening socket.
- Why: message mode keeps the one-write-is-one-record property that both the daemon and libunit read paths depend on (1.4), so no framing layer has to be added to libunit, which is compiled into every module. Each connected instance has a kernel-reported client pid (2.4), which gives PR 91's checks their pid. Pipes are libuv's local IPC on Windows, and libuv's IPC pipes already identify a peer by GetNamedPipeClientProcessId and GetNamedPipeServerProcessId (section 3).
- Required change: the "many writers share one write end" model (1.4) becomes "one instance per writer". NEW_PORT then stops carrying a descriptor and carries only `{pid, port id, type}` plus the port queue's section name, because peers connect by name.
- Required change, ordering between queue and pipe: a message that cannot go into a port's queue (it carries a descriptor, a buffer chain or more than the inline size) goes to the socket and leaves a one-byte READ_SOCKET marker in the queue, and the reader then takes "the next datagram" from the one socket (src/nxt_port_socket.c:358-405, :2020; src/nxt_unit.c:7440-7447, :7497-7500@872bf041). With one instance per writer there is no single next datagram, and IOCP completions from different instances arrive in any order. Fix: the marker names the writer's connection (a queue slot holds 31 bytes; the marker uses one), and the reader, on each marker, consumes the next record of that connection and holds back records that arrive before their marker. Per-connection FIFO order plus queue order then keeps today's guarantee: "A datagram is never read after a queue message that its sender sent later" (src/nxt_unit.c:7444-7446@872bf041).
- Conditional alternative: AF_UNIX SOCK_STREAM with a length prefix. Choose it if the event-engine study (out of scope here) picks a readiness engine instead of IOCP, or if selector-style integrations must keep working: uv_poll and asyncio selectors accept sockets only (2.5). Costs: framing and partial-write queues in libunit and the daemon; an emulated socketpair with a connect race; a peer-pid ioctl that has no learn.microsoft.com page and no stated minimum build (2.5).
- Rejected: anonymous pipes (no overlapped I/O, 2.4); loopback TCP (no peer identity, exposed to every local user, firewall interaction).

**D4. Waking competing workers: a semaphore replaces the shared-port socket.**
- Decision: each app gets a semaphore created by the router and inherited by the prototype and its workers. Where the router writes a READ_QUEUE doorbell into the shared port today, it releases the semaphore. Workers wait on the semaphore together with their own pipe reads.
- Why: on Unix every worker polls one shared socket (1.14), a competing-consumer pattern that a pipe instance does not offer. Semaphores can be duplicated and inherited (2.2) and wake one waiter per release. The `notified` flag protocol in the app queue (src/nxt_app_queue.h:84@872bf041) stays as it is.
- Invariant: a semaphore carries no data, so every message for the shared port must be enqueueable and its READ_SOCKET fallback must be unreachable. That holds today: requests go straight into the app queue with nxt_app_queue_send, and a full queue is answered with a 500 (src/nxt_router.c:8291, :8308-8313@872bf041); a QUIT is queued and woken with a plain READ_QUEUE (src/nxt_port_socket.c:347-357@872bf041). The Windows port layer should assert it. Workers drain their per-context queue before the app queue (1.14).
- Risk: the lost-wake-up analysis must be redone (section 5).

**D5. Descriptor passing: names for sections, trusted-side duplication for files and sockets.**
- Shared memory (rows 1, 2, 3, 5, 7, 8, 13 to 16 of 1.5): create named sections inside the private namespace `FreeUnit-<instance>`. Main creates the namespace with a boundary descriptor holding the unitd user SID. Names end in at least 128 random bits. ERROR_ALREADY_EXISTS on create is a hard failure, never "use the existing object" (2.6). The message carries the name; the receiver opens it with OpenFileMapping. The creator keeps its handle until the receiver acknowledges. That already holds for data segments and queues. It does not hold for the four blob kinds (rows 13 to 16): they are sent with NXT_PORT_MSG_CLOSE_FD and the sender's descriptor is closed right after the write (src/nxt_cert.c:1642-1644, src/nxt_script.c:742-744, src/nxt_controller.c:727-729, :3106-3108@872bf041). A named section disappears with its last handle (2.6), so certificates, scripts, the config store and the config push all need an acknowledgement before the creator closes. libunit's segment names need the random part too; today they are pid and thread id (1.6). This needs no process handles and works between siblings: controller to router, worker to router.
- Port write ends (rows 1, 2, 4): no transfer at all. Peers connect by pipe name (D3).
- Files (rows 9, 11, 12, 13, 14): duplicated into the receiver by the trusted side. Main pushes into its children; it holds PROCESS_ALL_ACCESS handles from CreateProcess (2.2). The router pushes the request-body file into a worker. It opens the worker with OpenProcess(PROCESS_DUP_HANDLE) once, when the prototype's NEW_PORT announces the worker (row 1), and checks the process creation time against pid reuse. NEW_PORT carries `{id, pid, max_size, max_share, type}` today (src/nxt_port.c:534-538@872bf041), so the creation time is a new field. Log reopen (row 12) becomes "reopen by path" in each process: all processes share one user in v1, so the reason for main opening the file is gone.
- Listening sockets (row 10): main keeps binding and sends the socket with WSADuplicateSocket, because it knows the router's pid (2.2). Apache httpd passes its listeners this way; PostgreSQL uses the same call for each accepted client connection (section 3). Whether the router could bind by itself depends on an open question about ports below 1024 (section 5).
- Ownership: today a handler that keeps a descriptor sets its slot to -1, and the port layer closes whatever is left set after dispatch (src/nxt_port.h:340-346, helper `nxt_port_recv_msg_close_fds` :352@872bf041). The same rule carries over to handle attachments. If delivery fails after a push, the pusher closes the remote copy with DUPLICATE_CLOSE_SOURCE (2.2).

**D6. Shared memory mapping: unchanged layout, Windows size check.**
- Decision: map every section whole at offset 0 with MapViewOfFile. Replace the `fstat` size check (1.6) with "map exactly PORT_MMAP_SIZE, or the queue size": a view beyond the section's maximum size fails (2.6). Keep the header checks. No fixed base addresses (1.6).
- Granularity: the offset is always 0, so the 64 KiB allocation-granularity rule (2.6) costs only virtual address space. The 4 KiB header is a layout constant, not a page assumption. The page size reported by GetSystemInfo is still to be confirmed per architecture (section 5).

**D7. Atomics: build with clang-cl first; add an MSVC branch only if MSVC must be supported.**
- Decision: nxt_atomic.h uses GCC `__sync` builtins and GNU inline assembly (1.7). clang-cl targets the MSVC ABI and C runtime that official PHP builds use (2.9) and should accept both; this is unverified (section 5). A plain MSVC build needs an Interlocked branch in nxt_atomic.h with size dispatch, because the same macros act on `uint32_t` ring counters and `uintptr_t` words (1.7), plus `YieldProcessor()` for the pause. C11 `stdatomic.h` in MSVC still needs an experimental switch on Visual Studio 2022 (2.9), so it is not the base.

**D8. Child supervision: per-process waits posted to the engine; job J for cleanup.**
- Decision: for each child, RegisterWaitForSingleObject(WT_EXECUTEONLYONCE | WT_EXECUTEINWAITTHREAD) on the process handle. The callback posts the pid and handle to the main engine; on IOCP that is PostQueuedCompletionStatus, as PostgreSQL does (section 3). The handler then runs today's SIGCHLD logic (1.9) with GetExitCodeProcess. Job J has KILL_ON_JOB_CLOSE and no breakaway flag, and only main holds its handle (non-inheritable). Main assigns its own children; workers join J automatically because their prototype is in J (2.3). If main dies, every FreeUnit process dies with it. libuv differs: its job allows silent breakaway, "so only the processes that we explicitly add are affected, and *their* subprocesses are not" (section 3). Per-app nested jobs (Windows 8 and later, 2.3) are optional, for app-level kill and accounting.
- Do not rely on job completion-port messages for reaping: their delivery "is not guaranteed" (2.3).
- The state store child becomes a thread in main.

**D9. Signals: console events, service controls and a control pipe, posted to the main engine.**

| Unix signal today | Windows v1 source | FreeUnit action |
|---|---|---|
| SIGINT, SIGTERM (fast quit) | Ctrl+C, Ctrl+Break, console close (5000 ms limit); later SERVICE_CONTROL_STOP (the handler returns within 30 s) and SERVICE_CONTROL_SHUTDOWN (about 20 s in total) | `rt->quit_mode = NORMAL`, nxt_runtime_quit |
| SIGQUIT (graceful quit) | `unitd --signal quit`, sent as one byte over `\\.\pipe\fu-<instance id>-ctl` (a findable name, protected as described below); later service control code 128 | GRACEFUL quit |
| SIGUSR1 (log reopen) | `unitd --signal reopen` over the same pipe; later code 129 | reopen by path in every process (D5) |
| SIGCHLD | D8 waits | reaper logic |
| SIGHUP, SIGUSR2, SIGPIPE | no equivalent | dropped; Winsock reports errors instead of SIGPIPE |

- The console handler runs on a system thread and only posts to the main engine. That is the shape of today's sigwait thread (1.9).
- Children run without the user's console: CREATE_NO_WINDOW or DETACHED_PROCESS, with stdout and stderr redirected to inherited handles. nginx starts its workers with CREATE_NO_WINDOW (section 3). CREATE_NEW_PROCESS_GROUP alone is not enough: it disables only Ctrl+C (2.1), while Ctrl+Break and console close still reach every process attached to the console (2.7). Main then sends QUIT as today. This must be tested (section 5).
- The control pipe follows PostgreSQL's per-process signal pipe (section 3), with a DACL for the unitd user only. nginx and Apache use named events instead; one pipe carries every command and reports the caller's pid. The operator must be able to find its name, so it cannot hide behind a secret: `unitd --signal` must open it with SECURITY_SQOS_PRESENT | SECURITY_IDENTIFICATION and check that the server process is unitd running as the expected user. Otherwise another local user who creates the name first could impersonate the operator (2.4).

**D10. Daemon mode, pid file and service.**
- v1: always foreground, no fork-to-background; write the pid file for tooling; log to stderr or the configured file.
- Later: `unitd --service install|remove|run`. Call StartServiceCtrlDispatcherW within 30 seconds (2.7). Map STOP and SHUTDOWN to fast or graceful quit, user codes to reopen and graceful quit. Write startup failures to the Event Log, as Apache does, and everything else to files. Use the SCM's recovery actions instead of a supervisor for main itself. The service account is a later decision; virtual and managed accounts were not researched here.

**D11. Identity and isolation.**
- v1: `user` and `group` are accepted only when they name the current account; anything else is a configuration error. No `isolation`.
- Later: workers on a restricted token. CreateRestrictedToken plus CreateProcessAsUser needs no SE_ASSIGNPRIMARYTOKEN_NAME for a restricted copy of the caller's token (2.8); PostgreSQL does exactly this (section 3). After that come per-app job limits as a partial cgroup replacement and AppContainer (Windows 8 and later, 2.1; mind the loopback rule, 2.8). The D5 rule that no worker duplicates handles is what keeps this upgrade possible.
- Elevated start: PostgreSQL's server refuses to run with administrative permissions, pg_ctl always starts it under a restricted token, and the other front-end tools re-execute themselves under a restricted token unconditionally (section 3). Recommend a refusal, or at least a warning, for an elevated unitd.

**D12. Language modules and the unitd import surface.**
- Loading: `php.unit.dll` and friends with LoadLibraryExW and GetProcAddress("nxt_app_module") in discovery and the prototype, in place of dlopen and dlsym (1.13).
- Link model: split the export macro. `NXT_EXPORT` (src/nxt_clang.h:123@872bf041) marks unitd's API and also each module's own `nxt_app_module` (src/nxt_php_sapi.c:369, src/python/nxt_python.c:49@872bf041), so it must stay `__declspec(dllexport)` everywhere. A second macro, for example `NXT_API`, marks unitd's surface: `__declspec(dllexport)` in unitd.exe and `__declspec(dllimport)` in modules. The linker then emits `unitd.lib`, the import library of the executable, which modules link against. PostgreSQL's MSVC build reaches the same result with a generated `.def` file that exports every function and needs PGDLLIMPORT only on variables; Node.js addons link `node.lib` (section 3 and 2.9). Also link libunit's helper objects (lvlhsh, murmur hash, socket message, websocket) into every module, as `libunit.a` already contains them (1.13). That cuts the PHP module's imports from 18 to 8: the four config getters, `nxt_malloc`, `nxt_zalloc`, `nxt_server` and `nxt_unit_default_init`.
- Longer term: replace even those 8 with a function table handed to the module at load, as SQLite does (2.9). Then a module no longer depends on the executable's file name.
- PHP: link against the import library of `php8.dll` from the official Development package (VS17, NTS). The embed library is not needed (2.9); this still has to be proven by a build (section 5). Opcache: recommend `opcache.file_cache` with `opcache.file_cache_fallback=1`, so a worker that cannot reattach still runs (2.9).
- Python WSGI: embed `python3XY.dll`. Python ASGI and Node.js need non-readiness integrations (2.5), so they are deferred. Java, Perl and Ruby are deferred. WASM with wasmtime is feasible on x64 (Tier 1, 2.9).

**D13. Control socket.**
- Decision: an AF_UNIX stream socket file (HTTP needs only a stream) in a directory whose ACL admits only the unitd user. On accept, read the peer pid with SIO_AF_UNIX_GETPEERPID, open the peer's token and compare its user SID with unitd's or with an allowed group. This mirrors the SO_PEERCRED check of 1.8. Fallback: TCP on loopback, documented as unauthenticated, as it is on Unix today (1.8).
- Risk: the peer-pid ioctl has no learn.microsoft.com page, no stated minimum build, and no documented behaviour after the socket is passed to another process (2.5, section 5).

### Gaps

- Which event-engine model the Windows build will use (IOCP or a readiness emulation) is decided elsewhere. D3 depends on it.
- No Microsoft source was found on whether concurrent writers sharing one message-mode pipe instance get atomic messages. D3 avoids the question by using one instance per writer.
- No authoritative source was found for the Windows Defender Firewall prompt rules (when a listener triggers it, and whether loopback-only binds avoid it).
- Virtual service accounts and their minimum version were not checked.
- Visual Studio 2022 (VS17) was confirmed for the PHP 8.5 Windows builds only; the compiler of the PHP 8.4 builds was not checked.

## 3. Precedent evidence table

### Takeaway

PostgreSQL, Apache and nginx re-execute their own binary, hand the child its startup state through a command line, an environment variable or a pipe, and use named events or pipes in place of signals. Handles cross between processes through a process that knows both pids: the parent in PostgreSQL and Apache, any pipe peer in libuv's IPC pipes. Fixed-address shared memory is the recurring pain on Windows; FreeUnit avoids it by design. libuv closed its handle-inheritance race with PROC_THREAD_ATTRIBUTE_HANDLE_LIST only in 2026.

### Cited Findings

| project, tag | files | mechanism | outcome | URL |
|---|---|---|---|---|
| PostgreSQL REL_18_6, spawn | src/backend/postmaster/launch_backend.c:417-557 | CreateProcess with handle inheritance and CREATE_SUSPENDED (:488); BackendParameters written to an inheritable unnamed file mapping whose handle value goes on the command line after `--forkchild=` (:440, :465-470); ResumeThread (:557). Struct holds the shared memory id and address, PostmasterHandle, signal pipe, syslog pipe and sockets (:76-149) | native Windows server since 8.0, released 2005-01-19: "the first PostgreSQL release to run natively on Microsoft Windows as a server. It can run as a Windows service." (https://www.postgresql.org/docs/release/8.0.0/) | https://github.com/postgres/postgres/blob/REL_18_6/src/backend/postmaster/launch_backend.c#L417 |
| PostgreSQL REL_18_6, handle passing | launch_backend.c:815-833, :845-851 | DuplicateHandle into the suspended child with DUPLICATE_CLOSE_SOURCE; the accepted client socket via WSADuplicateSocket(child pid) (:737-741) | the parent always brokers; no listener is passed | https://github.com/postgres/postgres/blob/REL_18_6/src/backend/postmaster/launch_backend.c#L815 |
| PostgreSQL REL_18_6, shared memory | src/backend/port/win32_shmem.c:65-81, :286-327, :424-445, :573-607 | paging-file mapping named `Global\PostgreSQL:<data dir>`; on ERROR_ALREADY_EXISTS sleep 1 s and retry; children reattach at the same address with MapViewOfFileEx after the parent reserved that range in the suspended child with VirtualAllocEx | spawn retried up to 100 times, "This might be caused by ASLR or antivirus software." (launch_backend.c:535-555); the reservation fix landed in 2009 for the "long-time issue with 'could not reattach to shared memory' errors" (https://www.postgresql.org/message-id/20090811115120.3931975331E%40cvs.postgresql.org) | https://github.com/postgres/postgres/blob/REL_18_6/src/backend/port/win32_shmem.c#L573 |
| PostgreSQL REL_18_6, signals and supervision | src/backend/port/win32/signal.c:79-109, :227-235, :274-327; src/port/kill.c:61-63; postmaster.c:1023, :4516-4523, :4532-4545; pmsignal.c:391 | one named pipe `\\.\pipe\pgsignal_<pid>` per process, read by a signal thread that sets an event; senders use CallNamedPipe; child exit via RegisterWaitForSingleObject(WT_EXECUTEONLYONCE) posting to an IOCP plus a queued SIGCHLD; postmaster death via WaitForSingleObject on PostmasterHandle | signal semantics preserved | https://github.com/postgres/postgres/blob/REL_18_6/src/backend/port/win32/signal.c#L227 |
| PostgreSQL REL_18_6, privilege, service, title, exports | src/common/restricted_token.c:45-149; src/bin/pg_ctl/pg_ctl.c:1530-1723, :1861-1886; src/backend/utils/misc/ps_status.c:45-46, :505-520; src/backend/meson.build:44-88 | the server refuses administrative permissions (src/backend/main/main.c:470-473); front-end tools re-execute themselves through CreateRestrictedToken (Administrators and Power Users removed, DISABLE_MAX_PRIVILEGE) and CreateProcessAsUser, guarded by PG_RESTRICT_EXEC, and pg_ctl always starts the server that way; `pg_ctl register` with CreateService and StartServiceCtrlDispatcher, STOP and SHUTDOWN handled together; job object with UI limits; process title published "as the name of a Windows event" `pgident(<pid>): <title>`; MSVC builds export every postgres.exe function through a generated `.def` file so "extension libraries can use them"; variables need PGDLLIMPORT | mature | https://github.com/postgres/postgres/blob/REL_18_6/src/backend/meson.build#L44 |
| Apache httpd 2.4.69, mpm_winnt | server/mpm/winnt/mpm_winnt.c:160-199, :242, :270-335, :349-430, :442-519, :635-677 | one parent and one child with ThreadsPerChild threads; events `apPID_shutdown` and `apPID_restart` opened with OpenEvent by `-k` or the service; the parent writes the generation, then DuplicateHandle'd ready and exit events and the scoreboard section into the child's stdin pipe; listeners via WSADuplicateSocket(child pid), child calls WSASocket(FROM_PROTOCOL_INFO); child finds `AP_PARENT_PID` in its environment | the code waits for the child's ready event before duplicating sockets: "if WSADuplicateSocket runs before the child process initializes the listeners will be inherited anyway" (:669-674) | https://github.com/apache/httpd/blob/2.4.69/server/mpm/winnt/mpm_winnt.c#L349 |
| Apache httpd 2.4.69, service and modules | server/mpm/winnt/service.c:347-348, :395, :531, :876; nt_eventlog.c:74-94; CMakeLists.txt:905, :927, :939 | RegisterServiceCtrlHandlerExW, StartServiceCtrlDispatcherW, CreateServiceW; stderr to the Event Log source "Apache Service"; the core is `libhttpd.dll`, `httpd.exe` is built from server/main.c, modules link libhttpd | mature | https://github.com/apache/httpd/blob/2.4.69/CMakeLists.txt#L927 |
| nginx release-1.31.6 | src/os/win32/ngx_process.c:79-81, :207-222; ngx_process_cycle.c:73-88, :320-323, :1013-1015; ngx_win32_init.c:258-272; ngx_shmem.c:62-91, :124-140; src/core/ngx_cycle.c:979-985 | CreateProcess with `bInheritHandles = 0` and CREATE_NO_WINDOW; a worker knows its role from the environment variable `ngx_unique`; events `ngx_master_<unique>`, `ngx_<name>_term_<pid>`, `..._quit_...`, `..._reopen_...`, `Global\ngx_stop_<unique>`; `nginx -s` opens `Global\ngx_<sig>_<pid>`; named mappings mapped at a wanted base, with a remap when the address differs | docs: "Although several workers can be started, only one of them actually does any work", only select() and poll(), UDP and QUIC unsupported, "considered to be a beta version", running as a service is a "possible future enhancement" (https://nginx.org/en/docs/windows.html) | https://github.com/nginx/nginx/blob/release-1.31.6/src/os/win32/ngx_process.c#L207 |
| libuv v1.53.0, spawn | src/win/process.c:70-120, :1041-1097, :1132, :1152, :1194-1196 | STARTUPINFOEXW plus PROC_THREAD_ATTRIBUTE_HANDLE_LIST "closing the race condition where concurrent uv_spawn calls could cause handles intended for one child to leak into another"; stdio in lpReserved2; global job with KILL_ON_JOB_CLOSE, BREAKAWAY_OK, SILENT_BREAKAWAY_OK and DIE_ON_UNHANDLED_EXCEPTION that also contains the parent, "so only the processes that we explicitly add are affected, and *their* subprocesses are not" (:77-79); exit via RegisterWaitForSingleObject | the handle list arrived in PR #5100, merged 2026-05-13 (https://github.com/libuv/libuv/pull/5100) | https://github.com/libuv/libuv/blob/v1.53.0/src/win/process.c#L1049 |
| libuv v1.53.0, pipes | src/win/pipe.c:51, :64-83, :135, :247, :1930-1981 | byte-mode named pipes `\\?\pipe\uv\<id>-<pid>`; IPC framing with a 16-byte header and flags for socket transfer; the remote pid comes from GetNamedPipeClientProcessId, or from GetNamedPipeServerProcessId when the client pid is our own (:1930-1940), so sockets move between any two pipe peers | AF_UNIX not used, issue #2537 open since 2019 | https://github.com/libuv/libuv/blob/v1.53.0/src/win/pipe.c#L64 |
| PHP php-8.5.11, opcache | ext/opcache/shared_alloc_win32.c:35, :72-91, :117-185, :216-219, :273-303 | named mapping `ZendOPcache.SharedMemoryArea@<user id>@<SAPI name>@<system id>`, PAGE_EXECUTE_READWRITE with SEC_COMMIT, fixed base candidates or `opcache.mmap_base`; reattach refused when `execute_ex` moved or the range is taken, then falls back to the file cache if enabled, else fatal "Unable to reattach to base address" | file cache fallback is the documented escape (2.9) | https://github.com/php/php-src/blob/php-8.5.11/ext/opcache/shared_alloc_win32.c#L117 |
| Node.js, node-gyp | src/win_delay_load_hook.cc:1-8 | addons import from the host executable; a delay-load hook returns the process image instead of searching for `node.exe` | lets addons survive a renamed host | https://github.com/nodejs/node-gyp/blob/main/src/win_delay_load_hook.cc |
| SQLite | loadable extensions | `sqlite3_api_routines` table passed to the extension's init function | no link-time dependency on the host | https://www.sqlite.org/loadext.html |

### Inferences

- Bootstrap: PostgreSQL (mapping handle on the command line), Apache (stdin pipe) and nginx (environment variable) all re-execute and pass startup state explicitly. D1 takes PostgreSQL's form, because FreeUnit's prototype and app configs are larger than an environment block wants to carry.
- Signals: named events (nginx, Apache) and per-process pipes (PostgreSQL) are both proven. A pipe also reports who called and can carry a parameter, so D9 uses one.
- Reaping: PostgreSQL and libuv converge on RegisterWaitForSingleObject plus a post into the event loop. D8 copies that.
- Handle transfer: PostgreSQL and Apache transfer between parent and child, with the parent doing the work. libuv's IPC pipes move sockets between any two pipe peers and identify the peer by the pipe-reported pid, the closest precedent for D3's per-connection identity. FreeUnit's sibling transfers (controller to router) and its worker-to-router transfers of sections have no exact precedent, which is why D5 uses names for shared memory and keeps duplication with the trusted side.
- Fixed-address shared memory caused the longest-running Windows bugs in PostgreSQL, nginx and opcache. FreeUnit's offset-based layout (1.6) must stay free of absolute pointers; a static check or a test should guard it.

### Gaps

- No credible numbers were found for Windows process start versus Unix fork in any of these projects. A 2004 pgsql-hackers thread on pre-fork speed-ups gives no Windows figures (https://hackorum.dev/topics/13052).
- The internals of Node.js cluster on Windows were not checked.

## 4. Unsupported in v1, and the abstractions to introduce

### Takeaway

v1 should run PHP, Python WSGI and WASM apps for one local user, in the foreground, without per-app users, isolation, process titles or Unix signals. The port needs about a dozen narrow abstractions. Most are confined to the process, port, shared-memory and signal files; libunit changes in its channel, wait and shared-memory code only.

### Cited Findings

- Per-app identity changes need either credentials and privileges or a restricted token (2.8). Windows has no setuid model. AppContainer is Windows 8 and later and isolates loopback (2.1, 2.8).
- Node.js waits with uv_poll and Python ASGI with asyncio `add_reader` (1.14); on Windows both accept only sockets, and the default asyncio loop has no `add_reader` at all (2.5).
- Fork-based isolation code, all Linux-only: unshare and the PID namespace fork (src/nxt_process.c:574, :588@872bf041), pivot_root, chroot and mounts (src/nxt_isolation.c:1130-1530, src/nxt_fs_mount.c:93@872bf041), cgroups (src/nxt_process.c:756@872bf041), uid/gid maps (nxt_clone_credential_map src/nxt_clone.c:190@872bf041, called from src/nxt_main_process.c:919), capabilities (src/nxt_capability.c:663@872bf041).
- Process titles overwrite argv and environ (src/nxt_process_title.c:46, :190@872bf041); Windows has no equivalent, and PostgreSQL publishes a named event instead (section 3).

### Inferences

#### 4.1 Unsupported in v1

1. `user` and `group` per application, unless they name the current account (D11).
2. Everything under `isolation`: namespaces, uid/gid maps, `rootfs`, `automount`, `cgroup`, `new_privs` (D11).
3. Running as a background daemon. v1 is foreground only; service mode comes later (D10).
4. Unix signals as an interface. Replaced by Ctrl+C and `unitd --signal quit|reopen|stop` (D9).
5. Process titles; the role shows in the command line instead.
6. Python ASGI and Node.js apps (fd-readiness integrations, 1.14 and 2.5).
7. External apps (`type: external`, Go and Node.js). Without exec, the prototype must start the user's binary directly and pass pipe and section names in `NXT_UNIT_INIT` (D1). It also needs libunit on Windows and, for Go, the cgo toolchain.
8. Perl, Ruby and Java modules. Not blocked in principle; out of scope for a first version.
9. Isolation between workers, and between workers and the router (same user, same integrity; 2.A).
10. Abstract-namespace sockets anywhere; Unix-domain listeners only as AF_UNIX socket files, if at all.
11. ARM64 and 32-bit Windows. x64 only; wasmtime's aarch64 Windows target is Tier 3 (2.9).
12. Windows releases older than Windows 10 1809 or Windows Server 2019 (2.A).

#### 4.2 Abstractions to introduce, and the files each one touches

All file references are @872bf041.

| abstraction | replaces | files |
|---|---|---|
| A1 process spawn: fork path or re-exec path with role, handle list, bootstrap block, suspended start | fork, the child fixup, execve | src/nxt_process.c (nxt_process_start :243, nxt_process_create :687, nxt_process_unshare :541, nxt_process_child_fixup :326 becomes a child bootstrap), src/nxt_main_process.c (start handlers :554, state store fork :2557), src/nxt_application.c (nxt_proto_start_process_handler :742), src/nxt_external.c (:60-201), src/nxt_runtime.c (argv :1334, daemon :399), src/nxt_main.c (role dispatch) |
| A2 process watch: exit notification, exit status, kill | SIGCHLD, waitpid, kill | src/nxt_main_process.c (:1469-1612), src/nxt_application.c (:1138-1153, kill :1052), src/nxt_port.c (kill :966), src/nxt_process.c (kill :769), src/nxt_signal.c |
| A3 OS handle type and message attachments: `{kind, value or name}` instead of `fd[2]`, same ownership rules | nxt_fd_t in messages, SCM_RIGHTS | src/nxt_port.h (send and receive message structs, close helpers), src/nxt_socket_msg.h and .c, src/nxt_port_socket.c, the 16 send sites and handlers of 1.5 |
| A4 port channel: create endpoint, connect writer with a server identity check, send record, receive record, peer pid per connection, READ_SOCKET markers that name the writer | socketpair, shared write ends, anonymous markers | src/nxt_socketpair.c, src/nxt_port_socket.c (:126-188, the marker path :358-405 and :2020, the read and write handlers), src/nxt_port.c (nxt_port_send_port :515, new-port handling), src/nxt_router.c (port creation :2053, :3325, :5199), src/nxt_main_process.c (:1281), src/nxt_unit.c (nxt_unit_create_port :6773, send and receive paths, markers :7440-7500, nxt_unit_read_buf :5899) |
| A5 doorbell for competing consumers, with an assert that only enqueueable messages reach it | the app shared port socket | src/nxt_router.c (shared port creation :3325-3342, START_PROCESS fds :721-722), src/nxt_unit.c (shared port receive, :5971-5991), src/nxt_external.c (`NXT_UNIT_INIT` fields) |
| A6 shared memory: create named or unnamed section, map with size check, open by name, transfer | memfd, shm_open, fstat, mmap | src/nxt_port_memory.c (nxt_shm_open :493, incoming checks :300-336, :442), src/nxt_unit.c (nxt_unit_shm_open :4958, :4895, :5159, :644, :6639), src/nxt_router.c (:3968, :3996, :4024), src/nxt_port.c (:808-830), src/nxt_controller.c (:702, :3083), src/nxt_cert.c (:1608), src/nxt_script.c (:708) |
| A7 peer identity: `nxt_recv_msg_cmsg_pid()` backed by the connection's client pid | SCM_CREDENTIALS, SCM_CREDS | src/nxt_port.h (:293-332), src/nxt_socket_msg.h, the check sites of 1.8 (src/nxt_main_process.c, src/nxt_application.c :719-735, src/nxt_port.c :751, src/nxt_router.c :1718, src/nxt_cert.c, src/nxt_script.c), control socket check src/nxt_controller.c:817-962 |
| A8 atomics for MSVC, only if MSVC is a target | GCC builtins, GNU inline assembly | src/nxt_atomic.h (:17-66) |
| A9 control events: console handler, service controls, control pipe, posted to the main engine | signal tables and the sigwait thread | src/nxt_signal.c, src/nxt_signal_handlers.c (:20-26), src/nxt_main_process.c (:152-157, :1326-1440), a new service file |
| A10 dynamic loading and exports: LoadLibraryExW and GetProcAddress; a new NXT_API macro (dllexport in unitd, dllimport in modules) beside NXT_EXPORT (always dllexport); unitd.lib | dlopen, dlsym, `-Wl,-E` | src/nxt_application.c (:413, :420, :1519, :1528), src/nxt_clang.h (:123), auto/os/conf, auto/make (:120-170), auto/modules/* |
| A11 process title | argv overwrite | src/nxt_process_title.c (:46, :190): no-op, or PostgreSQL's named event |
| A12 config validation rejects unsupported keys | n/a | src/nxt_conf_validation.c (:1342-1357 and the isolation validators) |

### Gaps

- The list of Unix-only code is complete for the process and IPC layer only. The event engines, the HTTP and TLS stack, file serving and the build system belong to other studies.

## 5. Open questions and the experiments that settle them

### Takeaway

Five questions decide the design. Can a worker be spawned fast enough without fork? Is opcache sharing reliable with a resident prototype? Do message-mode pipes, used one instance per writer, meet FreeUnit's throughput? Is the AF_UNIX peer-pid ioctl present on the target builds? Does clang-cl build the atomics unchanged? Each has a small experiment. Q16 tests the defence against pipe squatting.

### Cited Findings

- No published measurement of Windows backend start cost was found (section 3 gaps).
- SIO_AF_UNIX_GETPEERPID is in `afunix.h` but has no learn.microsoft.com page and no stated minimum build (2.5).
- MSVC still needs `/experimental:c11atomics` for C11 atomics on Visual Studio 2022 (2.9).
- Job completion-port messages may be lost (2.3), and pids can be recycled (2.3).

### Inferences

| # | question | experiment | decides |
|---|---|---|---|
| Q1 | How long does a worker take to start: CreateProcess, LoadLibrary of the module, `setup`, `start`? | On Windows 11 and Server 2022, time 200 spawns of `unitd.exe --role app` for PHP NTS with opcache, Python WSGI and a 10 MB WASM module with QueryPerformanceCounter. Report p50 and p95 against fork from the prototype on Linux on the same machine. | D2; whether WASM needs a serialized-module cache in v1 |
| Q2 | Does opcache reattach reliably when the prototype keeps php8.dll loaded? | Run 32 workers with `request_limit=50` for one hour. Count "Opcode handlers are unusable due to ASLR", "Base address marks unusable memory region" and "Unable to reattach" log lines with and without a resident prototype, with `file_cache_fallback` on and off. | D2, D12 |
| Q3 | Are writes from several processes atomic on one message-mode pipe instance, and does the writer-tagged marker keep order? | (a) Four processes share one client handle and each writes 10,000 records of 1 to 16 KiB with sequence numbers; the server checks boundaries and interleaving. (b) Eight writers on their own instances mix queue messages and pipe records with writer-tagged markers; the reader checks per-writer order. | whether one instance per writer is mandatory; the marker design (D3) |
| Q4 | Pipe or AF_UNIX: latency and throughput for FreeUnit's records? | Ping-pong of 31-byte and 16 KiB records over a message-mode pipe with IOCP, AF_UNIX stream with IOCP, and loopback TCP, one to eight pairs. | D3 |
| Q5 | Which builds support SIO_AF_UNIX_GETPEERPID, and do AF_UNIX sockets work with IOCP and WSAPoll? | Call the ioctl on Windows 10 1809 and 22H2, Windows 11 24H2, Server 2019, 2022 and 2025; record success or the error code, and which pid it reports after WSADuplicateSocket to another process. Run overlapped WSARecv and WSAPoll on an AF_UNIX pair. | D13, the D3 alternative |
| Q6 | What error does a view larger than the section return? | Create a section of 10 MiB and map 10 MiB + 4 KiB; record GetLastError. Read GetSystemInfo dwPageSize and dwAllocationGranularity on x64 and ARM64. | D6 |
| Q7 | Is the semaphore doorbell free of lost wake-ups with the `notified` protocol? | Model the router release and the worker wait-then-dequeue order; stress with 64 workers and bursts of 1 to 10,000 requests; assert no request waits longer than one drain cycle, and that no non-enqueueable message reaches the shared port. | D4 |
| Q8 | Does clang-cl compile nxt_atomic.h unchanged, and what does Visual Studio 2026 offer? | Compile nxt_nncq.h, nxt_app_queue.h and nxt_port_memory.c with clang-cl; inspect `pause` and `lock cmpxchg` in the object code; try `cl /std:c17` with and without `/experimental:c11atomics` on the newest MSVC. | D7 |
| Q9 | Can a standard user bind ports below 1024, and when does the firewall prompt appear? | On a clean Windows 11 VM as a standard user, bind 0.0.0.0:80, 127.0.0.1:80 and [::1]:80 from an unsigned test program; record bind results and any prompt. | D5 for listeners; first-run behaviour |
| Q10 | Does the job model hold when unitd starts inside someone else's job (IDE, Windows Terminal, CI runner)? | Start unitd under each; check AssignProcessToJobObject results, then kill main with TerminateProcess and confirm all children exit. | D8 |
| Q11 | Does the PHP module build against the official devel pack without the embed library? | Build src/nxt_php_sapi.c with VS17 against the PHP 8.5 NTS Development package import library; load it in a test unitd.exe; find the minimum DLL search-path setup for php8.dll (AddDllDirectory or LOAD_LIBRARY_SEARCH_* flags). | D12 |
| Q12 | Is OpenProcess by pid safe for the router's duplication into workers? | Kill a worker and restart one in a tight loop; compare the creation time reported by the prototype with GetProcessTimes on the opened handle; count mismatches. | D5 |
| Q13 | Can log files be rotated by renaming while FreeUnit holds them open? | Open the logs with FILE_SHARE_READ, FILE_SHARE_WRITE and FILE_SHARE_DELETE, then rename them with a typical rotation tool. | D9 log reopen |
| Q14 | Do Ctrl+C, Ctrl+Break and closing the console reach only main when children run with CREATE_NO_WINDOW or DETACHED_PROCESS? | Run unitd in conhost and Windows Terminal; send each event; check that workers exit only after main's QUIT and that their stderr still reaches the log. | D9 |
| Q15 | Can a future restricted or low-integrity worker still open the private namespace and the pipes? | Spawn a worker with a restricted token and with Low integrity; call OpenPrivateNamespace, OpenFileMapping and CreateFile on a port pipe; record access results. | D5 and D11 forward compatibility |
| Q16 | Can another local user squat or impersonate FreeUnit pipes? | As a second user, pre-create the control pipe name and a guessed port pipe name, then start unitd and run `unitd --signal`; check that the client refuses a server that is not unitd under the expected user, and that SECURITY_IDENTIFICATION blocks impersonation. | D3, D9 |

### Gaps

- None of these experiments has been run; they need Windows hosts. Each cell in the "decides" column names the decision that should be revisited when the result arrives.
