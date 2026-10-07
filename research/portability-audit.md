# FreeUnit native Windows port: Unix dependency audit of the core and libunit

Revision: freeunitorg/freeunit `872bf04170756e919211799ec5574000d628d521` (origin/master on 2026-10-07). Every code claim is cited as `path:line@872bf041` with a repository-relative path. A citation resolves at `https://github.com/freeunitorg/freeunit/blob/872bf04170756e919211799ec5574000d628d521/<path>#L<line>`; ranges are `path:A-B@872bf041`. Where a run of citations in one bullet or table cell refers to the same file and revision, later ones are abbreviated to `:line`.

Method. Static reading only, with `git show 872bf041:<path>` and `git grep ... 872bf041`. Nothing was built or run. Call-site counts come from a short script that strips comments and string literals and then matches `name(` (definitions and `#define` lines excluded). Areas: "core" is `src/*.c|h` minus the rest; "libunit" is `src/nxt_unit*`; "modules" is `src/nxt_php_sapi.c`, `src/nxt_java.c`, `src/python`, `src/perl`, `src/ruby`, `src/java`, `src/nodejs`, `src/wasm`, `src/wasm-wasi-component`; "go" is `go/`; "tests" is `src/test`. Windows facts are cited to their sources as links.

Grades used in every table:
- **none**: the call exists on Windows (CRT or Winsock) with the same contract, or an `#ifdef` is enough.
- **wrapper**: a `nxt_win32_*` function can keep the same contract for all callers.
- **redesign**: the contract itself must change, and its callers change with it.

## 1. Process model: fork, exec, signals, credentials, isolation

### Takeaway
FreeUnit creates every process with `fork()` and never re-executes itself. A child continues in the parent's code with the parent's memory, its process table and its port descriptors. Workers are forked from a per-application prototype that has already loaded the language module, run the module's setup hook (for PHP, the whole `php_module_startup()`), applied isolation and changed directory. Windows has no fork, so the prototype design, and everything a worker inherits, must be rebuilt around `CreateProcess`, explicit handle passing and per-worker runtime start-up. This is a redesign.

### Cited Findings

Table 1.1. Process-model Unix APIs.

| Unix API or mechanism | Where (call sites at 872bf041) | Windows counterpart | Grade | Evidence |
|---|---|---|---|---|
| `fork()` for every child | core 4: `src/nxt_process.c:704@872bf041` (all process creation), `:588` (second fork for a PID namespace), `:1385` (daemon); `src/nxt_main_process.c:2561@872bf041` (state-store child). Tests: 19 more. | `CreateProcess` of `unitd.exe` with a role argument. Win32 has no fork; Cygwin emulates it by starting the child with `CreateProcess` and copying the parent's memory ([Cygwin highlights](https://cygwin.org/cygwin-ug-net/highlights.html)). | redesign | The child returns into the parent's work queue: `src/nxt_process.c:287-293@872bf041`, child path `:716-742`. |
| `execve()` for external apps (Go, Node) | core 1: `src/nxt_external.c:201@872bf041` | `CreateProcess` with an explicit inheritable handle list | wrapper | Port fd numbers go in-band in `NXT_UNIT_INIT` (`src/nxt_external.c:121-134@872bf041`, `setenv` `:144`); `FD_CLOEXEC` is cleared on six fds first (`:89-117`). |
| `posix_spawn()` | core 1: `src/nxt_process.c:1361@872bf041` | not needed | none | `nxt_process_execute()` has no caller; only its declaration `src/nxt_process.h:220@872bf041` references it. |
| `waitpid(-1, WNOHANG)` on SIGCHLD | core 2: `src/nxt_main_process.c:1487@872bf041`, `src/nxt_application.c:1153@872bf041` | wait on process handles (`RegisterWaitForSingleObject`, or a job-object completion port) | wrapper | Reaping drives port cleanup, QUIT to the dead process's children and restarts: `src/nxt_main_process.c:1541-1616@872bf041`. |
| `kill()` | core 4: `src/nxt_process.c:769@872bf041` (SIGTERM), `src/nxt_application.c:1052@872bf041`, `src/nxt_main_process.c:2392@872bf041`, `src/nxt_port.c:966@872bf041` (SIGKILL) | `TerminateProcess`, or closing a job with `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE` ([job limits](https://learn.microsoft.com/en-us/windows/win32/api/winnt/ns-winnt-jobobject_basic_limit_information)) | wrapper | All four are hard kills of a known child. |
| daemonize: `setsid()`, `umask()`, `/dev/null` + `dup2()` | core: `src/nxt_process.c:1413@872bf041`, `:1422`, `:1426-1437`; called from `src/nxt_runtime.c:401@872bf041`; pid file `:423`, `:1615` | Windows service (SCM) or a detached console process | redesign (small) | `nxt_process_daemon()` `src/nxt_process.c:1371-1457@872bf041` forks, so it cannot be wrapped. |
| `prctl()` | core 3: `src/nxt_process.c:673@872bf041` (PR_SET_CHILD_SUBREAPER), `:1288` (PR_SET_NO_NEW_PRIVS), `src/nxt_capability.c:663@872bf041` (PR_CAP_AMBIENT) | job object for tree ownership; no counterpart for no-new-privs | drop | All behind probes or `#ifdef`: `src/nxt_process.c:672@872bf041`, `:1286`, `src/nxt_capability.c:641@872bf041`. |
| `unshare()`, PID-namespace double fork, uid/gid maps | `src/nxt_process.c:574@872bf041`, `:584-608`; `src/nxt_clone.c` (434 lines of uid/gid map writers, `src/nxt_clone.c:24-297@872bf041`) | none; AppContainer and restricted tokens isolate differently ([AppContainer isolation](https://learn.microsoft.com/en-us/windows/win32/secauthz/appcontainer-isolation)) | drop | Linux-only behind `NXT_HAVE_LINUX_NS` (`src/nxt_process.c:693-725@872bf041`). |
| cgroups | `src/nxt_cgroup.c` (318 lines); child placed by pid `src/nxt_process.c:749-774@872bf041` | job objects (CPU and memory limits; nested jobs since Windows 8 and Server 2012, [nested jobs](https://learn.microsoft.com/en-us/windows/win32/procthread/nested-jobs)) | redesign | `NXT_HAVE_CGROUP`. |
| `mount`, `umount2`, `pivot_root`, `chroot` (rootfs isolation) | mount 6: `src/nxt_isolation.c:1130@872bf041`, `:1287`, `:1296`, `:1315`, `:1455`, `src/nxt_fs_mount.c:93@872bf041`; umount2 2; pivot_root through `syscall()` `src/nxt_isolation.c:1494@872bf041`; chroot `:1528` | none | drop | Applied in the prototype under `NXT_HAVE_ISOLATION_ROOTFS`: `src/nxt_application.c:613-627@872bf041`. |
| `getpwnam`, `getgrnam`, `getgrouplist`, `initgroups`, `setgroups`, `setuid`, `setgid` | core 9, all in `src/nxt_credential.c`: `src/nxt_credential.c:22@872bf041`, `:42`, `:106`, `:127`, `:224`, `:269`, `:290`, `:318`, `:337` | `LogonUser` or S4U logon plus `CreateProcessAsUser`; service virtual accounts; restricted tokens | redesign | App `user` and `group` options: `src/nxt_main_process.c:197-207@872bf041`; applied by `nxt_process_apply_creds()` `src/nxt_process.c:1257@872bf041`. |
| Linux capabilities | `src/nxt_capability.c` (784 lines, 727 of them inside OS conditionals) | token privileges (`AdjustTokenPrivileges`) | drop | Linux-only. |
| signals as the external control plane | handler tables: main `src/nxt_main_process.c:151-157@872bf041` (HUP, INT, QUIT, TERM, CHLD, USR1); prototype `src/nxt_application.c:112-117@872bf041`; all others `src/nxt_signal_handlers.c:19-26@872bf041`; SIGSYS and SIGPIPE ignored `src/nxt_signal.c:41-45@872bf041` | `SetConsoleCtrlHandler` (Ctrl+C, Ctrl+Break, close), SCM stop and preshutdown, a named event or the control API for log reopen. `CTRL_C_EVENT` cannot be targeted at one process group ([GenerateConsoleCtrlEvent](https://learn.microsoft.com/windows/console/generateconsolectrlevent)). | wrapper for dispatch, redesign for sources | Delivery: signalfd `src/nxt_epoll_engine.c:736@872bf041`, EVFILT_SIGNAL `src/nxt_kqueue_engine.c:661@872bf041`, or a `sigwait()` thread `src/nxt_signal.c:153-167@872bf041` (three routes described at `:11-21`). |
| process title | `src/nxt_process_title.c` (256 lines), set at `src/nxt_process.c:838@872bf041` | none (the image name is fixed) | drop | The pytest harness finds processes by these titles (section 6). |
| `getpid()` cached in `nxt_pid` | core 4: `src/nxt_lib.c:54@872bf041`, `src/nxt_process.c:337@872bf041`, `:1404`, `src/nxt_main_process.c:2569@872bf041`; libunit 1: `src/nxt_unit.c:577@872bf041` | `GetCurrentProcessId()` | none | |
| anonymous `MAP_SHARED` counter shared through fork | `src/nxt_port_rpc.c:96-104@872bf041`, used `:184`; created in main `src/nxt_runtime.c:132@872bf041` | a section handle passed at spawn, or a named section | wrapper | Gives globally unique RPC stream ids only because every process inherits the same mapping. |
| state-store child (fork, close every inherited fd, write, exit) | `src/nxt_main_process.c:2557-2590@872bf041`; fd closing `:2741-2846` (`closefrom` `:2745`, `close_range` `:2753`, `/proc/self/fd` `:2805`) | a worker thread in main | redesign (small) | The child only writes and syncs state files. |

The process tree and how each process starts:
- Main starts the discovery process first (`src/nxt_main_process.c:186@872bf041`), then the controller and the router once the module list is known (`:2174-2176`).
- Start-up records: discovery `src/nxt_application.c:142-148@872bf041`; prototype `:154-161` (prefork `nxt_isolation_main_prefork`, setup `nxt_proto_setup`, start `nxt_proto_start`); worker `:165-172` (setup `nxt_app_setup`, start NULL until filled); controller `src/nxt_controller.c:195-203@872bf041`; router `src/nxt_router.c:574-582@872bf041`.
- The router asks main for a prototype with START_PROCESS. Main builds a `nxt_proto_process` record (`src/nxt_main_process.c:611@872bf041`), stores the app's shared port and queue descriptors from that message (`:626-627`) and forks (`:714`).
- The prototype loads the module with `dlopen(RTLD_GLOBAL | RTLD_LAZY)` (`src/nxt_application.c:593@872bf041`, `:1519`), sets the app environment (`:599`), runs the module's `setup` hook (`:606-607`), prepares the rootfs (`:613-627`) and calls `chdir()` (`:629-640`).
- For each worker the router sends START_PROCESS to the prototype. The handler builds a record whose start function is the module's `start` (`src/nxt_application.c:817@872bf041`) and whose `data.app` points at the prototype's own copy of the app configuration (`:829`), then forks (`:843`).
- A worker never runs a setup hook. `nxt_app_setup()` calls `init->start` directly (`src/nxt_application.c:1661-1670@872bf041`).
- The only exec in the tree is the external-app worker (Go, Node), which execs the configured binary after the fork (`src/nxt_external.c:201@872bf041`).
- Main restarts processes marked `restart` (controller, router) when they die (`src/nxt_main_process.c:1599-1616@872bf041`).

What a worker inherits from its prototype and relies on:
- The process table. After fork the child closes the ports of process types it should not hold (`nxt_proc_keep_matrix`, `src/nxt_process.c:80-87@872bf041`, applied at `:372-394`). A worker keeps main's and the router's ports and its parent's port (`:375`; process type order `src/nxt_process_type.h:12-17@872bf041`; APP row `src/nxt_process.c:86@872bf041`).
- The port descriptors handed to libunit. `nxt_unit_default_init()` copies them out of that inherited table: the prototype port's write end, the router port's write end, the worker's own pair, the app's shared port and queue, and `log_fd = 2` (`src/nxt_application.c:1789-1809@872bf041`).
- The configuration memory. `process->data.app = nxt_app_conf` (`src/nxt_application.c:829@872bf041`) is a pointer into the prototype; it is not re-sent.
- The loaded language runtime. For PHP, `nxt_php_setup()` runs in the prototype: TSRM start-up when PHP is thread-safe (`src/nxt_php_sapi.c:407-416@872bf041`), `zend_signal_startup()` (`:421`), `sapi_startup()` (`:428`) and `php_module_startup()` through `nxt_php_startup()` (`:445`, `:1400-1405`). For wasm, `nxt_wasm_setup()` initialises the wasmtime runtime in the prototype (`src/wasm/nxt_wasm.c:389-444@872bf041`). Java has a setup hook but creates the JVM in `start` (`src/nxt_java.c:58-66@872bf041`, JVM at `:339`). Python, Ruby and Perl have no setup hook (`src/python/nxt_python.c:49-57@872bf041`, `src/ruby/nxt_ruby.c:93-101@872bf041`, `src/perl/nxt_perl_psgi.c:111-119@872bf041`).
- Isolation state: the rootfs and working directory applied by the prototype (above).
- No listening sockets. Main creates listeners on the router's request (`socket` `src/nxt_main_process.c:1771@872bf041`, `bind` `:1814`) and sends them to the router (`:1735`). The control socket is created in main during runtime configuration (`nxt_runtime_controller_socket()` `src/nxt_controller.c:748-800@872bf041`, called from `src/nxt_runtime.c:1071@872bf041`), before the controller is forked.

Identity and stopping:
- A child whose parent is not main (a worker) sends WHOAMI to main with its own port's write end attached, so main can map namespace-local and global pids (`src/nxt_process.c:946-995@872bf041`, descriptor at `:981-984`).
- In main, SIGTERM and SIGINT start a normal exit and SIGQUIT a graceful one (`src/nxt_main_process.c:1326-1367@872bf041`). Main then tells children to stop with QUIT port messages (`src/nxt_runtime.c:598-606@872bf041`; children of a dead process get QUIT at `src/nxt_main_process.c:1562-1572@872bf041`).
- The other processes also act on signals. Discovery, controller, router and workers install `nxt_process_signals` (`src/nxt_application.c:150@872bf041`, `:172`; `src/nxt_controller.c:203@872bf041`; `src/nxt_router.c:582@872bf041`): SIGINT and SIGTERM call `nxt_runtime_quit()`, SIGQUIT calls `nxt_process_quit()` (`src/nxt_signal_handlers.c:19-27@872bf041`, `:47-66`). So the router and controller stop on a direct or process-group signal, which is what the pytest harness sends.
- SIGUSR1 reopens log files in main and sends ACCESS_LOG to the router (`src/nxt_main_process.c:1371-1391@872bf041`). New log descriptors reach other processes in CHANGE_FILE messages (`src/nxt_port.c:1158@872bf041`).
- Peer death is learned from SIGCHLD and `waitpid()`: in main for its children, and in each prototype for its workers (`nxt_proto_sigchld_handler()` `src/nxt_application.c:1138@872bf041`, `nxt_proto_child_exited()` `:1321`). Both broadcast REMOVE_PID (`nxt_port_remove_notify_others()` `src/nxt_port.c:1286@872bf041`, called at `src/nxt_application.c:1329@872bf041` and `:1450`).

### Inferences
- The prototype is a fork server, but it is also the per-app supervisor. It answers the router's START_PROCESS (`src/nxt_application.c:742-850@872bf041`), relays PROCESS_CREATED (`nxt_proto_process_created_handler()` `:1069`), kills workers that miss the start timeout (`nxt_proto_kill_silent()` `:1036`), and reaps and reports its workers (`:1138`, `:1321`). On Windows it can keep all of that by calling `CreateProcess` for its workers and handing each worker its handles by inheritance at spawn. The router protocol then stays as it is.
- The `setup`/`start` split of `nxt_app_module_t` (`src/nxt_application.h:177-189@872bf041`) maps onto "run setup, then start" inside each Windows worker.
- Per-worker start-up cost rises for PHP and wasm only, the two modules with a setup hook: PHP module start-up and opcache attach, and wasmtime initialisation, happen once per worker instead of once per app. Python, Perl, Ruby and Java already initialise per worker in `start`. Opcache on Windows is built for separate processes: all PHP processes map its shared memory at the same base address ([opcache.mmap_base](https://www.php.net/manual/en/opcache.configuration.php)), so workers can still share one cache.
- Everything a child gets from fork today must become a message, a command-line value or an inherited handle: the app configuration (now a pointer), the subset of the port table, the RPC stream counter mapping, the log handles.
- Handle duplication sets a trust rule, not a topology. The process that calls `DuplicateHandle` needs a handle with `PROCESS_DUP_HANDLE` to the target process, and to the source process when that is not itself ([DuplicateHandle](https://learn.microsoft.com/en-us/windows/win32/api/handleapi/nf-handleapi-duplicatehandle)). Under one account any process can get one with `OpenProcess` and the peer's PID. But such a handle grants full control of the target: Microsoft warns that its holder can duplicate the target's pseudo handle and gain maximum access ([Process Security and Access Rights](https://learn.microsoft.com/en-us/windows/win32/procthread/process-security-and-access-rights)). So the more trusted side must do the duplicating: main for controller and router traffic, the router or the prototype for traffic to workers. Shared-memory sections can be named and opened by name, which needs no duplication at all.
- The Linux isolation features (namespaces, rootfs, cgroups, capabilities; about 3,900 lines across `nxt_isolation.c`, `nxt_capability.c`, `nxt_clone.c`, `nxt_cgroup.c`, `nxt_fs_mount.c`, `nxt_credential.c`) should be out of scope for a first Windows release. Job objects cover resource limits. Per-app users and AppContainer are a separate project.

### Gaps
- The start-up time of a PHP worker on Windows (`php_module_startup()` plus opcache attach) is unmeasured.
- Whether opcache's fixed-base attach is reliable when the workers are started by a non-PHP parent (`unitd.exe`) with ASLR on is unverified; the PHP docs only describe the setting.
- The cost of initialising wasmtime per worker instead of once per prototype is unmeasured.

## 2. Port IPC and shared memory

### Takeaway
A port is a message-preserving `AF_UNIX` socketpair: `SOCK_SEQPACKET` where the configure probe succeeds, else `SOCK_DGRAM`. Each message has a 16-byte header and can carry up to two descriptors (SCM_RIGHTS). On Linux and FreeBSD each message also carries a kernel-validated sender PID, which PR 91 uses to authorise privileged messages. Small messages without descriptors travel through lock-free queues in shared memory, with a socket message used only as a wake-up. Windows has no socketpair, no datagram or seqpacket `AF_UNIX`, and no ancillary data, so the transport must be replaced. The shared-memory layouts hold indices and offsets, not pointers, so they can stay.

### Cited Findings

Table 2.1. IPC and shared-memory Unix APIs.

| Unix API or mechanism | Where (call sites at 872bf041) | Windows counterpart | Grade | Evidence |
|---|---|---|---|---|
| `socketpair(AF_UNIX, ...)` | core 1 `src/nxt_socketpair.c:26@872bf041`; libunit 1 `src/nxt_unit.c:6782@872bf041` | Winsock has no socketpair ([AF_UNIX comes to Windows](https://devblogs.microsoft.com/commandline/af_unix-comes-to-windows/)); emulate with an AF_UNIX listener plus connect, or `CreateNamedPipe` plus `CreateFile` | redesign | Type is `SOCK_SEQPACKET` when the run-probe `auto/sockets:127-142@872bf041` passes, else `SOCK_DGRAM` (`src/nxt_socketpair.c:16-20@872bf041`, `src/nxt_unit.c:6751-6754@872bf041`). |
| message boundaries | server read `src/nxt_port_socket.c:1854@872bf041`; libunit read `src/nxt_unit.c:7907@872bf041` | named pipes with `PIPE_TYPE_MESSAGE` keep boundaries ([pipe modes](https://learn.microsoft.com/en-us/windows/win32/ipc/named-pipe-type-read-and-wait-modes)); Windows AF_UNIX is `SOCK_STREAM` only, so it needs framing | redesign | One `recvmsg()` returns one port message. |
| `sendmsg()`/`recvmsg()` | 2 syscalls: `src/nxt_socket_msg.c:32@872bf041`, `:53`; 4 callers: `src/nxt_socketpair.c:180@872bf041`, `:264`, `src/nxt_unit.c:7548@872bf041`, `:7907`. The brief's pattern `sendmsg|recvmsg` finds 81 mentions in 12 files, mostly comments, tests and the dead `nxt_linux_sendfile.c`. | `WriteFile`/`ReadFile` or `WSASend`/`WSARecv` on the new transport | wrapper (I/O only) | The transport choke point is narrow. |
| SCM_RIGHTS, one or two descriptors | `src/nxt_socket_msg.h:28-42@872bf041` (control buffer sized for 2 ints), `:149-201` (pack), `:236`, `:299` (accept only 1 or 2) | `DuplicateHandle` into the target process; sockets need `WSADuplicateSocket`, which takes the target PID and yields a one-use `WSAPROTOCOL_INFO` blob ([WSADuplicateSocketW](https://learn.microsoft.com/en-us/windows/win32/api/winsock2/nf-winsock2-wsaduplicatesocketw)); DuplicateHandle must not be used on sockets or completion ports ([DuplicateHandle](https://learn.microsoft.com/en-us/windows/win32/api/handleapi/nf-handleapi-duplicatehandle)) | redesign | 18 send sites in 15 message kinds pass descriptors (Table 2.2). |
| sender PID per message (PR 91, [freeunit#91](https://github.com/freeunitorg/freeunit/pull/91), "authorize privileged IPC senders by kernel-validated PID") | `NXT_USE_CMSG_PID` `src/nxt_port.h:293-295@872bf041`; `SO_PASSCRED` `src/nxt_socketpair.c:49-65@872bf041`, `src/nxt_unit.c:6790-6803@872bf041`; SCM_CREDENTIALS or SCM_CREDS `src/nxt_socket_msg.h:13-26@872bf041`; a message without credentials is refused `:331-336`; fail-safe pid -1 `src/nxt_port_socket.c:1903-1912@872bf041` | per connection, not per message: `GetNamedPipeClientProcessId`/`GetNamedPipeServerProcessId` for pipes; `SIO_AF_UNIX_GETPEERPID` for AF_UNIX (defined in [mingw-w64 afunix.h](https://mingw.googlesource.com/mingw-w64/+/refs/heads/v14.x/mingw-w64-headers/include/afunix.h)) | wrapper | 28 uses in 6 files outside log messages (list below). On macOS, NetBSD and OpenBSD the code trusts the self-declared header pid (`src/nxt_port.h:327-332@872bf041`; platform note `src/nxt_port.c:356-373@872bf041`). |
| shared memory creation | server `nxt_shm_open()` `src/nxt_port_memory.c:493-560@872bf041`; libunit `nxt_unit_shm_open()` `src/nxt_unit.c:4958-5020@872bf041`: memfd through `syscall()` (`:509`, `src/nxt_unit.c:4973`), `shm_open(SHM_ANON)` (`src/nxt_port_memory.c:521`), or named `shm_open` + `shm_unlink` (`:533-545`); then `ftruncate()` (`:555`) | `CreateFileMapping(INVALID_HANDLE_VALUE, ...)`, backed by the paging file ([CreateFileMappingW](https://msdn.microsoft.com/en-us/library/Aa366537.aspx)), shared by duplicating the section handle | wrapper | The segment descriptor then travels as SCM_RIGHTS. |
| `mmap()`/`munmap()` | core: one wrapper `src/nxt_mem_map.c:16@872bf041`, `:34`; libunit: 6 mmap (`src/nxt_unit.c:649@872bf041`, `:680`, `:1450`, `:4900`, `:5166`, `:6644`), 7 munmap | `MapViewOfFile`/`UnmapViewOfFile` | wrapper | Every map is a whole segment at offset 0, so the rule that view offsets be multiples of the allocation granularity ([MapViewOfFile](https://msdn.microsoft.com/en-us/library/aa366761)) places no constraint on FreeUnit. |
| size check of a received segment (PR 173, [freeunit#173](https://github.com/freeunitorg/freeunit/pull/173), "validate peer-supplied incoming mmap id and geometry") | `fstat()` `src/nxt_port_memory.c:295-316@872bf041`; header snapshot and pid and id checks `:335-356`; libunit `fstat()` `src/nxt_unit.c:5145@872bf041` | query the section size before mapping (exact API to be chosen, see Gaps) | wrapper | macOS rounds shm objects up to a 16 KiB page, so the check accepts a longer object (`src/nxt_port_memory.c:301-309@872bf041`). |
| atomics | `src/nxt_atomic.h:17-54@872bf041`: only the GCC `__sync_*` branch (`NXT_HAVE_GCC_ATOMIC`); `nxt_cpu_pause()` `:57-67`; the other branch is commented out (`:70-86`); `__builtin_ffsll` `src/nxt_port_memory_int.h:267@872bf041` | clang (including clang-cl) and MinGW-w64 GCC accept the same builtins; MSVC `cl` needs `Interlocked*` and `_BitScanForward64` | none with clang or MinGW; wrapper with MSVC | No C11 atomics and no explicit fences beyond the `__sync` full barriers. |

PR 91 uses of `nxt_recv_msg_cmsg_pid()` outside log messages (28 in 6 files): `src/nxt_main_process.c:590@872bf041`, `:749`, `:890`, `:1059`, `:1212`, `:1681`, `:1913`, `:2040`, `:2055`, `:2231`, `:3189`, `:3211`; `src/nxt_application.c:725@872bf041`, `:735`, `:1074`; `src/nxt_port.c:752@872bf041`; `src/nxt_router.c:1669@872bf041`, `:1710`, `:1740`, `:1747`, `:1827`, `:9051`; `src/nxt_cert.c:1516@872bf041`, `:1734`, `:1870`; `src/nxt_script.c:607@872bf041`, `:789`, `:929`.

Table 2.2. Messages that carry descriptors (18 send sites).

| Message | Descriptor(s) | Sender to receiver | Parent and child? | Citation |
|---|---|---|---|---|
| START_PROCESS (to main only) | app shared port read end and app queue segment | router to main (new prototype) | yes | `src/nxt_router.c:708@872bf041`, `:721-722`, `:746`, `:4564-4567`, `:4604`; stored by main `src/nxt_main_process.c:626-627@872bf041`. The START_PROCESS the router sends a prototype for a new worker carries no descriptors (`src/nxt_router.c:688-693@872bf041`, `:4572-4574`). |
| reply to SOCKET | listening socket | main to router | yes | `src/nxt_main_process.c:1735@872bf041` |
| reply to ACCESS_LOG | access log file | main to router | yes | `src/nxt_main_process.c:3215@872bf041` |
| reply to CERT_GET | certificate file | main to the requesting process | yes | `src/nxt_cert.c:1568@872bf041` |
| CERT_STORE | shared memory with the certificate | controller to main | yes | `src/nxt_cert.c:1642@872bf041` (segment from `:1608`) |
| reply to SCRIPT_GET | script file | main to the requesting process | yes | `src/nxt_script.c:667@872bf041` |
| SCRIPT_STORE | shared memory | controller to main | yes | `src/nxt_script.c:742@872bf041` |
| CONF_STORE | shared memory with the configuration | controller to main | yes | `src/nxt_controller.c:3106@872bf041` (main maps it at `src/nxt_main_process.c:2260@872bf041`) |
| DATA (new configuration) | shared memory with the configuration | controller to router | siblings | `src/nxt_controller.c:727@872bf041` (router maps it at `src/nxt_router.c:1969@872bf041`) |
| NEW_PORT | port write end and port queue segment | parent (main or prototype) to every process allowed by `nxt_proc_send_matrix` | prototype to router: siblings | `src/nxt_port.c:540@872bf041`; broadcast on PROCESS_READY `:1069`, `:467-511`; matrix `src/nxt_process.c:89-96@872bf041` |
| CHANGE_FILE | reopened log file | main to others | yes | `src/nxt_port.c:1158@872bf041` |
| WHOAMI | sender's own port write end | worker to main | grandchild | `src/nxt_process.c:983@872bf041` |
| REQ_BODY | temporary file with a buffered request body | router to worker | no | `src/nxt_router.c:6482@872bf041`; file made by `mkstemp()` and unlinked at once, `src/nxt_http_request.c:737-746@872bf041` |
| MMAP (router side) | router's outgoing segment | router to worker | no | `src/nxt_router.c:9239@872bf041` (answers libunit's GET_MMAP, `src/nxt_unit.c:5604@872bf041`) |
| PROCESS_READY (libunit) | worker's port queue segment | worker to prototype | yes | `src/nxt_unit.c:1097-1115@872bf041` |
| MMAP (libunit) | worker's outgoing segment (response data) | worker to router | no | `src/nxt_unit.c:5031-5052@872bf041` |
| NEW_PORT (libunit) | per-thread port write end and its queue | worker thread to router | no | `src/nxt_unit.c:6844-6876@872bf041` |

The message header and the size rules:
- `nxt_port_msg_t` is `stream` (u32), `pid` (`pid_t`), `reply_port` (u16), `type` (u8) and four one-bit flags stored as u8: `last`, `mmap`, `nf`, `mf` (`src/nxt_port.h:251-271@872bf041`; `nxt_port_id_t` is `uint16_t`, `src/nxt_main.h:39@872bf041`). That is 15 bytes of fields, 16 with padding on common ABIs.
- There are 38 message types; a static assert pins the last one at 37 (`src/nxt_port.h:209@872bf041`).
- A socket message is at most `port->max_size`: 16 KiB by default, capped by `SO_SNDBUF`; buffers are grown with `setsockopt` (`src/nxt_port_socket.c:126-200@872bf041`). `max_share` is 64 KiB (`:188`).
- Larger payloads are split into fragments (the `mf` flag; reassembly `src/nxt_port_socket.c:2331-2653@872bf041`) or moved into shared memory (the `mmap` flag).

Shared memory layout and life cycle:
- A segment is a 4 KiB header plus 10 MiB of data in 16 KiB chunks (`src/nxt_port_memory_int.h:24-32@872bf041`).
- The header holds `id`, `src_pid`, `dst_pid`, `sent_over`, an `oosm` flag and atomic free bitmaps (`src/nxt_port_memory_int.h:93-124@872bf041`). A chunk address is the segment base plus an offset (`:177-193`). The message names a chunk by `mmap_id`, `chunk_id` and `size` (`:153-157`).
- The sender of data creates the segment: the worker for responses (`src/nxt_unit.c:4895-4933@872bf041`), the router for request bodies. The peer learns of it in an MMAP message with the descriptor attached (Table 2.2).
- Freed chunks are acknowledged with SHM_ACK when the sender ran out of memory (`src/nxt_port_memory.c:1147@872bf041`).
- When a peer dies, main reaps it and broadcasts REMOVE_PID (`src/nxt_port.c:1286@872bf041`); libunit handles REMOVE_PID at `src/nxt_unit.c:1289@872bf041`.

Queues in shared memory and dispatch:
- A port queue is `nitems` plus two NNCQ rings of 16,384 u32 entries and 16,384 items of 1 + 31 bytes (`src/nxt_port_queue.h:15-30@872bf041`; `NXT_NNCQ_SIZE` `src/nxt_nncq.h:12@872bf041`; ring `:25-28`). The app queue has the same shape with a `notified` word and a u32 `tracking` per item (`src/nxt_app_queue.h:15-30@872bf041`).
- The queues hold indices and inline bytes, not pointers. The NNCQ ring places `head`, `entries` and `tail` next to each other with no cache-line padding (`src/nxt_nncq.h:25-28@872bf041`).
- The router creates one shared port and one app queue per application (`src/nxt_router.c:2047-2067@872bf041`, `:3968-3992`) and a queue for each worker port (`:3996-4020`). Requests go into the app queue (`:8291`).
- Send path: a message without a descriptor that fits 31 bytes goes into the queue. On a port queue the socket gets a READ_QUEUE wake-up only when the item count goes from zero to one (`src/nxt_port_socket.c:347-401@872bf041`; rule explained at `:443-445`; count at `src/nxt_port_queue.h:84@872bf041`). A message with a descriptor goes over the socket, and a one-byte READ_SOCKET marker goes into the queue to keep order (`src/nxt_port_socket.c:402-418@872bf041`).
- The app queue has no item count. The router raises a wake-up when it flips `notified` from 0 to 1 (`src/nxt_app_queue.h:84-88@872bf041`), and the worker that reads the READ_QUEUE message resets it (`nxt_app_queue_notification_received()` `:95-97`, called at `src/nxt_unit.c:7821-7822@872bf041`).
- Every worker of an app holds the same shared-port read end, inherited from the prototype (`src/nxt_application.c:1806-1807@872bf041`, `src/nxt_unit.c:606@872bf041`).
- libunit waits by draining its own queue, then the app queue, then calling `poll()` on its own port socket and the shared port socket (`src/nxt_unit.c:5899-5991@872bf041`). No eventfd, pipe, futex or semaphore is used for IPC wake-ups. The only eventfd is the epoll engine's own wake-up (`src/nxt_epoll_engine.c:821@872bf041`).

Other shared-memory users, all moving a whole buffer through a descriptor: controller configuration (`src/nxt_controller.c:702-714@872bf041`, `:3083-3095`), certificates (`src/nxt_cert.c:1608-1619@872bf041`), scripts (`src/nxt_script.c:708-719@872bf041`).

### Inferences
- The transport can be swapped behind `nxt_socketpair_send_ex()`/`nxt_socketpair_recv()` and `nxt_sendmsg()`/`nxt_recvmsg()`, but three semantics travel with it and are the real work: message boundaries, descriptor passing between processes that are not parent and child, and per-message sender authentication.
- On Windows, sender authentication moves from "each message" to "each connection". Every port has many writers today: a port's write end goes to every process the send matrix allows (`src/nxt_port.c:467-511@872bf041`) and is inherited through the keep matrix (`src/nxt_process.c:80-87@872bf041`, `:372-394`); main's port is held by every process. So per-connection identity needs one connection per writer and port. Each port becomes a pipe server with one instance per writer, and NEW_PORT carries a pipe name instead of a descriptor.
- With named ports, the port-descriptor rows of Table 2.2 (NEW_PORT twice, WHOAMI) disappear. What is left to pass is files, sockets and shared-memory sections. Files become handle values duplicated into the receiver by the more trusted side (section 1). Sockets (listeners from main to the router) need `WSADuplicateSocket` with the router's PID. Sections can be named and opened by name, with the receiver keeping the PR 173 checks (`src/nxt_port_memory.c:295-356@872bf041`).
- The shared port's own problem is many readers, not many writers (section 4).
- Two descriptors per message and 18 send sites keep the handle-passing change small in count. The difficulty is the duplication protocol and the cleanup when a message is dropped.
- The shared-memory layouts are position independent, so unlike nginx's Windows port (which maps named sections at a fixed base address, `ngx_shmem.c:49`, `:62`, `:82` in [nginx src/os/win32/ngx_shmem.c](https://github.com/nginx/nginx/blob/2b5c2b605b5df669da5dec6749dcc76c07d1315d/src/os/win32/ngx_shmem.c)), FreeUnit does not need fixed-address mapping.
- The page-size assumption is mild. Segment sizes are compile-time constants not tied to the page size, and the PR 173 check already tolerates OS rounding. Windows maps whole sections, so the 64 KiB granularity applies only to the address the view lands at.

### Gaps
- The exact Win32 call to read a section's size before mapping (the replacement for `fstat()` in `src/nxt_port_memory.c:295@872bf041`) was not chosen or verified. Settle it with a test that maps a peer-created section and compares `VirtualQuery` on the view with the size the sender declared.
- Whether `ProcessSocketNotifications` accepts AF_UNIX sockets (its docs say only Microsoft Winsock provider sockets are supported) is unverified. Settle it with a small program on Windows Server 2022.
- Message rate and latency of message-mode named pipes against AF_UNIX stream plus framing, for 16-byte headers and 16 KiB payloads, is unmeasured.

## 3. Event engine, connections, sockets, timers, threads and static files

### Takeaway
The engine is a readiness model inherited from nginx: a 21-operation vtable with seven back ends. The select and poll engines are always compiled as fallbacks (`auto/sources:312-313@872bf041`) but have no post, signal or file support of their own. Windows has a documented readiness API for sockets, `ProcessSocketNotifications`, with level, edge, oneshot and persistent triggers on an I/O completion port, from Windows build 20348 (the Windows Server 2022 build). With it, a new engine keeps the existing contract (wrapper grade). Without it, the choice is an undocumented AFD poll, `WSAPoll` with a known connect bug, or a completion-based rewrite of the connection layer. Static files are opened on the engine thread, read into memory buffers and sent with `writev`. `sendfile()` is used only when a proxied request body that was spooled to a file is sent upstream. The six `nxt_*_sendfile.c` files are dead code.

### Cited Findings

The vtable `nxt_event_interface_t` (`src/nxt_event_engine.h:23-198@872bf041`): `name`; `create` (`:34`), `free` (`:38`), `enable` (`:45`), `disable` (`:49`), `delete` (`:56`), `close` (`:67`), `cancel_changes` (`:98`), `enable_read` (`:105`), `enable_write` (`:112`), `disable_read` (`:116`), `disable_write` (`:120`), `block_read` (`:124`), `block_write` (`:128`), `oneshot_read` (`:135`), `oneshot_write` (`:142`), `enable_accept` (`:149`), `enable_file` (`:156`), `close_file` (`:162`), `enable_post` (`:169`), `signal` (`:183`), `poll` (`:187`); then `io` (`:191`, the connection I/O vtable `nxt_conn_io_t`, `src/nxt_conn.h:39-87@872bf041`) and the `file_support` and `signal_support` flags (`:194`, `:197`).

Engines: seven facilities and eight engine objects (kqueue, epoll edge, epoll level, eventport, devpoll, pollset, poll and select), chosen by name from `src/nxt_service.c:12-40@872bf041`; a process switches to its configured engine in `nxt_process_setup()` (`src/nxt_process.c:856-863@872bf041`).

Table 3.1. Engine and I/O Unix APIs.

| Unix API or mechanism | Where (call sites at 872bf041) | Windows counterpart | Grade | Evidence |
|---|---|---|---|---|
| readiness engines (epoll, kqueue, devpoll, eventport, pollset) | 4,392 lines in 5 files, compiled per probe (`auto/sources:286-308@872bf041`) | a new engine on an I/O completion port with `ProcessSocketNotifications` (minimum build 20348; one IOCP per socket at a time, [ProcessSocketNotifications](https://learn.microsoft.com/en-us/windows/win32/api/winsock2/nf-winsock2-processsocketnotifications)); triggers `SOCK_NOTIFY_TRIGGER_LEVEL`, `EDGE`, `ONESHOT`, `PERSISTENT`; events IN, OUT, HANGUP ([SOCK_NOTIFY_REGISTRATION](https://learn.microsoft.com/en-us/windows/win32/api/winsock2/ns-winsock2-sock_notify_registration)) | wrapper (new back end) | The contract is readiness: handlers call `recv()`/`send()` themselves when told the socket is ready. |
| select engine | `src/nxt_select_engine.c` (384 lines): handler table indexed by fd, sized `FD_SETSIZE` (`src/nxt_select_engine.c:77@872bf041`, `:150`); `select()` `:329`; no post, signal or file entries | Winsock `select()` takes `SOCKET` arrays, so the fd-indexed table must change | wrapper | Engine vtable rows: `enable_file`, `close_file`, `enable_post`, `signal` are NULL. |
| poll engine | `src/nxt_poll_engine.c` (737 lines): fd hash `src/nxt_poll_engine.c:17-26@872bf041`, `:101-106`; `poll()` `:564` | `WSAPoll` (sockets only); it does not report failed connects, a bug Microsoft closed as Won't Fix ([curl: WSAPoll is broken](https://daniel.haxx.se/blog/?p=4300)) | wrapper, fallback only | Same NULL entries as select. |
| cross-thread post and wake-up | eventfd `src/nxt_epoll_engine.c:821@872bf041`; EVFILT_USER `src/nxt_kqueue_engine.c:687@872bf041`, `:717`; otherwise a pipe (`src/nxt_event_engine.c:153-200@872bf041`, `nxt_pipe_create` `:185`) | `PostQueuedCompletionStatus` | wrapper | A pipe cannot be waited on by a socket readiness API, so the fallback does not carry over. |
| signals into the engine | signalfd `src/nxt_epoll_engine.c:736@872bf041`; EVFILT_SIGNAL `src/nxt_kqueue_engine.c:661@872bf041`; `sigwait()` thread `src/nxt_signal.c:153-167@872bf041` | console or service control handler thread that posts a completion | wrapper | |
| timers | red-black tree `src/nxt_timer.h:62-67@872bf041`; poll timeout from `nxt_timer_find()` `src/nxt_event_engine.c:550-552@872bf041` | the timeout argument of `GetQueuedCompletionStatusEx` | none | |
| clocks | `clock_gettime` core 6 (`src/nxt_time.c:29@872bf041`, `:53`, `:76`, `:117`, `:140`, `:194`), libunit 2; `gettimeofday` core 3, libunit 1 | `QueryPerformanceCounter`, `GetTickCount64`, `GetSystemTimePreciseAsFileTime` | wrapper | Coarse and fast clock variants are probe-selected (`src/nxt_time.c:15-140@872bf041`). |
| `accept4()`/`accept()` | `accept4` call `src/nxt_epoll_engine.c:1079@872bf041` (`:307` is a run-time availability probe); `accept` `src/nxt_conn_accept.c:200@872bf041`, `src/nxt_kqueue_engine.c:1023@872bf041` | `accept` plus `ioctlsocket(FIONBIO)`, or `AcceptEx` | wrapper | |
| one listening socket shared by all router engine threads | `EPOLLEXCLUSIVE` in `nxt_epoll_enable_accept()` `src/nxt_epoll_engine.c:616-625@872bf041`; no `SO_REUSEPORT` anywhere in `src/` | one IOCP shared by the router threads, or one accept thread that hands sockets off; a socket can belong to only one IOCP registration | redesign (small) | Node's cluster module keeps OS scheduling on Windows until libuv can distribute IOCP handles without a performance cost ([Node cluster](https://nodejs.org/api/cluster.html)). |
| `readv`, `writev`, `recv`, `send` | `readv` `src/nxt_conn_read.c:153@872bf041`; `recv` `:204`; `writev` `src/nxt_conn_write.c:335@872bf041`, `:491`; `send` `:377`, `:531` | `WSARecv`/`WSASend` with `WSABUF` arrays | wrapper | |
| static file send | `nxt_http_static_body_handler()` allocates memory buffers (`src/nxt_http_static.c:1899@872bf041`), `nxt_file_read()` fills them (`:1964`), `nxt_http_request_send()` sends them (`:1994`) | `ReadFile` with an `OVERLAPPED` offset, then `WSASend` | wrapper | No sendfile on this path. |
| sendfile (proxied request body only) | `nxt_sendfile()` `src/nxt_conn_write.c:270-319@872bf041`: macOS, FreeBSD and Linux branches, else `mmap()` plus `write()` (`:295-316`); called only when a file buffer heads the write chain (`:201-202`); the one producer is a request body spooled to a file and sent to a proxy upstream (`src/nxt_h1proto.c:2974-2991@872bf041`) | `TransmitFile`; client editions of Windows run at most two `TransmitFile` operations at a time, server editions have no default limit ([TransmitFile](https://learn.microsoft.com/en-us/windows/win32/api/mswsock/nf-mswsock-transmitfile)) | wrapper | The two-operation limit touches only this proxy path. |
| `nxt_*_sendfile.c` (Linux, FreeBSD, macOS, Solaris, HP-UX, AIX; 967 lines) | assigned only to the `old_sendbuf` slot (`src/nxt_conn.c:29-39@872bf041`; also `src/nxt_epoll_engine.c:113-115@872bf041` and `src/nxt_kqueue_engine.c:127-131@872bf041`; slot `src/nxt_conn.h:72@872bf041`), which nothing reads | none | none | Compiled on the matching OS but unreachable at this revision. |
| socket options | `SO_REUSEADDR` `src/nxt_listen_socket.c:64@872bf041`, `src/nxt_main_process.c:1791@872bf041`; `IPV6_V6ONLY` `src/nxt_listen_socket.c:68-76@872bf041`, `src/nxt_main_process.c:1803@872bf041`; `TCP_DEFER_ACCEPT` under `#ifdef` `src/nxt_socket.c:65-68@872bf041`; `SOCK_NONBLOCK` `:20-23`; `SO_ERROR` `:329`; `TCP_NODELAY` `src/nxt_conn.h:226@872bf041`, `:240` | same names in Winsock; `SO_REUSEADDR` has different semantics on Windows (use `SO_EXCLUSIVEADDRUSE`, not verified here) | none or wrapper | `SO_ACCEPTFILTER` is not used. |
| Unix-domain listeners, abstract names | abstract `unix:@name` printing and parsing `src/nxt_sockaddr.c:292-295@872bf041`, `:607-618`; listener `unlink` `src/nxt_listen_socket.c:115@872bf041`; `chmod 0666` `src/nxt_main_process.c:1866@872bf041` | Windows AF_UNIX accepts abstract addresses but has no autobind ([AF_UNIX comes to Windows](https://devblogs.microsoft.com/commandline/af_unix-comes-to-windows/)) | wrapper | |
| static file open with `openat2()` resolve rules | `RESOLVE_NO_SYMLINKS` `src/nxt_http_static.c:280@872bf041`, `RESOLVE_NO_XDEV` `:286`, `RESOLVE_IN_ROOT` `:577`; opens `:585-623`; `openat2` through `syscall()` `src/nxt_file.c:63@872bf041`; gated by `NXT_HAVE_OPENAT2` (`src/nxt_http_static.c:13@872bf041`, `:251`) | `CreateFile` plus a path walk that checks reparse points and volume changes | redesign (feature) | `chroot`, `follow_symlinks` and `traverse_mounts` depend on it. |
| blocking file I/O | static files are opened and read on the engine thread. A thread pool object is created (`src/nxt_runtime.c:330@872bf041`, `src/nxt_process.c:865@872bf041`), but its threads start only on the first post (`src/nxt_thread_pool.c:42-66@872bf041`) and only the pool itself posts (`:250`, `:292`) | same behaviour on Windows | none | |
| threads and locks | `pthread_create` `src/nxt_thread.c:85@872bf041`; mutex `src/nxt_thread_mutex.c:137@872bf041`; condition `src/nxt_thread_cond.c:72@872bf041`; semaphores `src/nxt_semaphore.c:37@872bf041`, `:64`, `:90`; `sched_yield` `src/nxt_process.h:291-292@872bf041`; thread id through `syscall()` `src/nxt_thread_id.h:23@872bf041` | `_beginthreadex`, SRW locks, condition variables, `CreateSemaphore`, `SwitchToThread`, `GetCurrentThreadId` | wrapper | |
| thread-local storage | `__thread` or pthread keys behind one interface (`src/nxt_thread.h:14-52@872bf041`; `pthread_key_create` `src/nxt_thread.c:34@872bf041`) | `__declspec(thread)` or `TlsAlloc` | wrapper | |
| CPU count and page size | `sysconf(_SC_NPROCESSORS_ONLN)` `src/nxt_lib.c:97-99@872bf041`; `sched_getaffinity` `:115`; `getpagesize` `:157` | `GetActiveProcessorCount`, `GetSystemInfo` | wrapper | |
| descriptor types | `nxt_fd_t` is `int` (`src/nxt_file.h:11@872bf041`), invalid is `-1` (`:13`); `nxt_socket_t` is `int` (`src/nxt_socket.h:11@872bf041`) | `HANDLE` (a pointer) and `SOCKET` (`UINT_PTR`) with their own invalid values | redesign (wide, mostly mechanical) | The select engine uses the fd as an array index; the other engines and the poll hash do not. |

### Inferences
- The best fit is a `ProcessSocketNotifications` engine for sockets, with named-pipe port I/O as overlapped operations on the same completion port, `PostQueuedCompletionStatus` for `enable_post`, and the `GetQueuedCompletionStatusEx` timeout for timers. It keeps `nxt_conn_io_t` (`src/nxt_conn.h:39-87@872bf041`) and every read and write handler unchanged.
- That engine sets the minimum Windows version at build 20348, which is Windows Server 2022; Windows 11 builds are higher.
- A completion-only engine (classic IOCP with `WSARecv` posted in advance) would change who owns buffers while an operation is pending. Every reader in `nxt_conn_read.c`, the proxy and TLS would need rework. Avoid it for a first port.
- nginx's own Windows build shows the cost of not doing this work. It uses only `select()` and `poll()`, only one worker actually serves, and the port is labelled beta ([nginx for Windows](https://nginx.org/en/docs/windows.html)).
- `TransmitFile`'s two-operation limit on client editions affects only the proxy request-body path, which can keep `nxt_sendfile()`'s read-and-send fallback.

### Gaps
- Whether one listening socket can be served by several router engine threads under `ProcessSocketNotifications` is unknown, because a socket has one registration at a time. Settle it with a test: one IOCP drained by N threads versus an accept thread with hand-off.
- `WSAPoll`'s connect bug was reported in 2012 and marked Won't Fix. No source found on whether current Windows builds behave differently.
- `SO_REUSEADDR` and `SO_EXCLUSIVEADDRUSE` semantics on Windows were not re-verified in this pass.
- Two ways to get read readiness without build 20348 were not evaluated: a zero-byte overlapped `WSARecv` on a plain completion port (libuv's TCP code uses this, `UV_HANDLE_ZERO_READ` in `src/win/tcp.c:565` and `:1058-1078` at [libuv 954c1d1](https://github.com/libuv/libuv/blob/954c1d1887bfdb223b2b0dc2201199f938902e68/src/win/tcp.c)), and AFD polling (libuv's `uv_poll` uses `AFD_POLL_INFO`, `src/win/poll.c:50` and `:113` at [libuv 954c1d1](https://github.com/libuv/libuv/blob/954c1d1887bfdb223b2b0dc2201199f938902e68/src/win/poll.c)). Either could lower the minimum Windows version.

## 4. libunit and the language integrations

### Takeaway
libunit is one C file (8,713 lines) plus four server objects. It shares the port header, the segment layout, the queue layouts and descriptor passing with the router, so it cannot be ported apart from the router-side transport. Its own Unix surface is small and concentrated: `socketpair`, the `nxt_socket_msg` send and receive, `poll()` on two descriptors, memfd or `shm_open` plus `mmap`, and pthread mutexes. The real problem is its public API. It exposes `int` descriptors, and three integrations (Node, Python ASGI, Go) put those descriptors into their own event loops, which on Windows accept only sockets or no descriptors at all.

### Cited Findings

Table 4.1. libunit Unix APIs.

| Unix API or mechanism | Where (call sites at 872bf041) | Windows counterpart | Grade | Evidence |
|---|---|---|---|---|
| per-context ports | `socketpair` `src/nxt_unit.c:6782@872bf041` (type `:6751-6754`); `SO_PASSCRED` `:6790-6803`; queue segment `:6639-6647`; sent to the router with descriptors `:6844-6876` | same as the server transport | redesign | |
| send and receive | `nxt_sendmsg` `src/nxt_unit.c:7548@872bf041`; `nxt_recvmsg` `:7907`; both from `src/nxt_socket_msg.c` | same as the server transport | wrapper on the new transport | |
| wait loop | `nxt_unit_read_buf()` `src/nxt_unit.c:5899-6042@872bf041`: `poll()` on the context port and the shared port (`:5991`); single-descriptor `poll()` `:4024-4036` | `WaitForMultipleObjects` on an event or semaphore per port, or an IOCP per context | redesign | Queues are drained before the wait (`src/nxt_unit.c:5946-5974@872bf041`); the shared port is polled only when the context is ready (`:5970-5983`); a context with waiting items receives directly (`:5937-5939`). |
| shared memory | `nxt_unit_shm_open()` `src/nxt_unit.c:4958-5027@872bf041` (memfd `syscall` `:4973`); 6 `mmap`, 7 `munmap`; `fstat` `:5145` | `CreateFileMapping`, `MapViewOfFile` | wrapper | |
| locks | `pthread_mutex_lock` 34 calls, `pthread_mutex_init` 3, `pthread_mutex_destroy` 4, `pthread_self` 3 (all `src/nxt_unit.c@872bf041`) | SRW locks or critical sections | wrapper | |
| request body file | `read()` of `content_fd` `src/nxt_unit.c:3546@872bf041`, `:3655`; the descriptor arrives with REQ_HEADERS (`:1730`) or REQ_BODY (`:1829`) | `ReadFile` on the duplicated handle | wrapper | |
| log descriptor | `dup2()` on CHANGE_FILE `src/nxt_unit.c:1253@872bf041`; `write(log_fd)` `:8526`, `:8576` | `SetStdHandle` or a CRT descriptor from `_open_osfhandle` | wrapper | |
| bootstrap for external apps | `getenv("NXT_UNIT_INIT")` `src/nxt_unit.c:1012@872bf041`; `sscanf` of pids and fd numbers `:1043-1051` | inheritable handle values printed as integers (handle values are the same in the child) | wrapper | Format written by `src/nxt_external.c:121-134@872bf041`. |
| `ioctl(FIONBIO)` | `src/nxt_unit.c:8263@872bf041` | `ioctlsocket` | wrapper | |
| public API with `int` descriptors and `pid_t` | `nxt_unit_port_t.in_fd`, `out_fd` (`src/nxt_unit.h:81-82@872bf041`); `nxt_unit_init_t.shared_port_fd`, `shared_queue_fd`, `log_fd` (`:179-181`); `remove_pid(pid_t)` (`:143`); `nxt_unit_port_id_init(..., pid_t, ...)` (`:286`) | a handle type, or an opaque wait object | redesign (ABI break for external apps) | |

libunit's build and link:
- `libunit.a` is `nxt_unit.o` plus `nxt_lvlhsh.o`, `nxt_murmur_hash.o`, `nxt_socket_msg.o` and `nxt_websocket.o` (`auto/make:120-130@872bf041`, the four objects at `:121-124`; `NXT_LIB_UNIT_SRCS="src/nxt_unit.c"` `auto/sources:109@872bf041`).
- A language module links only `nxt_unit.o` and its own objects (`auto/modules/php:604@872bf041`, link rule `:643-646`). The helpers that `nxt_unit.o` needs are resolved from `unitd` at load time, because `unitd` is linked with `-Wl,-E` (`auto/os/conf:31@872bf041`); see section 5.
- The Go package links `libunit.a` with `-lunit` (`go/ldflags.go@872bf041`), plus `-lrt` on Linux and NetBSD (`go/ldflags-lrt.go:1@872bf041`).

Callbacks in `nxt_unit_callbacks_t` (`src/nxt_unit.h:125-160@872bf041`), 12 in all: `request_handler`, `data_handler`, `websocket_handler`, `close_handler`, `add_port`, `remove_port`, `remove_pid`, `quit`, `shm_ack_handler`, `port_send`, `port_recv`, `ready_handler`.

Table 4.2. How each integration drives libunit.

| Integration | Callbacks set besides `request_handler` | Raw descriptor handed to a foreign loop | Windows issue |
|---|---|---|---|
| Node.js (`src/nodejs/unit-http/unit.cpp`) | `add_port`, `remove_port` (`src/nodejs/unit-http/unit.cpp:363-364@872bf041`), `close_handler`, `quit`, `shm_ack_handler`, `websocket_handler` | yes: `uv_poll_init(loop, ..., port->in_fd)` `src/nodejs/unit-http/unit.cpp:586@872bf041`, `uv_poll_start` `:593`, `O_NONBLOCK` with `fcntl` `:570` | On Windows only sockets can be polled with `uv_poll` ([libuv poll](https://docs.libuv.org/en/v1.x/poll.html)); libuv can pass only TCP handles over IPC on Windows ([libuv uv_write2](https://docs.libuv.org/en/v1.x/stream.html)). |
| Python ASGI (`src/python/nxt_python_asgi.c`) | `ready_handler` (set for both protocols at `src/python/nxt_python.c:319@872bf041`, before `nxt_python_asgi_init()` at `:339`), then `add_port`, `remove_port`, `close_handler`, `data_handler`, `quit`, `shm_ack_handler`, `websocket_handler` (`src/python/nxt_python_asgi.c:193-200@872bf041`) | yes: `loop.add_reader(port->in_fd, ...)` `src/python/nxt_python_asgi.c:1050-1080@872bf041`; `ioctl(FIONBIO)` `:1026` | The default Windows loop (Proactor) does not support `add_reader`; the Selector loop accepts only sockets, up to 512 ([asyncio platforms](https://docs.python.org/3/library/asyncio-platforms.html)). |
| Go (`go/nxt_cgo_lib.c`) | `add_port`, `remove_port`, `port_send`, `port_recv`, `shm_ack_handler`, `ready_handler` (`go/nxt_cgo_lib.c:27-33@872bf041`) | yes: `dup` plus `net.FileConn` into a `*net.UnixConn` (`go/port.go:81-136@872bf041`) | Go does its own datagram send and receive with descriptor passing. An upstream user could not build a Go app for Windows (errors on `setresgid`, `setresuid`), with no maintainer reply shown ([nginx/unit#1008](https://github.com/nginx/unit/issues/1008)). |
| Java (`src/nxt_java.c`) | `ready_handler`, `close_handler`, `websocket_handler` | no; worker threads with `pthread_create` (`src/nxt_java.c:626@872bf041`) | threads only |
| Python WSGI (`src/python/nxt_python.c`, `nxt_python_wsgi.c`) | `ready_handler` | no; threads `src/python/nxt_python.c:682@872bf041` | threads only |
| Perl (`src/perl/nxt_perl_psgi.c`) | `ready_handler` | no; threads `src/perl/nxt_perl_psgi.c:1286@872bf041` | threads only |
| Ruby (`src/ruby/nxt_ruby.c`) | `ready_handler` | no | none found |
| PHP (`src/nxt_php_sapi.c`) | none | no; one context, `nxt_unit_run()` (`src/nxt_php_sapi.c:537@872bf041`) | none |
| wasm (`src/wasm/nxt_wasm.c`) and wasm-wasi-component (`src/wasm-wasi-component/src/lib.rs`) | none | no | none |

### Inferences
- libunit cannot be ported on its own. A Windows libunit can be unit-tested against a stub router, but end to end it needs the router's new transport, the handle broker and the wake-up objects.
- The public API needs a Windows-specific wait abstraction. The smallest change keeps `in_fd` as an opaque `intptr_t` that holds a socket or handle, and adds a wait handle per port for integrations that cannot poll it.
- Node can keep `uv_poll` only if each port is a socket (AF_UNIX stream). If ports become named pipes, the addon must instead run libunit's wait on a thread and signal libuv with `uv_async`.
- Python ASGI on Windows must either force the Selector loop and socket ports, or run the libunit wait on a thread and use `call_soon_threadsafe`.
- Go needs a new transport in `go/port.go`, or should drop its own `port_send`/`port_recv` and let the C side do I/O.
- Breaking the libunit ABI for external apps is unavoidable. Keep the Unix ABI as it is and give Windows its own.

### Gaps
- Whether a libuv-based Node addon can watch a Windows named pipe without a helper thread was not checked against libuv's pipe API.
- Go's `net` package behaviour on Windows AF_UNIX (stream only, no out-of-band data) was not confirmed from Go sources in this pass.

## 5. Platform layer, module loading and the size of OS-specific code

### Takeaway
The platform layer is nginx-style and partly Windows-aware by heritage: an errno abstraction with Windows-flavoured names, a separate `nxt_socket_errno`, a symbol-visibility macro and a vestigial `MINGW*` branch in configure. All real code paths are POSIX. OS-specific code today is about 13,600 lines: 25 OS-specific files (9,809 lines) and 3,783 lines inside OS conditionals in 51 other core and libunit files. Language modules resolve a few core functions from the `unitd` executable at load time, which on Windows needs an import library for `unitd.exe` or static linking.

### Cited Findings

Table 5.1. OS-specific source files.

| File(s) | Lines | Provides | Compiled when |
|---|---|---|---|
| `src/nxt_epoll_engine.c` | 1,225 | epoll engine with eventfd and signalfd | `NXT_HAVE_EPOLL` (`auto/sources:286-289@872bf041`) |
| `src/nxt_kqueue_engine.c` | 1,118 | kqueue engine | `NXT_HAVE_KQUEUE` (`auto/sources:291-293@872bf041`) |
| `src/nxt_devpoll_engine.c`, `nxt_eventport_engine.c` | 704, 657 | Solaris engines | `auto/sources:296-305@872bf041` |
| `src/nxt_pollset_engine.c` | 688 | AIX engine | `auto/sources:307-309@872bf041` |
| `src/nxt_poll_engine.c`, `nxt_select_engine.c` | 737, 384 | portable fallbacks | always (`auto/sources:312-313@872bf041`) |
| `src/nxt_linux_sendfile.c`, `nxt_freebsd_sendfile.c`, `nxt_macosx_sendfile.c`, `nxt_solaris_sendfilev.c`, `nxt_hpux_sendfile.c`, `nxt_aix_send_file.c` | 239, 143, 153, 169, 137, 126 (967) | `old_sendbuf` only (dead, section 3) | per probe (`auto/sources:316-351@872bf041`) |
| `src/nxt_clone.c` + `.h` | 434 + 54 | user-namespace uid/gid maps | `NXT_HAVE_LINUX_NS` (`auto/sources:354-356@872bf041`) |
| `src/nxt_cgroup.c` + `.h` | 318 + 16 | cgroup v2 placement | `NXT_HAVE_CGROUP` (`auto/sources:359-361@872bf041`) |
| `src/nxt_fs_mount.c` + `.h` | 257 + 48 | mounts for rootfs | `NXT_HAVE_ROOTFS` (`auto/sources:243-246@872bf041`) |
| `src/nxt_isolation.c` + `.h` | 1,557 + 24 | isolation config, rootfs, pivot_root | always (`auto/sources:18@872bf041`); 1,366 lines inside OS conditionals |
| `src/nxt_capability.c` + `.h` | 784 + 28 | Linux capabilities | always (`auto/sources:73@872bf041`); 727 lines inside OS conditionals |
| `src/nxt_credential.c` + `.h` | 353 + 30 | user and group lookup and switching | always (`auto/sources:17@872bf041`) |
| `src/nxt_process_title.c` | 256 | process titles | always (`auto/sources:20@872bf041`) |
| `src/nxt_unix.h` | 291 | per-OS feature macros (`src/nxt_unix.h:12-139@872bf041`) and 40-odd POSIX headers (`:142-196`) | always |

There is no `src/nxt_linux_*` file other than `nxt_linux_sendfile.c`, no OpenBSD or NetBSD file, and no `src/nxt_dyld.c`.

Size of OS conditionals. A script walks `#if` nesting and counts the lines inside blocks whose condition names an OS macro (`NXT_LINUX`, `NXT_FREEBSD`, `NXT_MACOSX`, `NXT_SOLARIS`, `NXT_HPUX`, `NXT_AIX`, `NXT_OPENBSD`, `NXT_NETBSD`) or an OS-related `NXT_HAVE_*` macro. Library and compiler features are excluded (NJS, OTEL, REGEX, OpenSSL, compression libraries, ASGI, PCRE2, wasm, PHP, GCC attributes and builtins, endianness, thread storage class). Both branches of such a block count. Results at 872bf041:
- Core: 713 OS directive lines (`#if`, `#elif`, `#else` and `#endif` all counted), 7,357 enclosed lines, out of 105,921 lines. libunit: 26 and 113 out of 9,468. Modules: 2 and 31. C tests: 104 and 3,499 out of 35,530. Counting only `#if` and `#elif` conditions roughly halves the directive numbers; the enclosed-line totals do not change.
- Top files by enclosed lines: `nxt_isolation.c` 1,366; `nxt_capability.c` 727; `nxt_epoll_engine.c` 490; `nxt_conf_validation.c` 449; `nxt_clone.c` 421; `nxt_process.c` 353; `nxt_time.c` 318; `nxt_fs_mount.c` 239; `nxt_lib.c` 234; `nxt_semaphore.c` 230; `nxt_file.c` 228; `nxt_http_static.c` 215; `nxt_credential.c` 204; `nxt_main_process.c` 187; `nxt_sockaddr.c` 174; `nxt_thread_id.h` 146; `nxt_controller.c` 139; `nxt_listen_socket.c` 122; `nxt_event_engine.h` 118; `nxt_unit.c` 113; `nxt_unix.h` 107.
- Without double counting: the 25 OS-specific files of Table 5.1 hold 9,809 lines. They are all rows except the portable poll and select fallbacks, and they include `nxt_credential`, `nxt_process_title` and `nxt_unix.h`, which are always compiled but OS-specific in content. The other 51 core and libunit files hold 507 OS directive lines with 3,783 enclosed lines.

Table 5.2. Platform-layer Unix APIs.

| Unix API or mechanism | Where (call sites at 872bf041) | Windows counterpart | Grade | Evidence |
|---|---|---|---|---|
| `dlopen`, `dlsym`, `dlclose`, `glob` | discovery `dlopen(RTLD_GLOBAL \| RTLD_NOW)` `src/nxt_application.c:413@872bf041`, `dlsym` `:420`, `dlclose` `:540`, module scan `glob()` `:268`; prototype `dlopen(RTLD_GLOBAL \| RTLD_LAZY)` `:1519`, `dlsym("nxt_app_module")` `:1528` | `LoadLibraryEx`, `GetProcAddress` (works for exported data such as `nxt_app_module`), `FreeLibrary`, `FindFirstFile` | wrapper | |
| module-to-executable symbol resolution | `unitd` links with `-Wl,-E` on Linux, FreeBSD, NetBSD, OpenBSD, DragonFly and the default branch (`auto/os/conf:31@872bf041`, `:54`, `:139`, `:161`, `:183`, `:266`), applied by the `unitd` link rule (`auto/make:480-485@872bf041`), but not on Solaris (`auto/os/conf:95@872bf041`); macOS modules use `-undefined dynamic_lookup` (`auto/os/conf:113@872bf041`); `-fvisibility=hidden` (`auto/cc/test:66@872bf041`, `:110`) with `NXT_EXPORT` (`src/nxt_clang.h:123-127@872bf041`; 318 declarations in non-test headers) | `unitd.exe` exporting `NXT_EXPORT` symbols with `__declspec(dllexport)` plus an import library linked into each module, or each module links the needed objects statically | redesign (small) | Modules call these core functions: PHP `nxt_conf_get_object_member`, `nxt_conf_get_string`, `nxt_conf_next_object_member`, `nxt_conf_object_members_count`, `nxt_malloc`, `nxt_zalloc`; Python 10 (`nxt_conf_*`, `nxt_int_parse`, `nxt_off_t_parse`, `nxt_malloc`); Perl `nxt_int_parse`; Ruby `nxt_int_parse`, `nxt_zalloc`; Java 4; wasm 5. All of them also call `nxt_unit_default_init()`, which lives in `unitd` (`src/nxt_application.c:1764@872bf041`). The module's `nxt_unit.o` also needs `nxt_lvlhsh`, `nxt_murmur_hash`, `nxt_socket_msg` and `nxt_websocket` from `unitd`. Logging macros call through a function pointer (`src/nxt_log.h:48-64@872bf041`, `:98-105`), so they add no symbol. `nxt_assert()` would need the `nxt_thread_context` data symbol (`src/nxt_main.h:123-126@872bf041`); libunit avoids it on purpose (`src/nxt_unit.c:3984-3987@872bf041`). |
| errno abstraction | `nxt_err_t` is `int` (`src/nxt_errno.h:11@872bf041`), `NXT_E*` constants (`:14-51`), including nginx's Windows-flavoured `NXT_ENOPATH` (`:16`) and `NXT_ENOMOREFILES` (`:49`); 247 uses of `nxt_errno` and 26 of `nxt_socket_errno` in non-test code; strerror table built at start-up by `nxt_strerror_start()` (`src/nxt_errno.c:36-129@872bf041`) | `GetLastError` and `WSAGetLastError` mapped to `NXT_E*`, `FormatMessage` | wrapper | The socket and system error split already exists. |
| random seed | `getrandom` `src/nxt_random.c:63@872bf041`, raw syscall `:69`, `getentropy` `:75`, `/dev/urandom` `:86-89` | `BCryptGenRandom` | wrapper | |
| syslog | `src/nxt_file.h:256@872bf041` | Event Log (`ReportEvent`) or the log file | wrapper | |
| files and state | `open` core 12; `pread` `src/nxt_file.c:137@872bf041`; `pwrite` `:109`; `rename` `:448`; `fsync` `:510`, `:542`; `F_FULLFSYNC` and `F_BARRIERFSYNC` (`:484-540`); directory sync `:578-594`; `unlink` core 7 | `CreateFile`, `ReadFile`/`WriteFile` with an `OVERLAPPED` offset, `MoveFileEx(MOVEFILE_REPLACE_EXISTING \| MOVEFILE_WRITE_THROUGH)`, `FlushFileBuffers` | wrapper | |
| open-then-unlink temporary files | `mkstemp` `src/nxt_http_request.c:737@872bf041` then `unlink` `:746`; `src/nxt_http_compression.c:397-402@872bf041` | `FILE_FLAG_DELETE_ON_CLOSE`, or POSIX delete semantics on NTFS | wrapper | The request-body file is then passed to a worker (Table 2.2). |
| `fcntl`, `ioctl(FIONBIO)`, `O_CLOEXEC`, `FD_CLOEXEC` | `fcntl` core 19; `FD_CLOEXEC` `src/nxt_socketpair.c:37@872bf041`, `:45` | `ioctlsocket`; handles are not inheritable by default, and `SetHandleInformation` sets `HANDLE_FLAG_INHERIT` | wrapper | |
| `posix_memalign`, `malloc_usable_size` | `posix_memalign` `src/nxt_malloc.c:134@872bf041` and `src/nxt_unit.c:8635@872bf041`; `malloc_usable_size` `src/nxt_malloc.h:57@872bf041`, `:82` | `_aligned_malloc` (must be freed with `_aligned_free`), `_msize` | wrapper | A mismatched free is a trap. |
| `off_t` | `nxt_off_t` is `off_t` (`src/nxt_types.h:42@872bf041`); `nxt_int_t` is `intptr_t` (`:29-30`) | `off_t` is 32-bit under MSVC; use `int64_t` | wrapper | `git grep -c -w long` over core sources (tests and modules excluded) gives 70 lines in 31 files for an LLP64 review. |
| control socket peer check | AF_UNIX only; TCP control connections are accepted without a check (`src/nxt_controller.c:902-906@872bf041`); `SO_PEERCRED` `:915`; supplementary groups with `SO_PEERGROUPS` `:847-887`, `:926-941`; `getpeereid` on BSD and macOS `:953-974`; socket ownership set with `nxt_file_chown()` `src/nxt_listen_socket.c:140@872bf041` | a named pipe with a security descriptor, then `GetNamedPipeClientProcessId` and the client token; or AF_UNIX plus `SIO_AF_UNIX_GETPEERPID` and `OpenProcessToken` | redesign (small) | |
| configure OS detection | `uname -s` `auto/os/test:6@872bf041`; a `MINGW*` branch sets `CC=cl` and `NXT_WINDOWS=YES` (`:77-88`), but nothing else reads `NXT_WINDOWS`; a MinGW `echo.exe` helper (`auto/echo/Makefile:3@872bf041`) | a real Windows branch | wrapper | Vestigial from nginx. |

Signs of nginx's Windows heritage in comments: `src/nxt_socket.h:38-40@872bf041` ("AF_UNIX is defined even on Windows although struct sockaddr_un is not"), `src/nxt_buf.h:21@872bf041` (buffer sizes "On Windows").

Reference sizes for a Windows layer, measured from shallow clones:
- nginx `src/os/win32`: 39 files, 6,271 lines; Windows event modules (IOCP, Windows select and poll): 1,245 lines; `src/os/unix` for contrast: 70 files, 12,347 lines (at [nginx 2b5c2b6](https://github.com/nginx/nginx/tree/2b5c2b605b5df669da5dec6749dcc76c07d1315d/src/os/win32), 2026-09-30).
- libuv `src/win`: 32 files, 27,099 lines, of which `pipe.c` 2,940 (named-pipe IPC with handle passing), `tcp.c` 1,748, `process.c` 1,488, `poll.c` 545, `signal.c` 282 (at [libuv 954c1d1](https://github.com/libuv/libuv/tree/954c1d1887bfdb223b2b0dc2201199f938902e68/src/win), 2026-10-05).

### Inferences
- The `NXT_EXPORT` macro already marks the exported surface. Defining it as `__declspec(dllexport)` when building `unitd.exe`, and generating an import library, is the least intrusive way to keep run-time module loading.
- Most of the platform layer is wrapper work of the kind nginx did in about 6,300 lines. The expensive parts are not here but in sections 1, 2 and 4.

### Gaps
- The full list of data symbols (not functions) that modules take from `unitd` was not extracted. Settle it on a Linux build: `nm -D --undefined-only build/lib/unit/modules/php.unit.so | grep ' nxt_'`.
- Whether MinGW-w64 GCC accepts every `__attribute__` and builtin used in the tree (11 `__builtin_*` lines in `src`, 24 with `auto`, 10 distinct builtins) was not tested; MSVC `cl` would need a replacement for each.

## 6. Build system and test harness

### Takeaway
`configure` is POSIX `sh` driving 148 feature probes, 50 of which run the compiled test program. Compile commands assume a GCC-style driver. A Windows build is practical with MSYS2's shell and a native MinGW-w64 GCC or clang toolchain, where run-probes still work; it is not practical with `cl.exe`. The pytest harness is Unix-only at its core (Unix control socket, process groups, `/proc`, `ps`, signals), but about half of the test functions use only HTTP and the control API.

### Cited Findings

Table 6.1. Build and test dependencies.

| Unix assumption | Where | Windows counterpart | Grade | Evidence |
|---|---|---|---|---|
| POSIX shell configure with `uname`, `sed`, `awk` | `configure`, `auto/*`; OS from `uname -s` (`auto/os/test:6@872bf041`) | MSYS2 `sh` | wrapper | |
| feature probes | 148 `nxt_feature=` probes; 50 set `nxt_feature_run=yes` or `value` (atomic 1, clang 2, endian 1, events 2, malloc 7, mmap 3, Perl 1, PHP 1, Python 1, Ruby 2, shmem 2, sockets 6, ssltls 2, threads 4, time 7, types 7, unix 1); compile command `$CC ... -o $NXT_AUTOTEST` (`auto/feature:38-39@872bf041`); run `:49-57`, `:74` | a native build runs them; cross-compiling cannot | wrapper | The SEQPACKET choice itself is a run-probe (`auto/sockets:127-142@872bf041`). |
| GCC-style flags | `-fvisibility=hidden` (`auto/cc/test:66@872bf041`, `:110`); `-Wl,-E` and `-shared` (`auto/os/conf:28-31@872bf041`) | MinGW-w64 and clang take the same flags; `-Wl,-E` becomes an export list | wrapper | |
| Makefile rules | generated by `auto/make`; install rules use `install -d`, `install -p` and `rm -f` (`auto/modules/php:652-660@872bf041`) | MSYS2 provides these | none with MSYS2 | |
| C test suite | `src/test` (35,530 lines; 3,499 inside OS conditionals); tests fork 19 times and call `waitpid` 18 times | per-test processes with `CreateProcess`, or threads | redesign | |
| pytest start-up | `conftest.py`: `import fcntl` (`test/conftest.py:2@872bf041`, use `:121`); unitd started with `--no-daemon` and `--control unix:<tmp>/control.unit.sock` (`:511-531`); new session and `os.killpg` (`:336`, `:534`, `:552`) | a Windows launcher; a job object for clean-up | redesign | |
| process inspection | `/proc` scan (`test/conftest.py:364-380@872bf041`); process lookup by title (`:591`); per-process fd counts (`:221-231`, `:590-592`); `ps ax -o state -o ppid` (`:626-627`); `/proc/self/ns` (`test/unit/utils.py:117@872bf041`); `/proc/sys/kernel/unprivileged_userns_clone` (`test/unit/check/isolation.py:148@872bf041`) | Toolhelp32 or psutil, and the `/status` API | redesign | |
| signals to unitd | SIGQUIT for graceful stop (`test/conftest.py:643-649@872bf041`); SIGTERM handler (`:465-472`) | `GenerateConsoleCtrlEvent` (Ctrl+Break) or a control-API stop | wrapper | |
| control client | AF_UNIX stream client (`test/unit/http.py:64-68@872bf041`) | Python's AF_UNIX on Windows: the CPython change to expose it ([bpo-33408 PR 14823](https://github.com/python/cpython/pull/14823)) was closed on 2020-01-15 and the issue was reported still open; not re-verified; a TCP control socket avoids the question | wrapper | |
| tools | `tools/unitc` is bash with `curl --unix-socket` (`tools/unitc:1@872bf041`, `:141`, `:148`) and `ps ax` (`:157`); `tools/setup-unit` is bash; `unitctl` uses `hyperlocal` (`tools/unitctl/unit-client-rs/Cargo.toml:27@872bf041`) and notes that sockets on Windows are not supported (`tools/unitctl/unit-client-rs/src/control_socket_address.rs:316-321@872bf041`) | PowerShell or a TCP control address; a named-pipe connector for `unitctl` | wrapper | |

Test portability count: 110 test files with 1,337 `def test_` functions at the top of `test/`. 42 files with 641 functions mention at least one Unix-only feature. The method is one regex per file for: `/proc/`, process lookup by name, `ps` or `pgrep`, `os.kill` and `signal.SIG*`, uid and gid calls, `pwd` and `grp`, isolation, rootfs, namespace, cgroup, chroot, `user` or `group` config keys, `unix:` addresses, abstract sockets, `AF_UNIX`, `.sock`, `resource.`, `os.fork` and capabilities. The other 68 files with 696 functions use only HTTP and the control API by this measure, but all of them depend on `conftest.py`. The split depends on the regex: a narrower or wider pattern moves it by several files.

### Inferences
- Keep `configure` and `auto/` under MSYS2 with a native MinGW-w64 GCC or clang (UCRT) toolchain for the core. Module builds must also match the language runtime's compiler and C runtime (section 7); clang can target the MSVC ABI where a runtime needs it.
- Port `conftest.py` before anything else in the test suite: a Windows launcher, a TCP or named-pipe control address, and process lookup through the API. That unlocks the 68 HTTP-only files.

### Gaps
- How many of the 148 probes fail under MSYS2 with clang or MinGW-w64 is untested. Settle it by running `./configure` there and reading `build/autoconf.err`.
- The 641-function figure is an upper bound from a regex. Each of the 42 files needs a look.

## 7. Language modules: direct Unix use and Windows status

### Takeaway
The modules' own Unix use is small: threads (Python, Perl, Java), `signal()` and `setlocale()` in Ruby, `fcntl`/`ioctl` on port descriptors (Node, ASGI), `realpath`/`stat` (PHP, Java). The hard parts are the process model and the event-loop hand-off, covered above. PHP matters most. FreeUnit implements its own SAPI on top of PHP's core API, so on Windows it needs PHP's core library from a build that matches its compiler and C runtime, and it must start PHP once per worker.

### Cited Findings

PHP (`src/nxt_php_sapi.c`, 1,835 lines):
- Own SAPI: `nxt_php_sapi_module` (`src/nxt_php_sapi.c:300@872bf041`). Start-up calls `php_module_startup()` (`:1400-1405`). Each request calls `php_request_startup()` (`:1344`), `php_execute_script()` (`:1363`) and `php_request_shutdown()` (`:1377`). It does not call `php_embed_init()`.
- Configure's "PHP embed SAPI" probe only checks that `php_module_startup()` links (`auto/modules/php:142-167@872bf041`). On Unix the library comes from `php-config` (`:91-115`).
- Process roles: setup in the prototype (section 1); `start` in the worker builds targets, calls `nxt_unit_default_init()` (`src/nxt_php_sapi.c:522@872bf041`), `nxt_unit_init()` (`:530`) and `nxt_unit_run()` (`:537`), then `exit(0)` (`:543`). One context per worker; the app config has no `threads` option for PHP (`src/nxt_main_process.c:325-337@872bf041`).
- Thread safety is optional: TSRM code runs only `#ifdef ZTS` (`src/nxt_php_sapi.c:407-416@872bf041`); the ZTS probe is informational (`auto/modules/php:170-186@872bf041`).
- Direct calls: `realpath()` (`src/nxt_php_sapi.c:1042@872bf041`), `stat()` (`:1204`), `fopen(filename, "re")` (`:1290`), whose `e` is glibc's close-on-exec flag (the Microsoft CRT spells non-inheritable `N`; not verified here). PHP's `chdir()` function is wrapped to track per-request directory changes (`:203-227`).
- Windows builds of PHP: 8.4 uses Visual Studio 2022 (VS17); earlier 8.x minors used VS16. Builds come as thread-safe and non-thread-safe; NTS is for single-threaded process models such as FastCGI. VS17 builds need the Visual C++ 2015-2022 redistributable ([windows.php.net](https://windows.php.net/)).
- Embed on Windows: today `--enable-embed=static` is ignored on Windows and produces a thin import wrapper that needs `php<N>.dll` at run time. php-src PR 22166 (open, created 2026-05-28) adds a self-contained static `php<N>embed.lib` to the development package, for TS and NTS ([php/php-src#22166](https://github.com/php/php-src/pull/22166)).
- Opcache on Windows: "All PHP processes have to map shared memory into the same address space"; `opcache.mmap_base` fixes "Unable to reattach to base address" errors; `opcache.file_cache_fallback` makes a process that failed to reattach use the file cache ([opcache configuration](https://www.php.net/manual/en/opcache.configuration.php)).

Other modules (direct calls beyond libunit):
- Python: start hook only (`src/python/nxt_python.c:49-57@872bf041`); `Py_InitializeFromConfig` in `nxt_python3_init_config()` (`:77-115`), called from `start` (`:215`); worker threads with `pthread_create` (`:682`); ASGI `ioctl(FIONBIO)` and `add_reader` on port descriptors (`src/python/nxt_python_asgi.c:1026@872bf041`, `:1076`).
- Perl: `PERL_SYS_INIT3` (`src/perl/nxt_perl_psgi.c:1151@872bf041`), `perl_alloc` (`:491`), worker threads (`:1286`).
- Ruby: `RUBY_INIT_STACK` and `ruby_init()` (`src/ruby/nxt_ruby.c:280-281@872bf041`), `signal()` (`:271`), `setlocale()` (`:278`), `pread` (`:1152`), `fstat` (`:1193`).
- Java: setup hook (`src/nxt_java.c:78@872bf041`), `JNI_CreateJavaVM` in `start` (`:339`), worker threads (`:626`), `realpath()` (`:122`, `:301`).
- Node.js: N-API addon; `uv_poll` and `fcntl` on port descriptors (`src/nodejs/unit-http/unit.cpp:570-593@872bf041`).
- Go: cgo glue (`go/nxt_cgo_lib.c:27-33@872bf041`); port descriptors wrapped as `*net.UnixConn` (`go/port.go:81-136@872bf041`).
- wasm: setup in the prototype (`src/wasm/nxt_wasm.c:389-444@872bf041`) with the wasmtime C API (`src/wasm/nxt_rt_wasmtime.c@872bf041`). MSYS2 packages the wasmtime C library for MinGW clang x86-64 ([mingw-w64-clang-x86_64-libwasmtime](https://packages.msys2.org/packages/mingw-w64-clang-x86_64-libwasmtime)), and wasmtime publishes x86_64 Windows release archives ([wasmtime install](https://docs.wasmtime.dev/cli-install.html)).

### Inferences
- The PHP module does not need an embed library on Windows. It should be able to link against the core import library that every PHP SAPI executable on Windows uses (`php8ts.lib` or `php8.lib` from the development package), provided the module uses the same compiler generation and the same dynamic UCRT. This matters because `zend_stream_init_fp()` receives a CRT `FILE *` that the module opened (`fopen` `src/nxt_php_sapi.c:1290@872bf041`; call `:1361`, macro `:1271`). Not verified; see Gaps.
- NTS fits FreeUnit's one-context PHP workers. Each Windows worker would run `nxt_php_setup()` itself, then `nxt_php_start()`.
- Python, Java, wasm and Node embedding are mainstream on Windows. The work there is the event-loop hand-off (section 4), not the runtime.

### Gaps
- Not verified: that the official PHP development package's `php8ts.lib` and `php8.lib` export `php_module_startup`, `sapi_startup`, `php_request_startup` and `php_execute_script`. Settle it with `dumpbin /exports php8ts.dll` and a 50-line SAPI built against the development package.
- Not verified in this pass: Ruby and Perl embedding on Windows (RubyInstaller and `ruby_init()` in a host process; Perl's `PERL_SYS_INIT3` and its thread-based fork emulation), Java JNI (`jvm.dll`), Node-API's `node.lib` delay-load hook, and the name of wasmtime's MSVC C-API release archive.

## 8. Synthesis: architectural assumptions, hardest problems, size and open questions

### Takeaway
The portability problem is architectural, not API-level. Fork inheritance, descriptor passing between unrelated processes, message-boundary sockets with kernel PID credentials, a socket shared by all workers of an app, and integer descriptors exposed through libunit's public API all have to change. The platform wrappers (time, files, threads, errno, dynamic loading) are routine and about nginx-sized. A credible first Windows target is a router, controller and PHP/Python workers on Windows Server 2022 or later, without isolation features: about 10,000 to 18,500 new lines plus 4,000 to 8,000 changed lines.

### Cited Findings

Architectural assumptions (not API-level), with evidence:
1. **Fork inheritance is the state channel.** A child keeps the parent's process table, ports and memory (`src/nxt_process.c:326-407@872bf041`). The worker's configuration is a pointer into the prototype (`src/nxt_application.c:829@872bf041`). The RPC stream counter is an anonymous shared mapping inherited by every process (`src/nxt_port_rpc.c:96-104@872bf041`).
2. **The prototype is a pre-initialised fork server.** Module load, the module setup hook (PHP start-up, wasm runtime), rootfs and `chdir` happen once per app before workers are forked (`src/nxt_application.c:569-645@872bf041`).
3. **Descriptors move between unrelated processes.** 18 send sites in 15 message kinds pass one or two descriptors. Pairs that are not parent and child: controller to router (DATA), prototype to router (NEW_PORT), router to worker (REQ_BODY, MMAP), worker to router (MMAP, NEW_PORT) and worker to main (WHOAMI) (Table 2.2).
4. **Ports are message sockets.** `SOCK_SEQPACKET` or `SOCK_DGRAM` (`src/nxt_socketpair.c:16-20@872bf041`); one receive is one message; READ_SOCKET markers keep queue and socket order (`src/nxt_port_socket.c:402-418@872bf041`).
5. **Authorisation is a per-message kernel PID** (PR 91; 28 uses in 6 files; `src/nxt_socket_msg.h:331-336@872bf041`), with a self-declared fallback on macOS and some BSDs (`src/nxt_port.h:327-332@872bf041`).
6. **Descriptor numbers in-band** appear only in `NXT_UNIT_INIT` for external apps (`src/nxt_external.c:121-134@872bf041`) and in `log_fd = 2` (`src/nxt_application.c:1809@872bf041`); everything else goes out-of-band.
7. **One socket read end is shared by all workers of an app** and polled by each of them (`src/nxt_unit.c:5976-5991@872bf041`). A wake-up is sent only when the router flips the app queue's `notified` flag from 0 to 1, and the worker that reads it clears the flag (`src/nxt_app_queue.h:84-97@872bf041`, `src/nxt_unit.c:7821-7822@872bf041`).
8. **Descriptors are `int` everywhere** (`src/nxt_file.h:11@872bf041`, `src/nxt_socket.h:11@872bf041`), including libunit's public API (`src/nxt_unit.h:81-82@872bf041`, `:179-181`), and three integrations put them into foreign event loops (Table 4.2).
9. **Signals are the external control plane** of main (`src/nxt_main_process.c:151-157@872bf041`), and the router and controller also stop on SIGINT, SIGTERM and SIGQUIT (`src/nxt_signal_handlers.c:19-27@872bf041`). In normal operation children are controlled by QUIT messages, and child death is learned from SIGCHLD in main and in prototypes (`src/nxt_main_process.c:1487@872bf041`, `src/nxt_application.c:1153@872bf041`).
10. **I/O is readiness-based**, with one listening socket shared by all router threads (`src/nxt_epoll_engine.c:616-625@872bf041`).
11. **Identity is the pid.** Ports are hashed by pid and id; pids are in every header and in REMOVE_PID (`src/nxt_port.h:251-271@872bf041`, `src/nxt_port.c:1286@872bf041`).
12. **Modules resolve core symbols from the executable** at load time (`auto/os/conf:31@872bf041`, `auto/modules/php:604@872bf041`).
13. **Privilege drops with setuid.** Main runs as root, children switch users (`src/nxt_credential.c:224-337@872bf041`); Unix permissions guard the control socket (`src/nxt_listen_socket.c:140@872bf041`).

Corrections to the briefing, verified at 872bf041:
- The Go package lives in `go/` at the repository root, not `src/go`.
- There is no `src/nxt_dyld.c`. `dlopen` is in `src/nxt_application.c:413@872bf041` and `:1519`.
- `auto/os/` holds only `conf` and `test`.
- Processes are created with `fork()` plus `unshare()` and a second fork for PID namespaces (`src/nxt_process.c:574@872bf041`, `:588`), not with `clone()`. `src/nxt_clone.c` only writes uid and gid maps.
- The port socket type is `SOCK_SEQPACKET` wherever the probe passes, Linux included. The comment "SOCK_SEQPACKET is disabled" (`src/nxt_socketpair.c:15@872bf041`, `src/nxt_unit.c:6750@872bf041`) is stale, because a passing probe defines the macro to 1 and `#if (0 || NXT_HAVE_AF_UNIX_SOCK_SEQPACKET)` then selects SEQPACKET. A third comment repeats the wrong reading (`src/test/nxt_unit_port_recv_test.c:1213-1216@872bf041`).
- `SO_REUSEPORT` is not used; router threads share one listening socket.
- The `nxt_*_sendfile.c` files are dead code; the live path is `nxt_sendfile()` in `src/nxt_conn_write.c:270-319@872bf041`.
- No request path uses the thread pool; static files are read on the engine thread into memory buffers, and `sendfile()` serves only spooled proxy request bodies.

### Inferences

The five hardest problems, ranked:
1. **Replacing the fork-based process model** (assumptions 1, 2, 6). Every child's start-up changes: role selection by command line, configuration by message, ports by brokered handles, and runtime start-up per worker. The worker-start latency the prototype hides today becomes visible for PHP and wasm, the two modules with a setup hook. The `nxt_process_init_t` life cycle (`prefork`, `setup`, `start`) and the WHOAMI and keep-matrix logic need a non-fork equivalent. A Windows prototype that spawns its workers with `CreateProcess` keeps the router protocol and the per-app bookkeeping; dropping prototypes instead moves START_PROCESS routing, the keep matrix, WHOAMI and the per-app child lists into main. It ranks first because every later piece depends on it, and because it removes the fast worker spawn that makes app restarts and scaling cheap.
2. **A port transport with handle passing and sender authentication** (assumptions 3, 4, 5, 11). The 2,951-line `nxt_port_socket.c`, fragment reassembly, the queue and socket ordering rules, 18 descriptor send sites, 28 authorisation uses and the libunit receive path all sit on SEQPACKET plus SCM_RIGHTS plus SCM_CREDENTIALS. Windows offers message-mode pipes for boundaries, `DuplicateHandle` and `WSADuplicateSocket` for handles, and per-connection PIDs for identity. Per-connection identity needs one pipe instance per writer and port, and duplication must go from the more trusted process to the less trusted one, because a `PROCESS_DUP_HANDLE` handle grants full control of its target.
3. **libunit wake-ups and the foreign event loops** (assumptions 7, 8). The shared port has many readers, which no Windows pipe or AF_UNIX socket models cleanly. It needs a per-app wake-up object plus private ports. The app queue's `notified` flag maps directly onto an auto-reset event or a semaphore with a maximum count of one. The public API change breaks the external-app ABI. Node (sockets-only `uv_poll`), Python ASGI (no `add_reader` on the default loop) and Go (its own datagram transport) each need rework.
4. **The event engine and connection I/O** (assumption 10). `ProcessSocketNotifications` keeps the readiness contract but needs build 20348 or later, allows one registration per socket (which complicates the shared listener) and covers sockets only, so pipes must ride the same completion port as overlapped operations. Older Windows means AFD polling (undocumented, but what libuv uses), zero-byte overlapped reads, or `WSAPoll` with its connect bug. `TransmitFile`'s client-edition limit touches only the proxy request-body path.
5. **The security model** (assumptions 9, 13). There is no setuid: per-app users need `CreateProcessAsUser` with logon or S4U tokens, or virtual accounts. Isolation (about 3,900 lines) has no direct counterpart. The control socket check must move to pipe ACLs or peer tokens. Static-file `chroot` and symlink rules need a reparse-point-aware path walk. Much of this is a product decision about what to drop in a first release.

Size estimate (an inference; basis stated):
- Today: about 13,600 lines of OS-specific code (9,809 in whole files, 3,783 inline), 11.8 percent of core and libunit.
- Files a Windows port must change, though not wholly: `nxt_process.c` (1,562), `nxt_main_process.c` (3,231), `nxt_application.c` (1,935), `nxt_port_socket.c` (2,951), `nxt_port.c` (1,572), `nxt_socketpair.c` (303), `nxt_socket_msg.c` and `.h` (405), `nxt_port_memory.c` (1,176), `nxt_unit.c` (8,713), `nxt_signal.c` and `nxt_signal_handlers.c` (259), `nxt_conn*.c` (3,051), `nxt_socket.c` (360), `nxt_listen_socket.c` (390), `nxt_sockaddr.c` (993), `nxt_file.c` (999): about 27,900 lines.
- New `nxt_win32` code, by part:
  - Base platform (types, errno, time, random, files, `dlopen`, threads, TLS, mapping, logging, socket addresses): 3,000 to 4,500 lines. Basis: nginx's `src/os/win32` is 6,271 lines and also covers services and console handling.
  - Engine (`ProcessSocketNotifications` plus overlapped pipes, post, timers, optional `WSAPoll` fallback): 1,200 to 2,500. Basis: the epoll engine is 1,225 lines; nginx's Windows event modules are 1,245.
  - Process spawn, job objects, supervision, console and service control: 1,500 to 3,000. Basis: libuv's `process.c` and `signal.c` total 1,770.
  - Port transport and handle broker (framing or message pipes, duplication, peer PIDs): 2,500 to 4,500. Basis: libuv's `pipe.c` is 2,940.
  - Shared-memory sections and wake-up objects: 400 to 1,000. A guess with no external basis.
  - libunit Windows back end (wait loop, handle API, per-thread ports): 1,500 to 3,000. A guess; the parts it replaces (`nxt_unit_shm_open()`, port creation and sending, `nxt_unit_read_buf()`, the send and receive paths) are a fraction of `nxt_unit.c`'s 8,713 lines.
- Total: about 10,000 to 18,500 new lines, plus 4,000 to 8,000 changed lines in shared code. The changed-lines figure is a guess, bounded above by the 27,900 lines of files listed. Excluded: the build system, the pytest harness, per-integration work in Node, Python ASGI and Go, and tests. The C test suite (35,530 lines, 19 `fork()` and 18 `waitpid()` calls) needs Windows equivalents for every test of changed code, which may add as much as the port itself.

Recommended direction for the plan:
- Target Windows Server 2022 or later and a 64-bit MinGW-w64 or clang toolchain under MSYS2 for the core.
- Main spawns the controller, the router and one prototype per app. Each prototype spawns its workers with `CreateProcess` and passes their handles by inheritance (an explicit handle list). Each worker runs its module's `setup`, then `start`. This keeps the router's START_PROCESS protocol and the prototype's bookkeeping. The alternative, main spawning workers directly, rewires more code for no gain found.
- Keep the port message format and the shared-memory layouts. Replace the transport with message-mode named pipes, one server instance per writer, driven by the same completion port as a `ProcessSocketNotifications` engine. Name the shared-memory sections; duplicate file and socket handles from the more trusted side only.
- Replace the shared-port socket with a per-app auto-reset event plus private ports.
- Defer isolation, per-app users and abstract or Unix-socket listeners.

### Gaps
Open questions, each with the code or experiment that would settle it:
1. **PHP core linkage.** Does a minimal SAPI built from `src/nxt_php_sapi.c`'s start-up and request path link against the official development package's `php8ts.lib` or `php8.lib` and run? Experiment: `dumpbin /exports php8ts.dll`, then build and run a 50-line SAPI with the matching VS17 toolset or clang-cl.
2. **Opcache across `CreateProcess` workers.** Do four workers started by `unitd.exe` share one opcache with ASLR on? Experiment: compare `opcache_get_status()` cache counts from each worker; watch for "Unable to reattach to base address".
3. **Worker start-up cost without fork.** Experiment: time process creation to first request for PHP with common extensions, Python and wasm on Windows, against fork on Linux.
4. **Transport choice.** Message-mode pipes versus AF_UNIX stream with framing. Experiment: a micro-benchmark of 16-byte and 16 KiB messages; also check whether `ProcessSocketNotifications` accepts AF_UNIX sockets.
5. **Wake-up correctness.** Can a per-app auto-reset event replace the shared socket without lost wake-ups under the `notified` rule? Read `src/nxt_app_queue.h:50-97@872bf041`, `src/nxt_router.c:8291-8299@872bf041` and `src/nxt_unit.c:7816-7827@872bf041`, then stress-test with N workers.
6. **Duplication across users.** Can the router or a prototype duplicate handles into a worker that runs under another account, and should it? Experiment: `OpenProcess(PROCESS_DUP_HANDLE)` and `DuplicateHandle` into a process started with `CreateProcessAsUser`; review against the full-control warning for `PROCESS_DUP_HANDLE`.
7. **Shared listener under IOCP.** One completion port drained by several router threads versus an accept thread with hand-off. Experiment on Windows Server 2022 with `listen_threads` greater than 1.
8. **LLP64 exposure.** Experiment: build the core on Linux with clang `-Wshorten-64-to-32 -Wconversion` and review the 70 `long` lines and `nxt_off_t` uses.
9. **Configure on Windows.** Run `./configure` under MSYS2 with MinGW-w64 GCC and clang; list failing probes from `build/autoconf.err`.
10. **Test harness reach.** Port `test/conftest.py` start-up and control access, then run the 68 HTTP-only files; first check `hasattr(socket, "AF_UNIX")` on the target CPython for Windows.
11. **Module symbol surface.** On Linux: `nm -D --undefined-only` on each `*.unit.so` to list every `nxt_` symbol (functions and data) a module takes from `unitd`.
12. **Go on Windows.** Can `go/port.go` move to C-side I/O (drop `port_send` and `port_recv`) so that Go needs no Windows socket semantics? Read `go/port.go:81-136@872bf041` and `go/nxt_cgo_lib.c:27-33@872bf041`.
13. **State store without fork.** Can `nxt_main_store_fork()` (`src/nxt_main_process.c:2557@872bf041`) become a thread on Windows? Read its job life cycle at `:2488-2740`.
14. **Runtime embedding facts not verified here:** Ruby and Perl hosts on Windows, the wasmtime MSVC C-API archive name, and whether `WSAPoll`'s connect bug persists on current builds.
