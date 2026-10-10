# FreeUnit for Windows: project plan

Status: proposal, 2026-10-07. Source revision: freeunitorg/freeunit `872bf041` (origin/master on 2026-10-07). Evidence: docs/research-report.md and the five notes in research/.

Code references are written `path:line@872bf041` and resolve at `https://github.com/freeunitorg/freeunit/blob/872bf04170756e919211799ec5574000d628d521/<path>#L<line>`. "Unverified" marks a claim that nobody has tested on Windows. "Verified" marks a claim rechecked on 2026-10-07 against the source or the cited primary page. Section references such as "report §2" point to the numbered sections of docs/research-report.md.

The port lands in https://github.com/freeunitorg/freeunit as pull requests. This repository holds the plan, the Phase 0 experiments and their results, the CI recipes and the feature matrix.

## 1. Recommendation

**Conditional go.** Run Phase 0 now: 23 experiments and readings plus a demand poll (E01 to E24; the poll is E22), from 2026-10-12 to a gate decision (G0) on 2026-12-18. Phase 0 needs no change to the FreeUnit repository and no paid CI, because hosted Windows runners are free for public repositories; it needs one Windows 11 virtual machine and its licence. Start milestone M1 only if G0 passes three conditions. (1) Technical: experiments E01, E02, E03, E04, E11, E12 and E15 meet their expected results, or each failure has a fallback that this plan already names. (2) People: a named owner for the Windows build commits to the Tier 3 period, which runs to 12 months after the preview, and a second person agrees to review process, port and shared-memory changes. (3) Demand: the poll E22 closes on 2026-12-04 with at least 40 respondents who develop PHP on Windows, of whom at least 10%, and at least 6 people, report that WSL2 or Docker is blocked at work. The FreeUnit maintainer (https://github.com/andypost) runs it as a GitHub Discussion in https://github.com/freeunitorg/freeunit/discussions and links it from DDEV and Drupal community channels. The condition is also met if, by the same date, at least 5 distinct people who maintain neither FreeUnit nor freeunit4drupal report in freeunit4drupal's public issue tracker that they cannot use WSL2 or Docker for it. freeunit4drupal is public at https://github.com/freeunitorg/freeunit4drupal (first commit 2026-10-07). The raw counts and the links go into decisions/g0.md, so anyone can recompute the result. If any condition fails, FreeUnit documents WSL2 as its Windows path, this repository stays as the record, and the question is reopened in 2027-06.

Three facts decide it.

1. **The work is a redesign of the process and IPC layer, not an API port.** `fork()` inheritance, message sockets with descriptor passing, per-message kernel pids and one socket read by competing workers all need Windows designs. The audit estimates 10,000 to 18,500 new lines plus 4,000 to 8,000 changed lines before tests, and calls the changed-lines figure a guess (research/portability-audit.md §8). nginx's whole Windows layer is about 6,200 lines, and nginx for Windows has been "beta" since 2009.
2. **Windows now has what a readiness design needs, without fork emulation.** ProcessSocketNotifications gives documented readiness through a completion port from build 20348 (verified). AF_UNIX stream sockets, named sections in private namespaces and job objects cover the rest. FreeUnit's shared memory uses offsets, not pointers. The official MSVC-built PHP links with clang-cl, as FrankenPHP showed in its 2026-03-06 release (verified).
3. **Demand is unmeasured where it matters, and the cost is known.** FreeUnit has no Windows request. Upstream had one native-build request in eight years, with zero reactions. 85% of Windows hosts in DDEV's telemetry already run WSL2. Mature projects spend about 3% of tickets or commits on Windows, and Windows ports have ended when their maintainers left (Microsoft's Redis port in 2016, Envoy in 2023).

## 2. Goals, non-goals, platform floor, support tier and success metrics

### 2.1 Goals

- G1. A native `unitd.exe` that serves static files and PHP applications, Drupal first, to a developer on Windows 11 x64. It starts from a terminal and needs no WSL2, no Docker and no administrator rights.
- G2. The same configuration format and control API as on Linux and macOS. Every difference is listed in the feature matrix (§2.4).
- G3. Parity with itself. The Windows build is measured across its own engines and against php-cgi based stacks for concurrency behaviour, never against Linux for speed.
- G4. No regression on Linux or macOS from shared-code changes.
- G5. Windows mechanisms stay behind the existing abstractions (process start, port transport, shared memory, event engine, files, time), so Unix code paths stay readable.

### 2.2 Non-goals

- Production support on Windows (Tier 1 or Tier 2) within this plan.
- Linux isolation on Windows: namespaces, rootfs, cgroups, capabilities and per-application users.
- Windows service mode, an MSI and a Chocolatey package before the preview.
- ARM64 and 32-bit builds. No official ARM64 PHP build exists.
- External applications (Go, Node.js), Python ASGI, Perl, Ruby, Java, njs and wasm before the preview.
- Daemon mode, process titles and Unix signals as an interface.
- Speed-ups over Linux, and completion I/O for connections without a measured gain.
- Cygwin, the MSYS runtime or any POSIX emulation layer in a shipped binary.

### 2.3 Platform floor

- Architecture: x64 only.
- Tested and supported: Windows 11 (versions in servicing), Windows Server 2022 and Windows Server 2025. These are builds 20348 and later, which have ProcessSocketNotifications.
- Best effort, untested: builds 17763 to 20347 (Windows 10 version 1809 and later, including 22H2 and the LTSC 2019 and 2021 editions, and Windows Server 2019). unitd selects the WSAPoll engine there and logs once that the platform is untested. Below build 19041, WSAPoll does not report failed connects, so a failed proxy connect is detected only at the connect timeout. Maintainer decision O3 may remove this range.
- Refused: builds below 17763. No serviced Windows release below that build has AF_UNIX, which the port transport needs. unitd exits at start with a message that names the build it found.
- The check is rule 13 of §7. Evidence and rejected alternatives are in D2.

### 2.4 Support tier: Tier 3 "development use"

Tier 3 follows the Rust, CPython and Node.js tier policies (research/toolchain-packaging-demand.md §7.1). CI builds the core and the PHP module and runs the C tests and a pytest subset on every pull request that touches `configure`, `auto/**`, `src/**` or `test/**`. Windows failures do not block FreeUnit releases. Hosted runners cover Windows Server 2022 and 2025 only, so Windows 11 x64 is tested by hand on the virtual machine with the first-demo checklist at every preview release and once a month from M5 to M8. At least one named owner exists. The feature matrix below is published with every release.

| Area | Feature | Status at the preview | Note |
|---|---|---|---|
| Platform | Windows 11, Server 2022, Server 2025 x64 | tested | build 20348 or later |
| Platform | Windows 10 1809 or later, LTSC 2019 and 2021, Server 2019 (builds 17763 to 20347) | best effort | WSAPoll engine; below build 19041 a failed proxy connect is detected at the connect timeout; decision O3 |
| Platform | builds below 17763, including Server 2016 | no | refused at start: no AF_UNIX |
| Platform | ARM64, x86 | no | |
| Process | foreground console process | yes | Ctrl+C fast stop; Ctrl+Break graceful stop from the same console |
| Process | `unitd --signal quit`, `stop`, `reopen` | yes | a new option on Windows; control pipe, D11 |
| Process | daemon mode | no | |
| Process | Windows service | later | after the preview |
| Engine | WSAPoll engine | yes | default below build 20348 and when the probe fails |
| Engine | ProcessSocketNotifications engine | after M7 | default on build 20348 or later once M7 passes |
| HTTP | TCP listeners, IPv4 and IPv6 | yes | documentation uses 127.0.0.1 |
| HTTP | Unix-domain and abstract listeners | no | |
| HTTP | TLS with OpenSSL 3.5 LTS | yes | |
| HTTP | proxy | yes | |
| HTTP | static files | yes | path rules, D19; links that leave the share are refused |
| HTTP | `chroot`, `follow_symlinks: false`, `traverse_mounts: false` | no | rejected by validation |
| HTTP | njs | no | njs has no Windows support |
| HTTP | OpenTelemetry | later | after E21 |
| Apps | PHP 8.4 and 8.5, non-thread-safe, x64 | yes | official runtime bundled, D13 |
| Apps | PHP thread-safe | no | built nightly, not shipped |
| Apps | `max_execution_time` | differs | wall-clock time on Windows and on Apple Silicon macOS; CPU time on Linux |
| Apps | `processes` (`max`, `spare`, `idle_timeout`), `limits` | yes | |
| Apps | `user` and `group` other than the current account, `isolation` | no | rejected by validation |
| Apps | Python WSGI, wasm, wasm-wasi-component | later | |
| Apps | Python ASGI, Node.js, Go, `type: external` | no | D20 |
| Apps | Perl, Ruby, Java | no | |
| Control | AF_UNIX control socket with a peer check | yes | default, D18 |
| Control | TCP control socket | yes | unauthenticated, as on Unix |
| Tools | unitctl | if E21 passes | TCP control only |
| Logs | error and access logs, reopen | yes | reopen by path; renames retried on sharing violations, D22 |
| Files | default layout | yes | per-user data under `%LOCALAPPDATA%\FreeUnit`, modules next to `unitd.exe`, D21 |
| Tools | `tools/unitc` | no | a bash script |

**Exit rule.** If the Windows CI job stays red for one full release cycle with no owner, or the named owner steps down and nobody replaces them within one release cycle, the next FreeUnit release ships without Windows artefacts and its release notes say so. The Windows code stays in the tree for one more release. If nobody revives it by then, a pull request removes it.

**Promotion rule.** Tier 2 needs two named maintainers, the ProcessSocketNotifications engine as the default, the HTTP-only pytest subset green, signed release artefacts and 12 months at Tier 3 without triggering the exit rule.

### 2.5 Success metrics

| ID | Metric | Target | Date |
|---|---|---|---|
| S1 | Phase 0 complete | every experiment in §5 has a results file with raw output, Windows edition, build number and compiler version; the G0 decision is written in decisions/ | 2026-12-18 |
| S2 | Windows CI | win-core builds unitd and runs the C test table on `windows-2025` for every relevant pull request; median leg at most 20 minutes; required check | 2027-03-31 |
| S3 | Linux and macOS unharmed | no red job caused by a Windows pull request; router CPU per request within the A/A noise of the io_uring harness (about ±1% to ±4%) for port-layer pull requests | continuous |
| S4 | First demo | the demo of §6.2 passes, including 1,000 requests over 5 URLs at concurrency 4 with zero 5xx responses | 2027-06-11 |
| S5 | Opcache | all workers of the demo application report the same opcache start time, and one hour with `request_limit` 50 logs zero reattach failures | 2027-06-11 |
| S6 | Shutdown | Ctrl+C ends every FreeUnit process within 5 seconds and leaves no orphan | 2027-06-11 |
| S7 | Tests | at least 40 of the 68 HTTP-only test files pass on Windows | 2027-09-24 |
| S8 | Engine parity | ProcessSocketNotifications router CPU per request within ±10% of WSAPoll at 1 to 10 connections; per-event cost stays flat from 100 to 10,000 idle connections, read as growth of at most 1.5 times, with wepoll measured as a reference | 2027-08-27 |
| S9 | Preview release | zip and SHA256SUMS on GitHub Releases; winget manifest accepted; Scoop bucket live; feature matrix published; a standard user runs the demo on a clean Windows 11 VM | 2027-09-24 |
| S10 | Maintenance share | Windows-labelled issues and pull requests at most 10% of all FreeUnit issues and pull requests per quarter | quarterly to 2028-09-30 |
| S11 | Adoption | at least 5 distinct people who maintain neither FreeUnit nor freeunit4drupal report using the Windows build, in FreeUnit issues or discussions or in freeunit4drupal's tracker | 2028-03-31 |

The dates assume one owner working about half time with agent help, one reviewer, and about half of FreeUnit's measured 2026 rate. Commits on master with committer dates from 2026-04-03 to 2026-10-07 (26.7 weeks) inserted about 1,200 lines per week of non-test C in `src/` and about 1,150 lines per week of C tests (`git log --no-merges --numstat` at 872bf041; gross insertions, rework included). September 2026 alone reached about 4,000 lines per week of non-test C. At half the 26.7-week rate, the 11,400 to 21,000 new C lines of §6.1 take 19 to 36 weeks, with their C tests written alongside at a similar rate, and the reviewer reads about 600 lines of C and 600 lines of tests per week. The window from M1 to M8 is 38 weeks, so the schedule fits the low end of the estimate and is tight at the high end. G0 re-plans the dates with the Phase 0 experience; the order of the milestones does not change.

## 3. Decisions

Each decision names the choice, the alternatives rejected, the evidence and the main risk. A Phase 0 result that contradicts a decision changes the decision in this file, in the same pull request as the result.

### D1. Compiler and build system

- **Choice.** clang-cl in MSVC mode, x64, `/MD` (dynamic Universal CRT), for `unitd.exe`, libunit and every MSVC-ABI module. Release artefacts are built on the `windows-2022` image with the Visual Studio 2022 17.14 Build Tools and their "C++ Clang tools for Windows" component, which match the v14.44 toolset of the official PHP 8.4 and 8.5 builds; one toolset serves the core and the modules. CI also builds the core on `windows-2025` with Visual Studio 2026 as an early warning for the next toolset, and release builds move to Visual Studio 2026 when PHP 8.6 ships with VS18. v14 binaries from these versions share one redistributable. Keep one build system: the existing `configure` and `auto/` scripts, run by the MSYS2 MSYS shell with GNU make. Add an MSVC-mode compiler case to `auto/cc/test`, a Windows case to `auto/os/conf`, a rewritten `MINGW*|MSYS*` branch in `auto/os/test` (the `auto/echo` helper goes), `.exe` handling for run-probes in `auto/feature` if E02 shows that MSYS2 needs it, a clang-cl branch in `auto/cc/hardening` (compiler `/guard:cf` and `/GS`; linker `/guard:cf`, `/DYNAMICBASE`, `/HIGHENTROPYVA`, `/NXCOMPAT` and `/CETCOMPAT`; each probed like the Unix flags), import-library linking for modules, and the Rust library name `otel.lib`. Take OpenSSL, PCRE2, zlib, brotli and zstd from a vcpkg manifest with a pinned baseline, triplet `x64-windows-static-md`, and OpenSSL pinned to the 3.5 LTS line. Pass compiler options in dash form. `cl.exe` stays a fallback; it would need shims for `nxt_inline` (`src/nxt_clang.h:11-12@872bf041`, verified), the `__sync` atomics (`src/nxt_atomic.h:17-89@872bf041`) and `__builtin_ffs`.
- **Rejected.** MinGW-w64 gcc: it cannot produce the `name@@N` references that 340 of the 6,116 exports of the official php8ts.lib use, and FrankenPHP's MinGW build failed on a C runtime mismatch. clang in GNU mode: it needs a patched `ZEND_FASTCALL`, which PHP does not support. A Windows-only CMake or Meson build: PostgreSQL and curl removed theirs. Release builds cross-compiled from Linux with xwin or msvc-wine: the Microsoft toolchain is not redistributable, and hosted Windows runners are free.
- **Evidence.** Report §3 and §5 (conflict a); research/toolchain-packaging-demand.md §1.0 to §1.7, §2.7, §2.8; research/portability-audit.md §6, whose MinGW suggestion this decision overrules.
- **Risk.** The 148 configure probes, 50 of them run-probes, are untested under clang-cl (E02). clang-cl's and lld-link's acceptance of these hardening flags is unverified (E02 and M1 probe them). FrankenPHP found `/GUARD:CF` costly for PHP itself (research/toolchain-packaging-demand.md §1.5); M7 measures its cost for unitd. MSYS2 becomes a prerequisite for Windows contributors.

### D2. Platform floor

- **Choice.** As §2.3: tested and supported from build 20348; best effort from build 17763 to 20347 with the WSAPoll engine, plus a connect timeout and an `SO_ERROR` check for proxy connects, which the I/O note's stage 1 prescribes for builds before Windows 10 version 2004; refuse below 17763. unitd reads the build with `RtlGetVersion` from ntdll.dll and probes `ProcessSocketNotifications` with `GetProcAddress` on ws2_32.dll before it creates any engine, as Microsoft's WIL does.
- **Rejected.** Build 17763 as the supported floor (research/process-and-ipc-design.md §2.A): it has no ProcessSocketNotifications, WSAPoll reports failed connects only "As of Windows 10 version 2004" (verified; E13 records the older behaviour), and GitHub retired its Server 2019 runner on 2025-06-30, so CI cannot test it. A hard 20348 runtime floor: simpler, but it shuts out Windows 10 machines under extended security updates (consumers to 2027-10-12, organisations to 2028-10-10), where developers without WSL2 may well work (inference); decision O3 can still choose it. Builds below 17763: no serviced release among them has AF_UNIX.
- **Evidence.** Report §2 and §5 (conflict b); research/windows-io-model.md §2, §5 and §6; research/process-and-ipc-design.md §2.A; research/toolchain-packaging-demand.md §3.1 and §4.7.
- **Risk.** The best-effort range has no CI. Drop it when Windows 10 updates end, or earlier if its reports cost more than one fix per quarter.

### D3. Process spawning and supervision

- **Choice.** `CreateProcessW` on `unitd.exe` itself, with `--role <main|discovery|controller|router|prototype|app>`, `--app <name>` where it applies, and `--startup <handle value>`. Use `STARTUPINFOEXW` with `PROC_THREAD_ATTRIBUTE_HANDLE_LIST` naming exactly the handles the child needs, `bInheritHandles` TRUE, and the flags `CREATE_SUSPENDED`, `CREATE_UNICODE_ENVIRONMENT` and `CREATE_NO_WINDOW`. `hStdOutput` and `hStdError` are the log handle. The parent writes the startup block (§4.5) and then calls `ResumeThread`. Main creates job J with `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE` and no breakaway flags, keeps the job handle non-inheritable and assigns each of its children; workers join J through their prototype. Every parent registers `RegisterWaitForSingleObject` with `WT_EXECUTEONLYONCE` on each child process handle; the callback posts `{pid, handle}` to the parent's engine, where today's SIGCHLD logic runs with `GetExitCodeProcess`. The state-store child becomes a thread in main. unitd always runs in the foreground and writes its pid file.
- **Rejected.** Inheriting every inheritable handle: libuv closed the resulting race only in 2026 ([libuv PR 5100](https://github.com/libuv/libuv/pull/5100)). An environment-only bootstrap (nginx): too small for application configurations. `DuplicateHandle` into the suspended child for every handle (PostgreSQL does it for single handles): kept only for handles that must never be inheritable. Fork emulation: Cygwin documents that "occasional fork failures are inevitable". Job completion-port messages for reaping: Microsoft says their delivery "is not guaranteed".
- **Evidence.** Report §1; research/process-and-ipc-design.md §2.1, §2.3, §3, D1 and D8; research/portability-audit.md §1.
- **Risk.** Process start cost (E12). Behaviour when unitd itself starts inside a job owned by an IDE, a terminal or a CI runner (E17).

### D4. Prototype and worker start

- **Choice.** Keep one prototype per application as a resident spawner and supervisor. The prototype loads the module, runs the module's `setup` (for PHP, `php_module_startup()`) and stays resident. That keeps `php8.dll` loaded at one base address and keeps the opcache mapping alive between worker restarts. It answers the router's START_PROCESS unchanged and spawns each worker with D3. Each worker loads the module, runs `setup` itself, then `start`. A worker receives its application configuration in the startup block instead of the pointer it inherits today (`src/nxt_application.c:829@872bf041`, verified). The documentation recommends `processes.spare` of at least 1 on Windows, so a warm worker is ready when the start cost matters.
- **Rejected.** One worker with many threads on a thread-safe PHP: a different concurrency model, while the module runs one request loop per process (`src/nxt_php_sapi.c:526-544@872bf041`, verified). Main spawning workers directly: it rewires the router's start path and the REMOVE_CHILD_PID flow for no gain.
- **Evidence.** Report §1; research/process-and-ipc-design.md §1.10, §1.11 and D2; research/portability-audit.md §1 and §8.
- **Risk.** PHP and wasm now pay their setup in every worker (E12). Opcache reattach across workers (E11).

### D5. Port transport and framing

- **Choice.** AF_UNIX `SOCK_STREAM` sockets, chosen for engine compatibility, not for measured speed (E24 measures it).
  - Each port is a listening socket bound to `<rundir>\p<pid>-<port id>` in a per-instance runtime directory. The directory's DACL admits only the unitd user and SYSTEM. Socket files take the directory's DACL, and Windows requires write permission on a socket file to connect to it ([AF_UNIX comes to Windows](https://devblogs.microsoft.com/commandline/af_unix-comes-to-windows/)), so the DACL also limits who can connect.
  - The parent binds the child's own port listener while the child is still suspended, because only then does it know the child's pid. It passes the listener with `WSADuplicateSocketW`, writes the protocol information into the startup block (§4.5, record PORT_LISTENER), and closes its own copy after the child's first message. Writers can connect at once; their connections wait in the backlog until the child accepts. This keeps today's rule that a parent can write to a child's port right after the spawn (`src/nxt_process.c:253-301@872bf041`, verified). E04 step 7 tests it.
  - Each writer process opens its own connection to a port on its first write and sends a HELLO frame with its pid and port id. The owner answers with a connection index.
  - A frame is a 4-byte little-endian length, the 16-byte `nxt_port_msg_t`, an attachment block (D8) and the payload. The reader checks the length against `max_size` plus the header and attachment limits before it reads, with `nxt_span_t` and `nxt_size_add()`.
  - The reader never reads past the current frame. It reads the length and header, then exactly the rest of the frame, so no complete message waits in user memory without a readiness event.
  - The writer keeps at most one partial frame per connection and returns NXT_AGAIN to the port layer while that tail is pending.
  - The framing lives in the Windows branch of `src/nxt_socket_msg.c`, which unitd and libunit both link. `nxt_port_socket.c` and `nxt_unit.c` keep the rule "one call, one message". Today a short write is a hard error there (`src/nxt_port_socket.c:1234-1238@872bf041`, verified), and it stays one.
  - READ_SOCKET markers in a port's shared-memory queue carry the writer's connection index. On a marker the reader consumes the next frame of that connection.
  - The runtime directory plus file name must fit `UNIX_PATH_MAX` (108 bytes). unitd checks this at start. Main holds `<rundir>\lock` open with share mode 0 for its lifetime and deletes stale socket files at start.
- **Fallbacks, by the result of E04.**
  - Step 2 passes (WSAPoll accepts AF_UNIX): stage 1 as planned. In stage 2, port sockets use ProcessSocketNotifications if step 3 passes, zero-byte overlapped reads on the same completion port if only step 6 passes, and otherwise WSAPoll stays the only engine for Tier 3.
  - Step 2 fails: M4 builds the stage 2 engine first, serving port sockets as above, and the best-effort range below build 20348 is dropped.
  - Steps 2, 3 and 6 all fail: ports switch to message-mode named pipes with one instance per writer (research/process-and-ipc-design.md D3), driven as overlapped I/O on the engine's completion port. The control socket of D18 then falls back to TCP loopback, documented as unauthenticated, and the stage 1 doorbell to a loopback TCP pair whose accepted peer address must equal the connecting socket's local address.
- **Rejected.** Named pipes as the default (research/process-and-ipc-design.md D3; research/portability-audit.md §8). The I/O note observes that a stage 2 completion port can carry pipe completions next to socket notifications (research/windows-io-model.md §4). That holds, but stage 1 cannot wait on pipes at all without a helper thread, and stage 2 would need a completion path in the port layer, which the io_uring work deliberately kept in poll mode. Selector-style integrations (libuv `uv_poll`, asyncio selectors) accept sockets only, but libuv does not use AF_UNIX on Windows ([libuv issue 2537](https://github.com/libuv/libuv/issues/2537)), so that argument is weak until someone tests it. Loopback TCP: no peer identity, reachable by every local user, firewall prompts. Anonymous pipes: no overlapped I/O. One connection shared by several writers: a stream gives no atomic writes across processes.
- **Evidence.** Report §2 and §5 (conflict c); research/process-and-ipc-design.md D3, the first half of whose own condition for AF_UNIX now holds; research/windows-io-model.md §4, §6 and §7; research/portability-audit.md §2 and §4; [io_uring design.md](https://github.com/andypost/unit/blob/6ee2a59dbdc3536609ec33e1fc32a14ed7e17f15/docs/io_uring/design.md) §2.3 and §2.4.
- **Risk.** ProcessSocketNotifications documents that "Only Microsoft Winsock provider sockets are supported" (verified); whether that includes AF_UNIX is unverified (E04). AFD poll appears to work with AF_UNIX in wepoll and mio, in issues that are still open and without a merged test. The peer-pid ioctl is undocumented (E05). The path length limit hits long user profile paths; the runtime directory is configurable.

### D6. Sender identity

- **Choice.** Identity per connection. At accept the owner reads the peer pid with `WSAIoctl(SIO_AF_UNIX_GETPEERPID)` and checks it against its registry of known FreeUnit processes: pid plus creation time from `GetProcessTimes`, filled from its own spawns, from the PEER records of its startup block and from NEW_PORT announcements, which carry the creation time (D8). `nxt_recv_msg_cmsg_pid()` returns that pid for every frame of the connection, so the checks added by [PR 91](https://github.com/freeunitorg/freeunit/pull/91), 28 uses by the audit's count, stay as they are (research/portability-audit.md §2). If E05 shows the ioctl missing or wrong on the floor builds, each child gets a 256-bit token in its startup block, sends it in HELLO, and the parent announces it to the processes that accept the child's connections in the existing NEW_PORT message. The trust model is stated in the documentation: all FreeUnit processes run as one user, so these checks stop other local users and bugs, not a compromised worker.
- **Rejected.** The self-declared header pid as the only identity (the macOS fallback at `src/nxt_port.h:327-332@872bf041`). Per-message signatures: cost without a threat they stop under one user account.
- **Evidence.** Report §1 and §2; research/process-and-ipc-design.md §1.8, §2.5 and D13.
- **Risk.** Which pid the ioctl reports after `WSADuplicateSocketW` is unknown (E05).

### D7. Shared memory

- **Choice.** Paging-file sections from `CreateFileMappingW`, named inside a per-instance private namespace. Main creates the namespace with `CreatePrivateNamespaceW`, a boundary descriptor that holds the unitd user SID, and `lpPrivateNamespaceAttributes` with a DACL for the unitd user and SYSTEM. The boundary alone does not restrict access: Microsoft states that "a process can open an existing namespace even if it is not within the boundary unless the creator restricted access to the namespace using the lpPrivateNamespaceAttributes parameter" ([Object Namespaces](https://learn.microsoft.com/en-us/windows/win32/sync/object-namespaces), verified). Every section and event in the namespace is created with the same explicit DACL. Main keeps the namespace handle open for the life of the instance. Names end in at least 128 random bits from `BCryptGenRandom`. `ERROR_ALREADY_EXISTS` on create is a hard failure. Receivers open by name and map the whole section at offset 0 with the exact expected size (`PORT_MMAP_SIZE` or the queue size). A view larger than its section fails, which replaces the `fstat()` size check (E18); the header checks of [PR 173](https://github.com/freeunitorg/freeunit/pull/173) stay. Certificate, script, configuration-store and configuration-push blobs get an acknowledgement before the creator closes its handle, because a section disappears with its last handle. The RPC stream counter, today an anonymous mapping inherited through `fork()`, becomes a named section opened by every process. Nothing maps at a fixed address, and a C test asserts that shared structures hold offsets, not pointers.
- **Rejected.** Unnamed sections moved by `DuplicateHandle`: siblings and workers would need `PROCESS_DUP_HANDLE` on each other. The `Global\` namespace: it needs `SeCreateGlobalPrivilege` outside session 0. Fixed-address mapping: nginx, PostgreSQL and opcache need it; FreeUnit's layouts do not (`src/nxt_unit_sptr.h:18-34@872bf041`, verified).
- **Evidence.** Report §1; research/process-and-ipc-design.md §1.6, §2.6, D5 and D6; research/portability-audit.md §2.
- **Risk.** Restricted or low-integrity workers in a later isolation project may not reach the namespace (research/process-and-ipc-design.md Q15).

### D8. Descriptor passing

- **Choice.** Port write ends no longer move: writers connect by path (D5). Sections move by name (D7). Files and sockets move by duplication, and only the more trusted process duplicates:
  - main into its children, with the `PROCESS_ALL_ACCESS` handles that `CreateProcessW` returned;
  - the router into a worker for REQ_BODY files; the router opens the worker once with `OpenProcess(PROCESS_DUP_HANDLE)` when the prototype's NEW_PORT announces it, and compares the creation time, which becomes a new NEW_PORT field;
  - listening sockets from main to the router with `WSADuplicateSocketW` for the router's pid, then `WSASocketW` with `FROM_PROTOCOL_INFO` in the router.

  Log reopen becomes reopen by path in each process. Attachments travel in the frame as `{kind, length, bytes}` records parsed with `nxt_span_t`. The kinds are a handle value already valid in the receiver, a section or event name, and a `WSAPROTOCOL_INFOW` blob for a socket (about 630 bytes). A frame carries at most two attachments and at most 2 KiB of attachment data. The ownership rule stays: a handler that keeps an attachment clears its slot, and the port layer closes the rest (`src/nxt_port.h:340-346@872bf041`). After a failed delivery the pusher closes the remote copy with `DUPLICATE_CLOSE_SOURCE`. Only a handle's final owner associates it with a completion port.
- **Rejected.** Duplication by workers: a `PROCESS_DUP_HANDLE` handle grants full control of its target. Passing temporary files by path.
- **Evidence.** Report §1; research/process-and-ipc-design.md §1.5, §2.2 and D5; research/portability-audit.md §1 and §2; research/windows-io-model.md §6.
- **Risk.** Pid reuse between announcement and open; the creation-time check covers it, and M3 tests it.

### D9. Wake-ups for competing workers

- **Choice.** One auto-reset event per application, created by the router as a named object in the private namespace. The router sets it exactly where it sends READ_QUEUE to the shared port today, when `nxt_app_queue_send()` flips `notified` from 0 to 1 (`src/nxt_router.c:8288-8299@872bf041`, verified). A worker waits with `WaitForMultipleObjects` on the application event and one port event; `WSAEventSelect` ties all of the worker's port connections and listeners to that port event. A worker that wakes on the application event calls `nxt_app_queue_notification_received()` and drains, as today. Before it sleeps it peeks its connections, because `FD_CLOSE` is reported only once; PostgreSQL does the same ([waiteventset.c](https://github.com/postgres/postgres/blob/REL_18_0/src/backend/storage/ipc/waiteventset.c#L1622-L1651)). The Windows port layer asserts that every message for the shared port is enqueueable; this holds today, because a full application queue answers 500 (`src/nxt_router.c:8308-8313@872bf041`, verified). The router creates the application queue section and the event when it starts an application and names both in the START_PROCESS message it sends to main; today that message carries the two descriptors instead (`src/nxt_router.c:721-722@872bf041`, verified). Main writes the names into the prototype's startup block (APP_QUEUE, APP_EVENT), and the prototype into each worker's. The START_PROCESS that the router sends a prototype for a new worker carries no descriptors and stays unchanged.
- **Rejected.** A shared socket read by many workers: no Windows message socket can be read safely by several processes. Per-worker doorbells chosen by the router: they change the scheduling semantics. `WaitOnAddress`: it works within one process only. The process note's semaphore: a semaphore whose count can exceed one wakes workers for empty queues, while an auto-reset event cannot over-count and matches the 0-to-1 rule. E15 tests both.
- **Evidence.** Report §1; research/process-and-ipc-design.md D4 and Q7; research/portability-audit.md §2 and §8.
- **Risk.** Lost wake-ups (E15). Foreign event loops cannot wait on an event object (D20).

### D10. Event engine stages

- **Stage 1, M4: `wsapoll` engine.** A copy of `src/nxt_poll_engine.c` (737 lines) keyed by `SOCKET` in its lvlhsh, with level semantics, oneshot by removal and `nxt_unix_conn_io` semantics over Winsock calls. The doorbell is an emulated AF_UNIX socket pair in the poll set: the engine listens on a path in the runtime directory, connects, accepts once and closes the listener at once. When E05 passes, it also checks that the accepted peer is its own pid, so no other process can become the doorbell. Console events, child exits and posts go through the locked work queue and one write to the doorbell. Every router engine keeps polling the shared listener: WSAPoll has no one-port rule, and the accept herd costs nothing at development scale.
- **Stage 2, M7: `psn` engine.** One completion port per engine. Connections and ports register with `SOCK_NOTIFY_TRIGGER_LEVEL | SOCK_NOTIFY_TRIGGER_ONESHOT` and re-arm after each delivery, as the eventport engine does (`src/nxt_eventport_engine.c:596-630@872bf041`). A BLOCKED direction is latched and not re-armed. Changes are batched into the next `ProcessSocketNotifications` call. Close queues `SOCK_NOTIFY_OP_REMOVE` and reports "changing" until the REMOVE notification arrives. Completion keys carry a slot index and a generation, so packets queued before a close are dropped. The doorbell, console events and child exits use `PostQueuedCompletionStatus` with reserved keys. The timeout comes from `nxt_timer_find()`, and both `ERROR_TIMEOUT` and `WAIT_TIMEOUT` mean timeout. One engine owns each listening socket with `LEVEL | PERSISTENT` and hands accepted sockets round-robin to the other engines with `nxt_event_engine_post()`; each receiving engine registers the socket with its own port. Before the hand-off becomes the default, M7 measures it against AFD poll on the shared listener from every engine and against `AcceptEx` on the owning engine (research/windows-io-model.md question 6; research/portability-audit.md §3); the hand-off stays unless another model cuts router CPU per request by more than 20% in the connection-heavy scenario. Before `create()` the engine probes the function and makes one test registration; on failure the process uses `wsapoll`.
- **Stage 3, after the preview, only with a measured gain.** `AcceptEx` on the owning engine; `TransmitFile` for large static files on server editions; completion receive last.
- **Rejected.** Completion I/O for connections (O4): it replaces the connection and port layers, and the io_uring analogue measured parity-minus. Zero-byte reads as the main engine (O3): no write readiness. select (O6): 64 sockets by default and one provider per call. AFD poll (O2) as the default: undocumented and called through ntdll, although de facto stable, because the JDK, libuv, mio, libzmq, Envoy and c-ares depend on it; M7 measures it as a reference, and it is the candidate engine if production on Windows 10 or Server 2019 ever becomes a goal. One completion port shared by all engines: FreeUnit's engines own their connections without locks. Per-engine listening sockets: after a second `SO_REUSEADDR` bind "the behavior for all sockets bound to that port is indeterminate" (verified), and Windows has no `SO_REUSEPORT` balancing.
- **Evidence.** Report §2 and §5 (conflict d); research/windows-io-model.md §1 to §7; research/portability-audit.md §3; [io_uring results-stage2.md](https://github.com/andypost/unit/blob/6ee2a59dbdc3536609ec33e1fc32a14ed7e17f15/docs/io_uring/results-stage2.md).
- **Risk.** ProcessSocketNotifications documentation gaps (E06, E07, E08). The hand-off cost, measured in M7.

### D11. Signals and service control

- **Choice.** Main installs `SetConsoleCtrlHandler`. `CTRL_C_EVENT` starts a fast quit, as SIGINT does on Unix. `CTRL_BREAK_EVENT` starts a graceful quit. That mapping is this plan's own, by analogy with Ctrl+\ and SIGQUIT: the process note maps Ctrl+Break to a fast stop, and the toolchain note warns that Ctrl+Break works only from a shared console. The documented graceful path is therefore `unitd --signal quit`. `CTRL_CLOSE_EVENT` starts a fast quit that must finish within the 5,000 ms the console allows. A control pipe `\\.\pipe\freeunit-<instance id>-ctl` carries a new option, `unitd --signal quit|stop|reopen` (graceful quit, fast quit, log reopen by path). FreeUnit has no such option today; Tier 3 adds it on Windows only, and Unix keeps its signals. Its DACL admits the unitd user and SYSTEM; it uses `PIPE_REJECT_REMOTE_CLIENTS` and `FILE_FLAG_FIRST_PIPE_INSTANCE`; a thread in main serves it and posts to the main engine. If the first instance cannot be created, unitd refuses to start, because someone else holds the name. The client finds the instance id in the pid file, opens the pipe with `SECURITY_SQOS_PRESENT | SECURITY_IDENTIFICATION`, and checks with `GetNamedPipeServerProcessId` that the server is `unitd.exe` running as the expected user before it writes. Children run with `CREATE_NO_WINDOW` and stop on QUIT port messages, as today; job J is the hard stop. Log files are opened with `FILE_SHARE_READ | FILE_SHARE_WRITE | FILE_SHARE_DELETE`, so a rotation tool can rename them while unitd holds them; reopen opens the configured path again and swaps the handle (research/process-and-ipc-design.md Q13; M1 tests it). Service mode comes after the preview: `unitd --service install|remove|run`, `StartServiceCtrlDispatcherW` within 30 seconds, STOP and SHUTDOWN to quit, control codes 128 (graceful quit) and 129 (reopen), and start failures in the Event Log.
- **Rejected.** CRT `signal()`: Windows never generates SIGTERM. A named event per command (nginx, Apache): it carries no caller identity. `GenerateConsoleCtrlEvent` towards children: `CTRL_C_EVENT` "cannot be limited to a specific process group". The toolchain note's mapping of Ctrl+C to a graceful stop: FreeUnit's Unix table sends SIGINT to the fast handler (`src/nxt_main_process.c:151-159@872bf041`, verified).
- **Evidence.** Report §1 and §5; research/process-and-ipc-design.md §2.7, D9 and D10; research/toolchain-packaging-demand.md §4.4.
- **Risk.** Console events reaching children despite `CREATE_NO_WINDOW` (E16). Another local user squatting the pipe name (E10).

### D12. Module loading and exports

- **Choice.** Modules are `<module>.unit.dll`, for example `php85.unit.dll`. Discovery lists them with `FindFirstFileW`. Before loading, it calls `AddDllDirectory` on the module's runtime directory (D13), then `LoadLibraryExW` with `LOAD_LIBRARY_SEARCH_DLL_LOAD_DIR | LOAD_LIBRARY_SEARCH_DEFAULT_DIRS`, then `GetProcAddress(module, "nxt_app_module")`. A new macro `NXT_API` marks unitd's exported surface: `__declspec(dllexport)` when building `unitd.exe`, `__declspec(dllimport)` in modules. `NXT_EXPORT` (`src/nxt_clang.h:121-129@872bf041`, verified) stays `__declspec(dllexport)` for each module's `nxt_app_module`. Linking `unitd.exe` emits `unitd.lib`, which modules link. libunit's helper objects (`nxt_lvlhsh`, `nxt_murmur_hash`, `nxt_socket_msg`, `nxt_websocket`) are linked into every module. That cuts the PHP module's imports from unitd from 18, measured with `nm` on a debug build of 2026-09-15 that predates 872bf041, to 8 by the process note's arithmetic; E23 measures both at 872bf041. Later, a function table passed at load time removes the dependence on the executable's file name, as SQLite extensions do.
- **Rejected.** A core DLL: it changes the Unix layout. Static copies of core objects in each module: duplicate globals such as `nxt_server`. `--export-all-symbols`: a GNU linker option with no MSVC-ABI equivalent.
- **Evidence.** research/process-and-ipc-design.md §1.13 and D12; research/portability-audit.md §5; research/toolchain-packaging-demand.md §2.7.
- **Risk.** Renaming `unitd.exe` breaks module loading until the function table exists.

### D13. PHP module build

- **Choice.** Non-thread-safe PHP. Link `php8.lib` from the official non-thread-safe development package of the same minor version, compiled with clang-cl `/MD` against its headers. No `php8embed.lib`; no PHP rebuild. One module per supported minor: `php84.unit.dll` and `php85.unit.dll` first, `php86.unit.dll` when PHP 8.6 ships with VS18. The Windows zip bundles the official non-thread-safe x64 runtime of each supported minor, unmodified and with its licence files, next to its module (D15). On Windows the module opens scripts with `zend_stream_init_filename()` (exported by PHP 8.4 and 8.5; `Zend/zend_stream.h:69` at php-8.5.11, verified) instead of `fopen(filename, "re")` (`src/nxt_php_sapi.c:1290@872bf041`, verified), because the UCRT rejects the "e" modifier (read from the UCRT source; E20 runs it) and because no C runtime `FILE *` should cross into `php8.dll`. Windows defaults are injected through `sapi_module.ini_defaults`, which is NULL today (`src/nxt_php_sapi.c:352@872bf041`, verified), so that `php.ini` and the application's `options` can override them: `opcache.cache_id` derived from the application name (one opcache per application, as on Unix), `opcache.file_cache` in a per-application directory, and `opcache.file_cache_fallback=1`. php-src calls `ini_defaults` before it parses `php.ini` (`main/php_ini.c:420-421` and `:603` at php-8.5.11, verified), so `php.ini` and the application's options override these defaults. The thread-safe variant (`php8ts.lib`) builds nightly as a compile and smoke test and is not shipped.
- **Rejected.** Thread-safe as the default: the module runs one request loop per process, so thread safety adds nothing. The embed library: an import library plus one `php_embed` object that FreeUnit never calls. MinGW: D1. A user-supplied PHP in Tier 3: the module needs the exact non-thread-safe minor version, and arbitrary installs multiply the test matrix.
- **Evidence.** Report §3; research/toolchain-packaging-demand.md §1.0 to §1.4 and §1.8; research/portability-audit.md §7; research/process-and-ipc-design.md §2.9 and D12.
- **Risk.** `max_execution_time` counts wall-clock time on Windows and on Apple Silicon macOS, and CPU time on Linux (`CreateTimerQueueTimer` at `Zend/zend_execute_API.c:1579`, `setitimer(ITIMER_REAL)` at `:1618`, `setitimer(ITIMER_PROF)` at `:1622`, php-8.5.11, verified); the feature matrix says so. The bundled runtime must be repackaged on every PHP security release. The official PHP DLLs carry no embedded Authenticode signature (catalog signatures were not checked), and Smart App Control has blocked the PHP extension DLLs that Laravel Herd ships ([herd-community#1757](https://github.com/beyondcode/herd-community/issues/1757)).

### D14. Test harness

- **Choice.** C tests: on Windows `nxt_test_in_child()` re-runs the test binary with `CreateProcessW` and the test name and reads the exit code; tests that need Unix-only features are compiled out on Windows and counted as skipped in the output. pytest:
  1. Import hygiene: `fcntl` only off Windows (`test/conftest.py:2@872bf041`, verified), `pwd` and `grp` behind markers, `getattr(socket, 'AF_UNIX', None)` in `test/unit/http.py` (`:61-65@872bf041`, verified).
  2. A `unix_only` marker set per file.
  3. One platform module in `test/unit/` with `spawn()`, `stop_graceful()` (`unitd --signal quit`), `kill_tree()` (a job object with `KILL_ON_JOB_CLOSE` created by pytest through ctypes), `alive()` and `children()` (psutil), `count_handles()` (psutil `num_handles()` deltas, never absolute counts) and `control_address()`. `conftest.py` (1,036 lines at 872bf041) calls only these for platform work.
  4. unitd runs with `--control 127.0.0.1:<random free port>`. The AF_UNIX control socket gets its own small client test in C, because CPython 3.13 and 3.14 have no `socket.AF_UNIX` on Windows (E14).
  The target is the 68 test files that match no Unix-only pattern by the audit's regex. The regex over-counts the Unix side and every file depends on `conftest.py`, so the portable set is uncertain in both directions; each file is checked when it is enabled.
- **Rejected.** Wine as a test platform: upstream Wine has no usable AF_UNIX support (the toolchain note found only a Wine Staging patchset), and it runs a different toolchain. Tests through WSL interop: they would test the Linux binary.
- **Evidence.** Report §3; research/toolchain-packaging-demand.md §3.5 to §3.7; research/portability-audit.md §6.
- **Risk.** Windows branches spread through `conftest.py` and rot. Rule: they live in the platform module only.

### D15. Packaging and signing

- **Choice.** The preview artefact is `freeunit-<version>-windows-x64.zip` on GitHub Releases with `SHA256SUMS`. It holds `unitd.exe`, the module DLLs, the bundled PHP runtimes (D13), licence files and a README that states the Visual C++ v14 redistributable prerequisite (https://aka.ms/vc14/vc_redist.x64.exe) and the first-run SmartScreen prompt. `unitctl.exe` ships only if E21 passes, with TCP control only. A winget manifest uses `InstallerType: zip`, `NestedInstallerType: portable`, `PortableCommandAlias: unitd` and `PackageDependencies: Microsoft.VCRedist.2015+.x64`, the shape of the ApacheLounge.httpd and PHP.PHP.8.5 manifests. An own Scoop bucket serves Scoop users; Scoop's main bucket asks for 500 stars and 150 forks, and FreeUnit had 57 and 8 on 2026-10-07 (verified). Signing: apply to SignPath Foundation during Phase 0; sign and timestamp every `.exe` and `.dll` that FreeUnit builds; document that PHP's own DLLs carry no embedded signature. Before each announcement, submit the new binaries to the Microsoft Security Intelligence portal and check them on VirusTotal. An MSI built with WiX v7, with service mode and a firewall rule, comes after the preview.
- **Rejected.** MSIX: a packaged service needs administrator rights and a restricted capability. EV certificates: they no longer bypass SmartScreen. App-local Visual C++ runtime DLLs: Microsoft advises against them. Chocolatey before a stable release: every new package version goes through human review unless the package is trusted.
- **Evidence.** Report §3; research/toolchain-packaging-demand.md §4.1 to §4.8.
- **Risk.** SmartScreen prompts during the first weeks of a signing identity. Antivirus false positives: antivirus engines flagged a FrankenPHP Windows zip in 2026 ([FrankenPHP issue 2446](https://github.com/php/frankenphp/issues/2446)). Smart App Control and the unsigned PHP DLLs.

### D16. CI

- **Choice.** One workflow, `.github/workflows/windows.yml` in FreeUnit, with the path filters of `build-test-macos.yml` (`configure`, `auto/**`, `src/**`, plus `test/**`):
  - **win-core** on `windows-2025` (Visual Studio 2026, the early warning of D1): MSYS2 through `msys2/setup-msys2` pinned by SHA; the MSVC environment from `vswhere` and `vcvarsall.bat` in one PowerShell step, with no third-party action; vcpkg with `VCPKG_BINARY_SOURCES=clear;files,<dir>,readwrite` and `actions/cache` keyed on the vcpkg commit, triplet and manifest hash; `./configure --tests`, `make`, `build/tests`; `build/autoconf.err` and the logs uploaded as artefacts.
  - **win-php** on `windows-2022` (the release toolset of D1): the PHP 8.5 non-thread-safe zip and development package, checked against `sha256sum.txt` and cached; the module build; the pytest smoke subset over TCP control.
  - **mingw-cross** on `ubuntu-24.04`, optional: a compile-only build of the core with `x86_64-w64-mingw32` and `-Werror`. Delete it if it fails for MinGW-only reasons more than twice in a quarter.
  - **nightly**: the wide pytest subset in 2 to 4 shards, the thread-safe PHP compile, and the release package built on `windows-2022` with a smoke test (unzip, start, one request).
  Windows jobs are non-required until they have run green for 14 days; then win-core becomes required, and win-php becomes required after the first demo. No job uses `windows-latest`. Every action is pinned by SHA and runs on Node 24.
- **Rejected.** Wine; self-hosted runners; larger runners, which are billed even for public repositories.
- **Evidence.** research/toolchain-packaging-demand.md §3.1 to §3.7 and §7.1.
- **Risk.** Runner image changes: `windows-latest` moved twice in nine months. Defender scanning slows test legs and makes renames fail at random. Test legs therefore disable real-time monitoring, as PostgreSQL's CI does ([pg-ci.yml](https://github.com/postgres/postgres/blob/d374280e1837adc1f8218bb0d69ffbae5d808fc6/.github/workflows/pg-ci.yml#L880-L884)), and one nightly job keeps it on, because developer machines run with it on (D22).

### D17. Descriptor types

- **Choice.** On Windows `nxt_socket_t` is `SOCKET` and `nxt_fd_t` is `HANDLE`. `NXT_SOCKET_INVALID` is `INVALID_SOCKET` and `NXT_FILE_INVALID` is `INVALID_HANDLE_VALUE`. Unix keeps `int` and `-1`. Shared code stops comparing descriptors with `-1` or `< 0` and uses the constants. A regex count made for this plan on 2026-10-07 is the size basis. `git grep -h -E '(fd|socket|sock|s)(\[[0-9]\])? *(==|!=) *-1\b|(fd|socket)(\[[0-9]\])? *< *0\b|NXT_FILE_INVALID|= -1;' 872bf041 -- 'src/*.c' 'src/*.h' ':!src/test/*' ':!src/nxt_unit.c'` prints 333 lines. The pattern `\bint +[a-z_]*fd\b|\bint +fd\[` over `src/*.c` and `src/*.h` without the tests matches 83 lines. Both are upper bounds, because the patterns also match values that are not descriptors. Port message attachment slots become a typed struct on all platforms: a descriptor on Unix, `{kind, value or name}` on Windows. The select engine is not built on Windows, because it indexes an array by descriptor (`src/nxt_select_engine.c:77@872bf041`). C runtime descriptors appear only where a C runtime function requires one.
- **Rejected.** Keeping `int` and casting: Microsoft states that a socket may take any value except `INVALID_SOCKET`. A table of fake small integers: Microsoft's Redis port needed such "a virtual file descriptor mapping layer".
- **Evidence.** research/portability-audit.md §3; research/windows-io-model.md §1 and §2.
- **Risk.** A wide mechanical change in Linux code paths. It lands first, in M1, behind the full Linux CI.

### D18. Control API channel

- **Choice.** The default `--control` on Windows is an AF_UNIX socket file, `<rundir>\control.unit.sock`. The runtime directory's DACL admits only the unitd user and SYSTEM, and Windows requires write permission on a socket file to connect to it ([AF_UNIX comes to Windows](https://devblogs.microsoft.com/commandline/af_unix-comes-to-windows/)); socket files take the directory's DACL, so it already limits callers. On accept, the controller reads the peer pid (`SIO_AF_UNIX_GETPEERPID`), opens the peer's token and compares its user SID with unitd's, or with the Windows equivalents of `--control-user` and `--control-group`. A TCP control socket stays available and is documented as unauthenticated, as it is on Unix (`src/nxt_controller.c:902-906@872bf041`, verified). Tests and unitctl use TCP until their clients speak AF_UNIX on Windows.
- **Rejected.** TCP as the default: any local user could reconfigure FreeUnit and run code as its user. A named-pipe control channel: new server code, and no pipe support in curl or unitctl.
- **Evidence.** research/process-and-ipc-design.md D13; research/toolchain-packaging-demand.md §2.4 and §3.5.
- **Risk.** If E05 fails, the directory DACL is the only check.

### D19. Static files and Windows path safety

- **Choice.** Static files are opened with `CreateFileW` and read into memory buffers on the engine thread, as on Unix today (`src/nxt_http_static.c:1899@872bf041`, `:1964`, `:1994`); `ReadFile` replaces the `mmap()` fallback in `nxt_sendfile()` (`src/nxt_conn_write.c:295-316@872bf041`). Configuration validation rejects `chroot`, `follow_symlinks: false` and `traverse_mounts: false` on Windows until a reparse-point-aware path walk exists; Windows has no `openat2()` `RESOLVE_*` equivalent. The URI-to-path step rejects names that Windows resolves differently from their spelling: a trailing dot or space in a path component, an alternate data stream (any `:` in a component, for example `::$DATA`), and reserved device names (`CON`, `NUL`, `COM1` and the rest). At configuration time, unitd canonicalises the static prefix of each `share` path, the part before its first variable, with `GetFinalPathNameByHandleW`, so a share under a junction, a symbolic link, a mount point or a `subst` drive keeps working. Per request, after `CreateFileW`, the handler compares `GetFinalPathNameByHandleW` of the file with that canonical prefix plus the request-derived remainder, case-insensitively and with separators normalised, and answers 404 on any difference. That catches 8.3 short names and reparse points inside the share. Links inside a share that point elsewhere are therefore refused on Windows, which is stricter than the Unix default; the feature matrix says so. `NXT_HAVE_CASELESS_FILESYSTEM` is set on Windows (`src/nxt_file.h:46-58@872bf041`, verified). Windows absolute paths with drive letters are accepted in `share`, `root` and `working_directory`.
- **Rejected.** `TransmitFile` for static files in Tier 3: client editions run at most two at a time, and development needs none.
- **Evidence.** research/history-upstream-and-fork.md §5 (nginx's Windows path fixes in 0.7.65, 0.7.66 and 1.3.1); research/portability-audit.md §3; research/windows-io-model.md §2.
- **Risk.** Path bugs are security bugs. Every rule gets a test that fails without it (§7, rule 22).

### D20. External applications and the libunit public API

- **Choice for the preview.** `type: external` (Go, Node.js), Python ASGI and the Node.js module are not supported on Windows. The design kept for later: the prototype starts the user's binary with `CreateProcessW`, since Windows has no exec; `NXT_UNIT_INIT` carries port socket paths, section and event names, and inherited handle values; `nxt_unit_port_t.in_fd` and `out_fd` and the descriptors in `nxt_unit_init_t` (`src/nxt_unit.h:81-82@872bf041`, `:179-181`, verified) take a new type `nxt_unit_fd_t`, which is `int` on Unix and `intptr_t` on Windows, so the Unix API and ABI do not change; foreign event loops wait through a helper thread that signals `uv_async` or `call_soon_threadsafe`, because the application event is not a socket; Go drops its own `port_send` and `port_recv` and lets the C side do the I/O. Until then, M5 makes the refusal explicit. Today both fail on Windows with compiler errors: `src/nodejs/unit-http/package.json` has no `os` field and runs node-gyp on install (`:10@872bf041`), and `go/` carries build tags only for darwin and for linux or netbsd (`go/ldflags-darwin.go:1@872bf041`, `go/ldflags-lrt.go:1@872bf041`). M5 adds `"os": ["!win32"]` to the package, so npm refuses the install on Windows, and a `//go:build !windows` constraint to the files in `go/`, so `go build` reports that no files match.
- **Rejected for now.** Porting `go/port.go`'s datagram transport to Windows.
- **Evidence.** research/portability-audit.md §4, which poses these designs as options; research/process-and-ipc-design.md §1.12 and §4.1. The type `nxt_unit_fd_t` is this plan's proposal.
- **Risk.** The Windows libunit ABI differs from the Unix one (decision O5).

### D21. Windows file layout

- **Choice.** unitd computes its default paths at start, because a portable zip can be unpacked anywhere. `configure` options and command-line options override every default.
  - Program files: `unitd.exe` in the install directory, found with `GetModuleFileNameW`; modules in `<install dir>\modules`; bundled PHP runtimes in `<install dir>\php\<minor>` (D13).
  - Per-user data under `%LOCALAPPDATA%\FreeUnit`, read with `SHGetKnownFolderPath(FOLDERID_LocalAppData)`: state in `state`, logs in `log` (default `log\unit.log`), temporary files for request bodies and compression in `tmp`.
  - Runtime files: the pid file `%LOCALAPPDATA%\FreeUnit\run\unit.pid` holds the pid and the instance id, so `unitd --signal` can find the instance; the instance directory `run\<instance id>` (an 8-character id) holds `lock`, `control.unit.sock`, the port sockets `p<pid>-<port id>` and the doorbell sockets. Main creates `run`, `state`, `log` and `tmp` with a DACL for the current user and SYSTEM only (D5, D18).
  - Path length: an AF_UNIX path must fit `UNIX_PATH_MAX`, 108 bytes including the terminating NUL. At start unitd computes the longest socket path it may create (`p`, a 10-digit pid, `-` and a 5-digit port id) and exits with a message that names that path and the `--runstatedir` option if it does not fit. With the default layout that leaves room for a profile folder name of about 44 ASCII characters (`C:\Users\` plus the name plus `\AppData\Local\FreeUnit\run\` plus the instance id and the 17-character socket name). The fallback is a shorter run directory chosen by the user, for example `C:\fu-run`. A `\\?\` prefix is no help, because AF_UNIX does not take one (unverified; E04 step 8 checks the limit and the prefix).
  - A Windows service, after the preview, uses `%ProgramData%\FreeUnit` instead, with a DACL for the service account.
- **Rejected.** The Unix defaults under `/usr/local` (`auto/options:161-184@872bf041`, verified). A writable directory next to `unitd.exe`: it may sit under Program Files, where a standard user cannot write. `%TEMP%` for runtime files: it is no shorter than `%LOCALAPPDATA%`, and cleanup tools delete its contents.
- **Evidence.** research/process-and-ipc-design.md §2.5 (`UNIX_PATH_MAX` 108); research/toolchain-packaging-demand.md §4.4 and §4.8.
- **Risk.** Long or non-ASCII profile paths. Whether Windows encodes AF_UNIX paths in UTF-8 or in the ANSI code page is unverified; E04 step 8 includes a non-ASCII path.

### D22. File sharing, renames and antivirus

- **Choice.**
  - Files that unitd opens for reading or appending use `FILE_SHARE_READ | FILE_SHARE_WRITE | FILE_SHARE_DELETE`, unless exclusion is the point (the lock file uses share mode 0).
  - Renames use `MoveFileExW` with `MOVEFILE_REPLACE_EXISTING | MOVEFILE_WRITE_THROUGH`. When a rename or a delete fails with `ERROR_SHARING_VIOLATION` or `ERROR_ACCESS_DENIED` because another process holds the file (Defender real-time scanning, an editor, a backup agent, the search indexer), unitd retries 10 times with a delay that starts at 10 ms and doubles up to 1 second, about 3 seconds in all, then reports the error with the path. This covers the state store (configuration, certificates, scripts), log reopen, temporary files and stale socket files in the run directory.
  - Temporary files for request bodies and compression are created with `FILE_FLAG_DELETE_ON_CLOSE`, so a failed delete cannot leave them behind.
  - A C test in M1 holds a target from a second process without `FILE_SHARE_DELETE` and checks both outcomes: the rename succeeds when the holder closes within the retry window, and fails with a clear error when it does not.
  - Defender: CI test legs disable real-time monitoring, as PostgreSQL's CI does ([pg-ci.yml](https://github.com/postgres/postgres/blob/d374280e1837adc1f8218bb0d69ffbae5d808fc6/.github/workflows/pg-ci.yml#L880-L884)), and one nightly job keeps it on (D16). E12 records the Defender state and measures worker start with real-time scanning on, because developer machines run with it on. The Windows documentation describes an optional Defender exclusion for the state and tmp directories, as Laravel Herd's documentation does for its own directory (research/toolchain-packaging-demand.md §6.1), and says what it trades away.
- **Rejected.** Treating sharing violations as fatal: state-store writes would fail at random on developer machines. Unbounded retries: a held file would block reconfiguration.
- **Evidence.** The PostgreSQL wiki on antivirus interference and Herd's installation guide (research/toolchain-packaging-demand.md §6.1); research/portability-audit.md §5 (open-then-unlink temporary files).
- **Risk.** Retries slow a reconfiguration while a scanner holds a file. The retry budget is one constant, tuned in M1.

## 4. Architecture

### 4.1 Process tree

```
unitd.exe                                  main: console process, foreground
|   owns: job J (KILL_ON_JOB_CLOSE, handle not inheritable)
|         private namespace FreeUnit-<instance id>
|         runtime directory <rundir> (DACL: unitd user + SYSTEM) and <rundir>\lock
|         console control handler; control pipe thread; state-store thread
|         listening sockets (bound here, passed to the router)
|
|-- unitd.exe --role discovery              LoadLibraryExW each *.unit.dll, send MODULES, exit
|-- unitd.exe --role controller             control API on <rundir>\control.unit.sock
|-- unitd.exe --role router                 N engines (N = CPU count by default),
|                                           application queues, application events
|-- unitd.exe --role prototype --app drupal
|     |   loads php85.unit.dll, runs php_module_startup() once, stays resident,
|     |   answers START_PROCESS, spawns, watches and reports its workers
|     |-- unitd.exe --role app --app drupal    setup, then start (nxt_unit_run)
|     |-- unitd.exe --role app --app drupal
|
every child:
  CreateProcessW("unitd.exe", "--role ... --startup 0x<handle>",
                 STARTUPINFOEXW + PROC_THREAD_ATTRIBUTE_HANDLE_LIST
                   = {startup section, log handle},
                 CREATE_SUSPENDED | CREATE_NO_WINDOW | CREATE_UNICODE_ENVIRONMENT)
  parent: bind the child's port listener <rundir>\p<child pid>-<port id>
          -> WSADuplicateSocketW(child pid) -> write startup block
          -> AssignProcessToJobObject (main only)
          -> ResumeThread -> RegisterWaitForSingleObject(child, WT_EXECUTEONLYONCE)
          -> close its copy of the listener after the child's first message
  child exit: wait callback -> post {pid, handle} to the parent's engine
              -> existing SIGCHLD logic with GetExitCodeProcess
```

### 4.2 Port channel

```
owner X, port P                              writer Y (any FreeUnit process)
---------------                              ------------------------------
listen on <rundir>\p<pid X>-<P>              on first write to P:
                                             connect to <rundir>\p<pid X>-<P>
accept -> connection c                       send HELLO {pid Y, port id of Y}
peer = SIO_AF_UNIX_GETPEERPID(c)
check peer == pid Y and registry
  (pid + creation time)
reply {connection index i}           ---->   remember i for markers to X

frame on c:  | length u32 LE | nxt_port_msg_t (16 B) | attachments | payload |
             length checked against max_size + header + attachment limit
             reader: read length and header, then exactly the rest of the frame
             writer: at most one partial frame pending; NXT_AGAIN until flushed

small message, no attachment:  item in X's port queue (shared memory, unchanged)
large message or attachment:   frame on c, plus marker {READ_SOCKET, i} in X's queue
reader on a marker:            consume the next frame of connection i
queue 0 -> 1 transition:       READ_QUEUE frame on c (unchanged wake-up rule)
identity:                      nxt_recv_msg_cmsg_pid() returns the peer pid of c
```

### 4.3 Shared-memory handover (worker response data to the router)

```
worker W                                          router R
--------                                          --------
CreateFileMappingW(INVALID_HANDLE_VALUE,
    size = PORT_MMAP_SIZE (4 KiB + 10 MiB),
    name = "<namespace>\seg-<W>-<R>-<id>-<128 random bits>")
    ERROR_ALREADY_EXISTS -> hard failure
MapViewOfFile; header {id, src_pid W, dst_pid R}
frame MMAP {id, name}             ---------->    OpenFileMappingW(name)
                                                 MapViewOfFile(exact PORT_MMAP_SIZE)
                                                   (fails if the section is smaller)
                                                 check header id, src_pid, dst_pid
                                                 (https://github.com/freeunitorg/freeunit/pull/173)
messages name chunks {mmap_id, chunk_id, size}; layouts hold offsets only
free chunks: SHM_ACK as today; the section disappears with its last view and handle
blobs (config push, config store, certificate, script):
  receiver answers ACK -> creator closes its handle
```

### 4.4 Engine wait loop

The loop in `nxt_event_engine_start()` does not change: run the work queues, take the timeout from `nxt_timer_find()`, call the engine's `poll()`, expire timers (`src/nxt_event_engine.c:507-558@872bf041`). Only `poll()` differs.

```
stage 1: wsapoll_poll(engine, timeout)
  apply batched changes to the WSAPOLLFD array (lvlhsh keyed by SOCKET)
  n = WSAPoll(fds, nfds, timeout)            fds[0] is the doorbell socket
  fds[0] readable -> drain doorbell bytes; byte 0 = posts, other bytes = mapped events
  for each ready entry -> read or write handler; oneshot entries removed after delivery

stage 2: psn_poll(engine, timeout)
  rc = ProcessSocketNotifications(port, nchanges, changes, timeout,
                                  max, entries, &got)
       changes = queued ENABLE (IN | OUT | HANGUP, LEVEL | ONESHOT) and REMOVE
       check changes[i].registrationResult for every change
  rc == WAIT_TIMEOUT or ERROR_TIMEOUT -> return
  for each entry:
    key == KEY_POST  -> run the locked work queue; dwNumberOfBytesTransferred = event
    key == KEY_CHILD -> child exit posted by a wait callback -> reaper
    otherwise slot = index(key); generation(key) != slot.generation -> drop
      events = SocketNotificationRetrieveEvents(entry)
      REMOVE            -> slot quiesced; finish the deferred close; free the slot
      IN, HANGUP, ERR   -> read handler, or latch read_ready if reads are BLOCKED
      OUT               -> write handler, or latch if writes are BLOCKED
      directions still wanted and not BLOCKED -> queue ENABLE for the next call
  listener engine: accept in batches -> nxt_event_engine_post(target engine, socket)
                   target engine registers the socket with its own port
```

### 4.5 Startup block

The parent creates an inheritable, unnamed section, writes the block, passes the handle value on the command line and closes its own handle after `ResumeThread`. The child maps the view, checks every length with `nxt_span_t` before use, copies what it needs, then unmaps and closes the handle.

```
offset 0: fixed header (all fields little-endian)
  u32 magic "FUSB"            u16 version (1)        u16 header size
  u32 total size (must equal the size the child computes from the view)
  u32 role (NXT_PROCESS_*)    u32 flags
  u32 parent pid              u64 parent creation time (FILETIME)
  u32 stream                  u32 own port id
  u8  instance id[16]
then records { u16 type, u32 length, length bytes }:
  RUNDIR        UTF-8 path of the runtime directory
  NAMESPACE     private namespace alias
  PEER          { process type, pid, creation time, port id } for each port
                the keep matrix keeps
  PORT_LISTENER WSAPROTOCOL_INFOW of the child's own port listener (D5)
  RPC_COUNTER   section name of the RPC stream counter
  APP_NAME      application name
  APP_CONF      application configuration, JSON, UTF-8
  MODULE        module DLL path and its runtime directory
  WORKDIR       working directory
  APP_QUEUE     section name of the application queue      (prototype and app)
  APP_EVENT     event name of the application doorbell     (prototype and app)
  LIMITS        shm_limit, request_limit
  HANDLES       { kind, inherited handle value } for inherited handles (log)
  TOKEN         256-bit connection token (only if E05 selects tokens, D6)
unknown record types are an error in version 1
```

## 5. Phase 0 experiments

Phase 0 runs from 2026-10-12 to 2026-12-18. Every experiment lives in `experiments/Exx-<name>/` in this repository with its program, a build command and a README that repeats the question, the expected result and the decision it unblocks. Raw output goes to `experiments/Exx-<name>/results/<date>-<build>.txt` together with the Windows edition, the build number and the compiler version. Results are never edited after the fact.

Machines:

- Hosted runners through `.github/workflows/experiments.yml` in this repository: `windows-2022` (Windows Server 2022, build 20348) and `windows-2025` (Windows Server 2025, build 26100).
- One Windows 11 x64 virtual machine, version 24H2 or later, with a standard user and a second local user, for E10, E16 and E19.
- Optional virtual machines with Windows 10 22H2 (build 19045) and Windows Server 2019 (build 17763) for E05 and E13.

Compiler for every C program: clang-cl from the Visual Studio 2022 17.14 Build Tools, `/W4 /WX /MD`, linked with lld-link or link.exe.

"Blocking" marks the experiments that gate G0 (§1). The others record behaviour that a later milestone needs.

#### E01. PHP exports and a minimal SAPI link (blocking)

- **Question.** Does a minimal SAPI built with clang-cl link against the official non-thread-safe import library and run a script? Does MinGW gcc fail as predicted? Does LLVM-MinGW clang, the one fully open toolchain, link it?
- **Reading.** Download `php-8.5.11-nts-Win32-vs17-x64.zip`, `php-devel-pack-8.5.11-nts-Win32-vs17-x64.zip` and the thread-safe pair; check them against https://windows.php.net/downloads/releases/sha256sum.txt. Run `dumpbin /exports php8.dll` and `dumpbin /exports php8ts.dll`. Count the names that contain `@@`. Confirm the exports `php_module_startup`, `php_module_shutdown`, `sapi_startup`, `sapi_shutdown`, `php_request_startup`, `php_request_shutdown`, `php_execute_script`, `zend_stream_init_filename` and `_emalloc@@8`.
- **Program** (`e01.c`, about 120 lines). Define a `sapi_module_struct` named "e01" with `ub_write` writing to stdout and `startup` calling `php_module_startup()`. `main()` calls `sapi_startup()`, then `startup`, then for each script argument `php_request_startup()`, `zend_stream_init_filename()` and `php_execute_script()`, then `php_request_shutdown()`, and finally `php_module_shutdown()` and `sapi_shutdown()`. It calls `emalloc()` and `efree()` once, so a `__vectorcall` symbol is referenced. Script: `<?php echo PHP_VERSION, " ", PHP_ZTS ? "ts" : "nts", "\n";`. Build: `clang-cl /MD /I<devel>\include /I<devel>\include\main /I<devel>\include\Zend /I<devel>\include\TSRM /DZEND_WIN32 /DPHP_WIN32 /DZEND_DEBUG=0 e01.c php8.lib`; for the thread-safe variant add `/DZTS=1` and link `php8ts.lib`. Then build the same file with MSYS2 UCRT64 gcc. Then build it with LLVM-MinGW (https://github.com/mstorsjo/llvm-mingw, the `x86_64-w64-mingw32-clang` UCRT release): `clang -target x86_64-w64-mingw32 -O1 -Wall` with the same include paths and defines, linking `php8.lib` directly, and record whether clang mangles the `__vectorcall` reference as `_emalloc@@8` and whether `lld` accepts the MSVC import library. Finally, build the SAPI as `e01mod.dll` with a small loader that calls `AddDllDirectory()` on the unpacked PHP directory and then `LoadLibraryExW()` with `LOAD_LIBRARY_SEARCH_DLL_LOAD_DIR | LOAD_LIBRARY_SEARCH_DEFAULT_DIRS`, and record the minimum setup that finds `php8.dll` and the extension DLLs (research/process-and-ipc-design.md Q11; D12).
- **Expected.** `php8ts.lib` lists 6,116 exports and `php8.lib` 6,079, 340 of them `@@` names in each (the thread-safe count from research/toolchain-packaging-demand.md §1.2; the non-thread-safe count verified on 2026-10-07 by parsing the short import records of `php-devel-pack-8.5.11-nts-Win32-vs17-x64.zip`). Both export every function listed above and no `php_embed_*` symbol. clang-cl links, and the program prints `8.5.11 nts`. The defines may need adjusting to match `include/main/config.w32.h` of the development package. gcc fails with undefined references such as `_emalloc` (research/toolchain-packaging-demand.md §1.2). LLVM-MinGW: unverified. clang supports `__vectorcall` on x86_64 Windows targets, so the reference may resolve; the UCRT dependency and any `__declspec(dllimport)` mismatch in the PHP headers are the likely failure points.
- **Unblocks.** D1 and D13. A passing LLVM-MinGW row reopens D1: a fully open toolchain for `unitd.exe` and the PHP module becomes possible, and the `mingw-cross` CI job (D16) stops being compile-only. It does not change G0 either way.
- **If not.** Try `cl.exe`. If clang-cl and `cl.exe` both fail, G0 fails. An LLVM-MinGW failure only confirms D1.

#### E02. Configure under MSYS2 with clang-cl (blocking)

- **Question.** How far do `configure` and `auto/` get with clang-cl, and how much must change?
- **Reading.** On `windows-2025`, install MSYS2 (MSYS environment, packages `make` and `diffutils`), load the MSVC environment with `vcvarsall.bat x64`, export `CC=clang-cl`, check out FreeUnit 872bf041, and apply a five-line local patch to `auto/os/test` that adds `MSYS*` to the `MINGW*` case and sets `echo=echo`. Run `./configure --tests` and keep `build/autoconf.err`.
- **Record.** How `auto/cc/test` identifies clang-cl; which flags fail; every failing probe; whether run-probes execute (`[ -x $NXT_AUTOTEST ]` against `.exe`); the edits needed to reach `make`.
- **Expected.** Compiler identification and GCC-style flags fail first; most Unix-facility probes report "not found" (research/toolchain-packaging-demand.md §2.7).
- **Unblocks.** D1. Keep `configure` and `auto/` if the Windows changes stay within about 1,000 lines in `auto/cc`, `auto/os`, `auto/feature` and `auto/make`. Otherwise write a Meson proposal before M1.

#### E03. clang-cl on the atomics and GNU extensions (blocking)

- **Question.** Does clang-cl compile FreeUnit's GCC-only constructs unchanged?
- **Program** (`e03.c`). Copy the GCC block of `src/nxt_atomic.h:17-89@872bf041` and `nxt_inline` and `nxt_noinline` from `src/nxt_clang.h:11-12@872bf041`. Exercise `nxt_atomic_cmp_set`, `nxt_atomic_fetch_add`, `nxt_atomic_release` and `nxt_cpu_pause` on `uint32_t` and `uintptr_t` operands, and call `__builtin_ffs` and `__builtin_ffsll`. Build with `clang-cl /O2 /W4 /WX`; disassemble with `llvm-objdump -d`. Also build with `cl /std:c17` and keep its errors.
- **Expected.** clang-cl compiles it; the object code contains `lock cmpxchg`, `lock xadd` and `pause` (unverified). `cl.exe` rejects `__attribute__` and the `__sync` builtins.
- **Unblocks.** D1: no MSVC branch in `src/nxt_atomic.h`.

#### E04. AF_UNIX sockets under WSAPoll and ProcessSocketNotifications (blocking)

- **Question.** Can the port transport of D5 run under both engine stages?
- **Program** (`e04.c`).
  1. Delete and recreate `%TEMP%\e04\l.sock`; listen on it as L; connect a non-blocking client C; accept S.
  2. Build one `WSAPOLLFD` array with S and an accepted TCP loopback socket T. Make S readable, then T, then both. Print `revents` and `WSAGetLastError()` each time.
  3. Create a completion port. Register S with `ProcessSocketNotifications`: operation `SOCK_NOTIFY_OP_ENABLE`, events `SOCK_NOTIFY_REGISTER_EVENT_IN | SOCK_NOTIFY_REGISTER_EVENT_OUT | SOCK_NOTIFY_REGISTER_EVENT_HANGUP`, triggers `SOCK_NOTIFY_TRIGGER_LEVEL | SOCK_NOTIFY_TRIGGER_ONESHOT`, key 1. Print `registrationResult`.
  4. Send 1 MiB from C to S in 16 KiB frames; re-enable S after each IN; count notifications and check the bytes.
  5. Repeat step 3 for the listener L (IN means a pending connection), for a socket received through `WSADuplicateSocketW` in a child process (the child prints its own `registrationResult`), and for a TCP socket accepted with `AcceptEx`.
  6. Associate a fresh AF_UNIX connection with a completion port and post a zero-byte overlapped `WSARecv`; check that it completes when data arrives.
  7. Start a child suspended, bind a listener at `%TEMP%\e04\p<child pid>.sock`, duplicate it into the child with `WSADuplicateSocketW`, connect to it and write 100 bytes before resuming the child, then resume it; the child creates its socket from the protocol information, accepts and reads the 100 bytes (D5).
  8. Bind AF_UNIX sockets at paths of 107 and 108 characters, at a `\\?\`-prefixed path, and at a path with non-ASCII characters; record which binds and connects succeed (D21).
- **Expected.** WSAPoll accepts the mixed array, because its page states no single-provider rule. ProcessSocketNotifications accepts AF_UNIX sockets (unverified; AFD poll appears to work with them in wepoll and mio, in open issues without a merged test).
- **Unblocks.** D5 and D10.
- **If not.** If step 3 fails but step 6 works, the stage 2 engine serves port sockets with zero-byte reads. If both fail, ports switch to named pipes (D5 fallback) and E09 decides their details.

#### E05. Peer pid of an AF_UNIX connection (decides the D6 variant; not a G0 condition)

- **Question.** Does `SIO_AF_UNIX_GETPEERPID` return the connecting process, on which builds, and after duplication?
- **Program** (`e05.c`). The server accepts and calls `WSAIoctl(s, SIO_AF_UNIX_GETPEERPID, NULL, 0, &pid, sizeof(ULONG), &bytes, NULL, NULL)`; the constant is `_WSAIOR(IOC_VENDOR, 256)` from `afunix.h`. The client prints `GetCurrentProcessId()`. Then the client duplicates its connected socket into a grandchild with `WSADuplicateSocketW` and exits, the grandchild writes, and the server asks again. Run on builds 20348 and 26100, and on 19045 and 17763 if the optional machines exist.
- **Expected.** The pid of the connecting process (unverified; the ioctl has no Microsoft reference page). After duplication, still the original connector (unverified).
- **Unblocks.** D6: identity by pid, or by token. D18: the control socket's peer check.

#### E06. HANGUP on a graceful FIN

- **Question.** When does `SOCK_NOTIFY_EVENT_HANGUP` fire, and can edge mode lose an end of file?
- **Program** (`e06.c`). A TCP loopback pair. Register the accepted socket with IN and HANGUP, once with `LEVEL | ONESHOT` and once with `EDGE | PERSISTENT`. Case A: the peer sends 1 byte, then `shutdown(SD_SEND)`. Case B: the peer sends 1 byte and FIN back to back. Case C: the peer sets `SO_LINGER {1, 0}` and closes (RST). Log every event mask from `SocketNotificationRetrieveEvents()` and every `recv()` result.
- **Expected.** With `LEVEL | ONESHOT`, IN fires in every case and `recv()` returns 0 at the FIN. Whether HANGUP fires on a FIN is unknown.
- **Unblocks.** D10: confirms level with oneshot; edge mode stays unused.

#### E07. Disable and re-enable semantics

- **Question.** How do `SOCK_NOTIFY_OP_DISABLE` and re-enabling behave?
- **Program** (`e07.c`). ENABLE, then DISABLE, then dequeue for 200 ms and print the raw `dwNumberOfBytesTransferred` values; ENABLE again and print `registrationResult`; repeat with the event filter `SOCK_NOTIFY_REGISTER_EVENT_NONE`; re-enable a ONESHOT registration in the call right after its first notification; finally call ENABLE again with the same key and only IN, and record the result and the masks that follow.
- **Expected.** Unknown. The documentation names a DISABLE event that the 10.0.20348 header does not define, and its two pages disagree on re-registration (research/windows-io-model.md §2).
- **Unblocks.** How D10 maps `disable_read`, `disable_write` and `block_*`.

#### E08. Coexistence with overlapped TransmitFile and AcceptEx

- **Question.** Can a socket registered with ProcessSocketNotifications also carry overlapped operations?
- **Program** (`e08.c`). Register a connected socket on completion port P; call `CreateIoCompletionPort(socket, P, key2, 0)`; start an overlapped `TransmitFile`; check that both kinds of packet arrive on P. Repeat with a second port Q and record which call fails. Repeat with `AcceptEx` on a registered listener.
- **Expected.** Unknown.
- **Unblocks.** D10 stage 3 only. Not needed for the preview.

#### E09. Named-pipe write atomicity

- **Question.** Are writes from several processes atomic on one message-mode pipe instance?
- **Program** (`e09.c`). The server creates a message-mode pipe with one instance. Four processes share one inherited client handle and each writes 10,000 records of 1 to 16 KiB carrying a writer id and a sequence number. The server reads in message mode and checks boundaries and per-writer order. Second run: one instance per writer.
- **Expected.** Unknown for a shared instance; atomic per instance.
- **Unblocks.** The details of the D5 fallback; confirms the control pipe of D11, which has one writer per connection.

#### E10. Pipe squatting by another local user

- **Question.** Can another local user capture the control pipe?
- **Steps** (Windows 11 VM, users A and B). As B, create `\\.\pipe\freeunit-<instance id>-ctl` and wait. As A, start unitd: creating the first instance with `FILE_FLAG_FIRST_PIPE_INSTANCE` must fail, and unitd must refuse to start with a log line. As A, run the `unitd --signal quit` client against B's server: it must refuse to write after the `GetNamedPipeServerProcessId` and token check. As B, try `ImpersonateNamedPipeClient` on A's connection: it must not get beyond identification level.
- **Expected.** As stated (research/process-and-ipc-design.md §2.4, D9).
- **Unblocks.** D11.

#### E11. Opcache with a resident prototype (blocking)

- **Question.** Do workers started with `CreateProcessW` share one opcache reliably, and does `opcache.cache_id` separate applications?
- **Program** (`e11.c`, built on E01). `e11 --proto` runs `php_module_startup()` with `opcache.enable=1`, `opcache.file_cache=<dir>` and `opcache.file_cache_fallback=1`, stays resident and keeps 32 workers alive. `e11 --worker` runs module startup, serves 50 requests of a script that includes 200 PHP files, prints `opcache_get_status(false)` start time, hits and memory, exits, and is replaced. Run one hour. Variants: (a) resident prototype; (b) no resident prototype; (c) `opcache.file_cache_fallback=0`; (d) two prototypes with different `opcache.cache_id`. Count the log lines "Unable to reattach to base address", "Opcode handlers are unusable due to ASLR" and "Base address marks unusable memory region".
- **Expected.** (a) zero failures and one start time across workers (unverified). (b) occasional failures after all processes unload `php8.dll` (research/process-and-ipc-design.md D2). (d) two caches (inference from the opcache documentation).
- **Unblocks.** D4 and D13. Thresholds: zero failures in (a) passes; up to 1% of worker starts ships with the file cache fallback on; above 1% ships with `opcache.file_cache_only=1` (risk R6).

#### E12. Worker start time without fork (blocking)

- **Question.** How long does a PHP worker take from `CreateProcessW` to its first request?
- **Program.** Time 200 spawns of `e11 --worker` with QueryPerformanceCounter, from `CreateProcessW` to the first response, with the extensions the Drupal installer needs (gd, mbstring, pdo_sqlite, xml, dom, opcache). Report p50 and p95 on `windows-2025` and on the Windows 11 VM. Record whether Defender real-time protection is on (`Get-MpComputerStatus`), and measure with it on, as on a developer machine, and off, as in CI (D22). For comparison, measure FreeUnit on Linux on similar hardware from the prototype's `fork()` to PROCESS_READY in the debug log.
- **Expected.** Unknown; no published measurement exists (research/process-and-ipc-design.md §3).
- **Unblocks.** D4. Pass: p95 at most 1,000 ms. Above that, the documentation sets `processes.spare` to at least 1 and discourages scale-to-zero on Windows. Above 2,000 ms, risk R6 applies.

#### E13. WSAPoll failed connect on build 17763

- **Question.** Does a WSAPoll engine need a workaround for proxy connects on old builds?
- **Program** (`e13.c`). A non-blocking `connect()` to 127.0.0.1 on a closed port; `WSAPoll()` for `POLLWRNORM` with a 5-second timeout; print `revents`. Then the workaround: a timeout plus `getsockopt(SO_ERROR)`. Run on 17763, 19045 and 20348.
- **Expected.** Nothing on 17763 until the timeout (unverified; the 2012 bug report and the WSAPoll page imply it); `POLLHUP | POLLERR | POLLWRNORM` from 19041 on (the WSAPoll page, verified).
- **Unblocks.** The degradation note of D2 for builds below 19041, and decision O3.

#### E14. CPython and AF_UNIX on Windows

- **Reading.** Run `python -c "import socket, sys; print(sys.version, hasattr(socket, 'AF_UNIX'))"` with every Python on `windows-2025` (3.12 is preinstalled; 3.13 and 3.14 through `actions/setup-python`). Read the state of https://github.com/python/cpython/issues/77589 and https://github.com/python/cpython/pull/137420.
- **Expected.** False for every version; the pull request still open (open and last updated on 2026-07-31 when checked on 2026-10-07).
- **Unblocks.** D14 (TCP control in tests) and D18.

#### E15. Doorbell without lost wake-ups (blocking)

- **Reading.** `src/nxt_app_queue.h:50-97@872bf041`, `src/nxt_router.c:8288-8315@872bf041` and `src/nxt_unit.c:7816-7827@872bf041`. Write down the ordering argument: the item is enqueued before the compare-and-swap of `notified`, and the worker resets `notified` before it drains.
- **Program** (`e15.c`). One producer process and 64 consumer processes share a copy of the application queue ring in a named section and one auto-reset event. Variants: a semaphore with maximum count 1, and a semaphore with a large maximum. The producer sends bursts of 1 to 10,000 items with random gaps for 10 minutes. Consumers follow FreeUnit's order: wait, reset `notified`, drain. Check that every item is consumed, that no item waits longer than one drain cycle while a consumer is idle, and count spurious wake-ups.
- **Expected.** No lost wake-ups with the event. The semaphore with a large maximum shows spurious wake-ups only.
- **Unblocks.** D9.

#### E16. Console control delivery

- **Question.** Which processes receive Ctrl+C, Ctrl+Break and console close?
- **Program** (`e16.c`, Windows 11 VM, in conhost and in Windows Terminal). Main installs `SetConsoleCtrlHandler` and starts three children with `CREATE_NO_WINDOW`; variants use `DETACHED_PROCESS` and `CREATE_NEW_PROCESS_GROUP`. The children install handlers that log what they receive; their stdout and stderr go to a log file through `STARTUPINFO`. Press Ctrl+C, press Ctrl+Break, close the window. Record who receives what, check that the children live until main sends QUIT over a pipe, and check that main's close handler finishes within 5,000 ms.
- **Expected.** Main receives all three events; children with `CREATE_NO_WINDOW` receive none (unverified).
- **Unblocks.** D11 and the creation flags of D3.

#### E17. Job objects inside other jobs

- **Program** (`e17.c`). Main creates job J with `KILL_ON_JOB_CLOSE`, starts three suspended children, assigns them, resumes them, and one child starts a grandchild. Run main under Windows Terminal, the VS Code terminal, a GitHub Actions step, and a test harness that holds its own `KILL_ON_JOB_CLOSE` job. Record each `AssignProcessToJobObject` result. Then `TerminateProcess(main)` and check that every descendant exits within 1 second.
- **Expected** (unverified). Assignment succeeds everywhere, because jobs nest since Windows 8; every descendant exits.
- **Unblocks.** D3.

#### E18. Section size check

- **Program** (`e18.c`). Create a section of `PORT_MMAP_SIZE` (4 KiB plus 10 MiB) under a name. In a second process, open it by name and map it once with a size 4 KiB larger and once with the exact size. Record `GetLastError()` for each, call `VirtualQuery` on the view, and print `dwPageSize` and `dwAllocationGranularity` from `GetSystemInfo`.
- **Expected.** The larger view fails, because "All bytes must be within the maximum size specified by CreateFileMapping"; the exact view succeeds.
- **Unblocks.** D7: the size check that replaces `fstat()`.

#### E19. Firewall prompt and low ports

- **Steps** (clean Windows 11 VM, as a standard user and as an administrator). An unsigned test program listens on 127.0.0.1:8080, [::1]:8080, 0.0.0.0:8080 and 127.0.0.1:80. Record each bind result, any firewall prompt, and the rules created (`Get-NetFirewallRule`). Record `netsh int ipv4 show excludedportrange protocol=tcp` and `netsh http show servicestate`.
- **Expected.** Loopback listeners cause no prompt, and port 80 binds unless HTTP.sys holds it (both unverified).
- **Unblocks.** The listener address in the documentation and the text of the error for `WSAEACCES` on excluded ports.

#### E20. UCRT and fopen mode "re"

- **Program** (`e20.c`). `fopen("x.php", "re")` once with the default invalid parameter handler and once after `_set_invalid_parameter_handler()` installs a handler that returns. Print the result, `errno` and the exit code.
- **Expected.** The default handler ends the process; with the returning handler, NULL and `EINVAL`.
- **Unblocks.** Nothing new: D13 already avoids `fopen()`. The result documents why.

#### E21. Rust parts on Windows

- **Reading.** Run `cargo check --target x86_64-pc-windows-msvc` in `tools/unitctl` and `src/otel` at 872bf041 on `windows-2025`.
- **Expected.** unitctl fails on hyperlocal's use of `tokio::net::UnixStream`; `src/otel` passes (both unverified).
- **Unblocks.** Whether unitctl ships in the preview, after hyperlocal moves behind `cfg(unix)`, and when OpenTelemetry enters the feature matrix.

#### E22. Demand poll (blocking)

- **Reading.** The FreeUnit maintainer (https://github.com/andypost) opens a GitHub Discussion in https://github.com/freeunitorg/freeunit/discussions (Discussions were enabled on 2026-10-07) and links it from DDEV and Drupal community channels. It runs from 2026-10-19 to 2026-12-04 and asks three questions: (1) On which operating system do you develop PHP? (2) If Windows: can you use WSL2 and Docker at work? Answers: yes; WSL2 blocked; Docker blocked; both blocked; I do not want them. (3) Would a native Windows FreeUnit development server help you? Answers: yes; no; maybe.
- **Pass.** At least 40 respondents who develop PHP on Windows, of whom at least 10%, and at least 6 people, report WSL2 or Docker blocked at work. Or, by the same date, at least 5 distinct people who maintain neither FreeUnit nor freeunit4drupal report in freeunit4drupal's issue tracker (https://github.com/freeunitorg/freeunit4drupal/issues) that they cannot use WSL2 or Docker for it. The raw counts and the links go into decisions/g0.md.
- **Unblocks.** G0 condition 3.

#### E23. Module imports from unitd

- **Reading** (Linux, 872bf041). Build every module and run `nm -D --undefined-only build/lib/unit/modules/*.unit.so | grep ' nxt_'`. List functions and data symbols per module, before and after linking libunit's helper objects into the module.
- **Expected.** 18 `nxt_` imports for the PHP module, 8 after the change (a measurement of 2026-09-15 on an older revision).
- **Unblocks.** D12: the `NXT_API` surface.

#### E24. Transport benchmark: AF_UNIX against named pipes

- **Question.** What does the AF_UNIX choice of D5 cost against message-mode named pipes? Three notes asked for this measurement (research/portability-audit.md §2, research/process-and-ipc-design.md Q4, research/windows-io-model.md question 13).
- **Program** (`e24.c`). Ping-pong of 31-byte and 16 KiB records with one to eight pairs over: AF_UNIX stream with the D5 framing under WSAPoll; AF_UNIX under ProcessSocketNotifications, if E04 step 3 passes; message-mode named pipes with overlapped I/O on a completion port; loopback TCP as a reference. A second run passes one handle per message through `DuplicateHandle`. Report messages per second and p50 and p99 latency on `windows-2025` and on the Windows 11 VM.
- **Expected.** Unknown.
- **Unblocks.** Nothing at G0, because D5 rests on engine compatibility. If pipes are more than twice as fast for 16 KiB records, the result goes into the stage 2 design review in M7.

#### Coverage of the open questions in the notes

Every open question that the notes list is handled here, in a milestone, or dropped with a reason.

- research/portability-audit.md §8: 1 E01; 2 E11; 3 E12; 4 E24 and E04; 5 E15; 6 dropped for Tier 3, because all processes run as one user (returns with isolation work); 7 the M7 listener measurement; 8 the M1 LLP64 check; 9 E02; 10 E14 and M6; 11 E23; 12 D20, deferred; 13 M2 (the state store becomes a thread); 14 after the preview (Python and wasm), dropped for Perl and Ruby.
- research/process-and-ipc-design.md §5: Q1 E12; Q2 E11; Q3 E09, and the marker-order test of M3; Q4 E24; Q5 E05 and E04; Q6 E18; Q7 E15; Q8 E03; Q9 E19; Q10 E17; Q11 E01; Q12 the pid-reuse test of M3; Q13 the log-rename test of M1; Q14 E16; Q15 dropped for Tier 3 (restricted tokens come with isolation); Q16 E10.
- research/windows-io-model.md §7: 1 E06; 2 E07; 3 E08; 4 E04; 5 E04 step 2; 6 the M7 listener measurement; 7 E13; 8 the M7 benchmark behind S8, with wepoll as a reference; 9 M7 (timer accuracy); 10 after the preview (TransmitFile belongs to stage 3); 11 an M7 test for rule 21; 12 dropped, because D3 does not use job messages for reaping; 13 E24.
- research/toolchain-packaging-demand.md §7.2: 1 this plan; 2 E02; 3 E01; 4 E20; 5 D18, E05 and E14; 6 E22; 7 E19; 8 an M8 check on a Smart App Control VM; 9 the SignPath application in Phase 0 (D15); 10 decision O4; 11 the first CI runs of M1; 12 E21; 13 dropped (ARM64 is out of scope); 14 dropped (no release builds cross-compiled from Linux, D1); 15 after the preview (MSI); 16 dropped (the published shares are enough for planning); 17 an optional reading before the first demo.

#### Gate G0, 2026-12-18

G0 passes when the three conditions of §1 hold. The record goes to `decisions/g0.md`: one row per experiment with its result and link, every decision that changed and why, the dates re-planned for M1 to M8, and the names of the owner and the reviewer.

## 6. Milestones

All milestones land in https://github.com/freeunitorg/freeunit as pull requests. Every pull request names its milestone and decision IDs and links this plan by full URL. Each milestone also carries tests: the audit warns that the C tests for changed code may add as much as the port itself (research/portability-audit.md §8), so a test budget of 0.5 to 1.0 times the new lines applies on top of the figures below.

### 6.1 Overview

| Milestone | Window | New lines | Changed shared lines | Depends on |
|---|---|---|---|---|
| M1 Build and platform base | 2027-01-04 to 2027-02-26 | 3,500 to 5,700 | 600 to 1,500 | G0 |
| M2 Process model | 2027-02-15 to 2027-04-30 | 1,200 to 2,500 | 800 to 2,000 | M1 |
| M3 Transport, identity, shared memory, broker | 2027-02-15 to 2027-04-30 | 2,900 to 5,500 | 800 to 2,000 | M1 |
| M4 WSAPoll engine, router, static files | 2027-03-15 to 2027-04-30 | 900 to 1,800 | 300 to 800 | M1, M3 |
| M5 libunit back end and PHP module; first demo | 2027-04-19 to 2027-06-11 | 1,700 to 3,500 | 300 to 800 | M2, M3, M4 |
| M6 Test harness and CI | 2027-01-04 to 2027-09-24 | 1,200 to 2,900 (Python and YAML) | in test/ | runs alongside |
| M7 ProcessSocketNotifications engine | 2027-06-07 to 2027-08-27 | 1,200 to 2,000 | 100 to 300 | M4 |
| M8 Tier 3 preview release | 2027-08-02 to 2027-09-24 | 300 to 800 | none | M5; M7 optional |

C totals for M1 to M5 and M7: 11,400 to 21,000 new lines and 2,900 to 7,400 changed lines. The new-line total is close to the audit's 10,000 to 18,500 (research/portability-audit.md §8); the build layer and the second engine explain the higher top. The changed-line total sits below the audit's 4,000 to 8,000, which the audit calls a guess bounded by the 27,900 lines of files a port must touch; planning uses the audit's figure.

A milestone may start once the interfaces it needs from its predecessors are merged, but its gate needs their gates first. That is why the windows overlap.

```
P0 --> G0 --> M1 --+--> M2 ---------------------+
                   |                            |
                   +--> M3 --+------------------+--> M5 --> first demo --> M8 preview
                   |         |                  |                          ^
                   +---------+--> M4 -----------+                          |
                                   |                                       |
                                   +--> M7 (parallel with M5) -------------+ (optional)
M6 (harness and CI) runs from M1 to M8.
```

M2 and M3 run in parallel once the startup block (§4.5) and the transport interface are agreed in their first week. M7 runs in parallel with M5. If M7 has not passed its gate by 2027-09-10, the freeze date of M8, the preview ships with the WSAPoll engine only; development scale does not need the scalable engine.

### 6.2 Milestone details

**M1. Build and platform base.**

- Deliverables:
  - `auto/`: the clang-cl compiler case, the Windows link templates (`.exe`, `.dll`, `.lib`, import library), the rewritten `MINGW*|MSYS*` branch without `auto/echo`, `.exe` run-probes if E02 needs them, the clang-cl hardening branch, vcpkg paths through the existing `--cc-opt` and `--ld-opt`, and a clear refusal of `--njs` on Windows (D1).
  - `src/`: `nxt_win32.h` in place of `nxt_unix.h` on Windows; descriptor types and attachment slots (D17); error mapping from `GetLastError()` and `WSAGetLastError()` to `NXT_E*`; time (`QueryUnbiasedInterruptTime` for the engine's monotonic clock, which, like Linux `CLOCK_MONOTONIC`, does not advance during sleep or hibernation, and `GetSystemTimePreciseAsFileTime` for wall time), random numbers, files (wide paths, `ReadFile` and `WriteFile` with offsets for `pread` and `pwrite`, `MoveFileExW` with `MOVEFILE_REPLACE_EXISTING | MOVEFILE_WRITE_THROUGH`, `FlushFileBuffers`, the sharing and retry rules of D22), threads, locks, condition variables, semaphores, thread-local storage, CPU count, page size and dynamic loading; the platform floor check (D2); the default layout of D21.
  - Type shims: `ssize_t`, `pid_t`, `uid_t`, `mode_t`, `socklen_t`, a 64-bit `nxt_off_t`, `struct iovec` to `WSABUF` (whose fields are in the other order, so no cast), `gmtime_r` and `localtime_r` to `gmtime_s` and `localtime_s` (whose arguments are in the other order), and `posix_memalign` to `_aligned_malloc` with a matching `_aligned_free`, the free-mismatch trap of research/portability-audit.md §5.
  - Command line, environment and console: `wmain` or `GetCommandLineW` with `CommandLineToArgvW`, converted to UTF-8; the environment block for `CREATE_UNICODE_ENVIRONMENT` built from UTF-8; log lines to a console written with `WriteConsoleW`, and to files as UTF-8.
  - Long paths: file paths longer than 259 characters are opened in the `\\?\` form after normalisation, and `unitd.exe` carries the `longPathAware` manifest entry.
  - The C test runner: `nxt_test_in_child()` through `CreateProcessW` (D14).
- Gate: on `windows-2025`, `./configure --tests && make && build/tests` passes the helper tests, with process and port tests compiled out and counted as skipped; no compiler warning at the M1 warning level; Linux and macOS CI green; `sh .github/scripts/check-test-hooks.sh` exits 0 on Linux; the full Linux pytest run passes after the descriptor type change; a Linux build of the core with clang `-Wshorten-64-to-32 -Wconversion` is reviewed, including the 70 `long` lines the audit lists (LLP64); a C test renames an open log file and reopens it (D11); a C test holds a file open from a second process during a rename (D22).
- Size basis: `auto/` 400 to 900 lines (nginx's MSVC layer is 267 lines at release-1.31.6; FreeUnit adds DLL modules, `.exe` probes and the Rust library name); platform 3,000 to 4,500 lines (research/portability-audit.md §8; nginx's `src/os/win32` is about 6,200 lines including services and console code); test runner 100 to 300 lines; changed lines from the descriptor change (333 and 83 lines by the regex counts of D17, both upper bounds).

**M2. Process model.**

- Deliverables: the spawn path with the startup block (D3, §4.5); role dispatch at start; job J; child supervision posted to the engine; the resident prototype spawning workers (D4); worker configuration from the startup block; the state-store thread; the console handler and the control pipe with `unitd --signal` (D11); the pid file; process titles as a no-op; validation that rejects `user` and `group` other than the current account, and `isolation`.
- Gate: C tests for the handle list, a startup-block round trip with a fuzzed parser, a supervision event within 100 ms of a child's exit, the job killing every descendant when main dies, and `--signal quit` delivery; each test fails on the pre-change code.
- Size basis: research/portability-audit.md §8 gives 1,500 to 3,000 lines for spawn, jobs, supervision, console and service code; service mode is out, so 1,200 to 2,500. libuv's `process.c` and `signal.c` total 1,770 lines. Changed files: `nxt_process.c` (1,562 lines), `nxt_main_process.c` (3,231), `nxt_application.c` (1,935), `nxt_runtime.c`, `nxt_signal.c` and `nxt_signal_handlers.c`.

**M3. Port transport, identity, shared memory and handle broker.**

- Deliverables: the Windows branch of `nxt_socket_msg.c` with framing (D5); port endpoints and lazy connections behind a narrow transport interface used by `nxt_port_socket.c` and `nxt_port.c`; HELLO and connection indexes; writer-tagged READ_SOCKET markers in `nxt_port_socket.c` and `nxt_unit.c`; per-connection identity behind `nxt_recv_msg_cmsg_pid()` (D6); named sections, the private namespace and blob acknowledgements (D7); attachments and duplication by the trusted side, with the creation time in NEW_PORT (D8); the application event in the router (D9); the emulated socket pair used by engine doorbells.
- Gate: C tests for framing (partial reads and writes, `max_size`, a bad length rejected through `nxt_span_t`), marker order with 8 writers mixing queue items and frames, rejection of an unknown peer pid, the section size check, blob acknowledgements and the pid-reuse check; each fails on the pre-change code; Linux unchanged (S3).
- Size basis: transport and broker 2,500 to 4,500 lines (research/portability-audit.md §8; libuv's `pipe.c` is 2,940 lines); sections and wake-ups 400 to 1,000 lines (an estimate without an external basis). Changed files: `nxt_port_socket.c` (2,951 lines), `nxt_port.c` (1,572), `nxt_port_memory.c` (1,176), `nxt_socket_msg.c` and `.h` (405) and the 18 descriptor send sites in 15 message kinds (research/portability-audit.md Table 2.2; the process note's table groups them into 16 rows).
- Coordination: before touching `nxt_port_socket.c`, `nxt_port_memory.c` or `nxt_unit.c`, check overlap with open pull requests with `git merge-tree --write-tree origin/master <their head>`.

**M4. WSAPoll engine, router and static files.**

- Deliverables: the `wsapoll` engine (D10 stage 1); Winsock connection I/O; accept with `ioctlsocket(FIONBIO)`; `SO_EXCLUSIVEADDRUSE` on listeners; listeners bound by main and passed with `WSADuplicateSocketW` (D8); static files with `ReadFile` and the path rules (D19); validation of Windows paths; proxy connects checked with `SO_ERROR`.
- Gate: the router serves static files and proxies on `windows-2025` in C tests and in a smoke test with the `curl.exe` that ships with Windows; the path-rule tests fail on the pre-change code; 1,000 idle connections are held.
- Size basis: the engine is 0.6 to 0.7 times the 1,225-line epoll engine, about 735 to 860 lines (research/windows-io-model.md §4); connection I/O, doorbell, static files and path rules add 200 to 900 lines.

**M5. libunit back end, PHP module and the first demo.**

- Deliverables: libunit's transport and wait loop (`WSAEventSelect` plus the application event, D9); libunit's sections; module loading with `NXT_API` and `unitd.lib` (D12); the Windows branch of the PHP module (D13); the Windows branch of `auto/modules/php`, linking `php8.lib` from a development package path; the bundled runtime layout; the explicit refusal of the Node.js package and the Go package on Windows (D20).
- Gate: the first demo below.
- Size basis: libunit 1,500 to 3,000 lines (research/portability-audit.md §8, an estimate; it replaces parts of `nxt_unit.c`, which has 8,713 lines); PHP module 100 to 300; module loading 100 to 200.

**First demo (S4 to S6), target 2027-06-11.**

- Machine: Windows 11 x64, version 24H2 or later, a standard user, no WSL2, no Docker, the Visual C++ v14 redistributable installed.
- Artefact: the nightly zip of M5, unpacked into the user's profile.
- Start: `unitd.exe` in Windows Terminal, in the foreground. For the demo the control socket is `--control 127.0.0.1:8081`, so the `curl.exe` that ships with Windows can load the configuration.
- Application: the latest Drupal 11 release, created with Composer using the bundled PHP 8.5 `php.exe`, with SQLite through pdo_sqlite. If the SQLite library in the PHP zip is older than Drupal's minimum (unverified), the demo uses MariaDB for Windows instead.
- Configuration: a listener on 127.0.0.1:8080; routes that deny Drupal's private file patterns, serve `web/` as a share, and fall back to the PHP application with `index.php`, following the upstream Drupal how-to (https://unit.nginx.org/howto/drupal/, adapted to Windows paths); the PHP application with 4 processes.
- Acceptance:
  1. The web installer completes with the standard profile.
  2. 1,000 requests over `/`, `/node/1`, `/user/login`, `/admin/content` (logged in) and a missing path, at concurrency 4, return their expected status codes and no 5xx.
  3. A CSS aggregate is served as a static file with the right `Content-Type` and `Last-Modified`.
  4. S5: one opcache start time across the 4 workers; one hour with `request_limit` 50 logs no reattach failure.
  5. S6: Ctrl+C ends every FreeUnit process within 5 seconds; no orphan remains.
  6. The error log holds no `alert` line.

**M6. Test harness and CI.**

- Deliverables: the jobs of D16, added in stages: compile-only and C tests in M1, the pytest smoke subset in M5, the wide nightly subset in M7; the `conftest.py` import hygiene, the markers and the platform module (D14); job-object process control; psutil handle deltas.
- Gate: S2 by 2027-03-31; S7 by 2027-09-24. The manual Windows 11 checks of §2.4 run from M5 on, and include one suspend and resume of the virtual machine for 5 minutes with idle keepalive connections and an application starting: after resume, timers fire after their remaining time, not all at once.
- Size basis: Python 1,000 to 2,500 lines (`test/conftest.py` has 1,036 lines and `test/unit/http.py` 460 at 872bf041); YAML 200 to 400 lines (FreeUnit's macOS workflow has 83 lines).

**M7. ProcessSocketNotifications engine.**

- Deliverables: the `psn` engine with the probe and the fallback (D10 stage 2); listener ownership and hand-off; the listener comparison of D10 (hand-off, AFD poll on the shared listener, `AcceptEx`); a JDK-style microbenchmark with WSAPoll, ProcessSocketNotifications and wepoll; router CPU per request measured with the io_uring harness method; the accuracy of 1, 10 and 100 ms wait timeouts (research/windows-io-model.md question 9); a test that I/O issued by an engine survives the exit of the thread that created the engine (question 11); the results of E06 to E08 folded into the engine.
- Gate: S8; the listener decision written to decisions/; the Windows C tests and pytest subset pass with both engines.
- Size basis: the engine is 0.8 to 1.2 times the epoll engine, about 980 to 1,470 lines (research/windows-io-model.md §4); hand-off 200 to 500 lines; changes in `nxt_router.c` and `nxt_conn_accept.c`.

**M8. Tier 3 preview release.**

- Deliverables: the nightly packaging job; the zip layout; `SHA256SUMS`; the winget manifest; the Scoop bucket repository; signing if SignPath has approved; documentation (installation, the file layout of D21, the feature matrix, a Drupal guide for Windows, the SmartScreen note, the optional Defender exclusion of D22); release notes with the Tier 3 statement and the exit rule.
- Gate: S9.
- Size basis: a workflow, two manifests and documentation, 300 to 800 lines.

**After the preview, not scheduled.** Service mode and an MSI; Python WSGI; wasm; restricted-token workers and per-application job limits; Python ASGI, Node.js and external applications (D20); AF_UNIX listeners; OpenTelemetry; ARM64 once official ARM64 PHP builds exist.

## 7. Code rules

Every change that lands in FreeUnit for this project meets these rules. Paste the block into pull request templates and agent briefs.

```
FreeUnit code rules (all platforms)
 1. New code builds warning-free under ./configure --hardening=strict.
 2. No variable-length arrays.
 3. Use nxt_fallthrough; never a "Fall through." comment.
 4. Length sums and products from a peer, a client, shared memory or a file
    go through nxt_size_add() and nxt_size_mul() (src/nxt_checked.h). They
    report overflow instead of wrapping.
 5. Records parsed from untrusted bytes go through nxt_span_t (src/nxt_span.h):
    nxt_span_take(), nxt_span_copy(). No hand-rolled "p + n <= end" checks.
 6. A protocol field narrower than its value is checked before the copy and
    answered with a status, never truncated.
 7. C tests of helpers go into the table in src/test/nxt_tests.c
    (nxt_test_in_child(), NXT_TEST_CHECK()).
 8. Fault-injection hooks live under #if (NXT_TESTS). After
    make build/lib/libunit.a && make tests, run
    sh .github/scripts/check-test-hooks.sh; it must exit 0.
 9. Every new test fails on the pre-change code. Run that mutation and quote
    the failing value in the pull request.
10. Commit messages and pull request bodies use plain English, short
    sentences, exact names and numbers, and full URLs (never a bare #NNN).

Windows rules added by this plan
11. No int descriptor in a new cross-platform interface. Use nxt_socket_t,
    nxt_fd_t or a typed attachment, and compare with NXT_SOCKET_INVALID or
    NXT_FILE_INVALID, never with -1 or < 0.
12. Every Windows API call is checked for its documented failure value and
    error source: NULL or INVALID_HANDLE_VALUE as its page states;
    SOCKET_ERROR or INVALID_SOCKET with WSAGetLastError(); GetLastError()
    otherwise; WAIT_FAILED and WAIT_ABANDONED for waits; ERROR_ALREADY_EXISTS
    after creating a named object (a hard failure); every registrationResult
    of ProcessSocketNotifications; both ERROR_TIMEOUT and WAIT_TIMEOUT from
    its wait.
13. unitd asserts the platform floor at start: it reads the build number
    with RtlGetVersion, refuses builds below 17763, selects the WSAPoll
    engine below build 20348, and probes ProcessSocketNotifications with
    GetProcAddress before creating any engine.
14. Windows code lives in nxt_win32_*.c files and in the Windows branches of
    the existing abstraction files (process start, port transport, shared
    memory, engines, files, time). Other shared .c files gain no
    #ifdef _WIN32.
15. Windows code builds warning-free with clang-cl at the warning level set
    in M1, with /guard:cf and /GS, and links with /guard:cf, /DYNAMICBASE,
    /HIGHENTROPYVA, /NXCOMPAT and /CETCOMPAT where the probes accept them.
16. Handles are created non-inheritable. A child inherits only what
    PROC_THREAD_ATTRIBUTE_HANDLE_LIST names.
17. Only the more trusted process duplicates a handle into another process:
    main into its children, the router into workers. No worker holds
    PROCESS_DUP_HANDLE on another FreeUnit process.
18. Named kernel objects live in the instance's private namespace, which main
    creates with a DACL for the unitd user and SYSTEM
    (lpPrivateNamespaceAttributes). Each object gets the same explicit DACL,
    and its name ends in at least 128 random bits from BCryptGenRandom.
19. Shared memory holds offsets and indices, never absolute pointers. Nothing
    maps at a fixed address.
20. Every TCP listening socket sets SO_EXCLUSIVEADDRUSE. SO_REUSEADDR is not
    used on Windows. AF_UNIX listeners rely on the runtime directory's DACL.
21. Only a handle's final owner associates it with a completion port or a
    ProcessSocketNotifications registration. Overlapped I/O is issued only
    from the thread of the engine that owns the handle.
22. Paths, the command line and the environment reach Windows as UTF-16
    through the W functions, converted from UTF-8 with
    MultiByteToWideChar(CP_UTF8). No ANSI (A) functions. File paths longer
    than 259 characters use the \\?\ form. Every path-safety rule has a
    test that fails without it.
23. Renames and deletes retry on ERROR_SHARING_VIOLATION and
    ERROR_ACCESS_DENIED with the bounded backoff of D22. Files are opened with
    FILE_SHARE_READ | FILE_SHARE_WRITE | FILE_SHARE_DELETE unless exclusion is
    the point.
24. The engine's monotonic clock is QueryUnbiasedInterruptTime. Wall time is
    GetSystemTimePreciseAsFileTime.
```

## 8. Repository layout and contribution workflow

This repository holds the plan and the evidence. The port itself lands in https://github.com/freeunitorg/freeunit.

```
README.md                         what this repository is, status, how to read it
LICENSE                           Apache-2.0
docs/research-report.md           the synthesis of the research
docs/plan.md                      this plan
research/*.md                     the five research notes of 2026-10-07
experiments/Exx-<name>/           one Phase 0 experiment: program, build command, README
experiments/Exx-<name>/results/   raw output, Windows edition and build, compiler version
ci/windows.yml                    the workflow proposed for FreeUnit (D16)
.github/workflows/experiments.yml builds and runs the experiments on hosted runners
matrix/feature-matrix.md          the Tier 3 feature matrix, kept in step with releases
decisions/                        gate records: g0.md, then one file per later gate
```

Only README.md, LICENSE, docs/ and research/ exist today. The other paths are created by the first pull request that needs them.

Workflow:

1. Experiments. Open an issue that names the experiment ID. Send one pull request per experiment with the program, the build command and at least one results file. A result that contradicts this plan updates the affected decision (D-n) in the same pull request, with the reason.
2. Code. Each FreeUnit pull request names its milestone and decision IDs and links this plan by full URL. No Windows change merges with Linux or macOS CI red. Port, process and shared-memory changes need a second reviewer.
3. Research notes are not rewritten. A correction goes into a new note or into the report, with the evidence.
4. Only public material: no private host names, no local absolute paths, no credentials, no internal infrastructure details in results.
5. Everything in this repository is licensed Apache-2.0.

## 9. Risks

| # | Risk | Mitigation | Stop condition |
|---|---|---|---|
| R1 | The process and IPC work outgrows the estimate. | Gates per milestone; the preview does not need M7; isolation and service mode are deferred. | M2 and M3 together exceed twice their upper line estimate or twice their window: re-plan at the next gate. If the re-plan puts the preview after 2028-03-31, stop and keep WSL2 as the Windows path. |
| R2 | One maintainer. Envoy's Windows support ended on 2023-08-31 "due to a lack of resources", and Microsoft's Redis port stopped in 2016. | A named owner and a named reviewer at G0; the design written down here; the exit rule (§2.4). | No named owner for one release cycle: the next release drops the Windows artefacts. |
| R3 | Maintenance cost above precedent. Windows items are about 3% of nginx's trac tickets and about 2.6% of PostgreSQL's commits. | Windows code in platform files only (rule 14); one CI workflow; the feature matrix limits scope. | Windows-labelled issues and pull requests above 10% of the total for two quarters in a row after the preview: freeze Windows features and review the exit. |
| R4 | Shared changes regress Linux: descriptor types, the transport interface, attachments. | Linux CI required; the io_uring harness A/A check for port-layer pull requests (S3). | Any Linux regression beyond A/A noise blocks the pull request. No exceptions. |
| R5 | Undocumented or young interfaces: the peer-pid ioctl, ProcessSocketNotifications with documentation gaps and one runtime adopter (Pony). | Phase 0; the runtime probe and the WSAPoll fallback; the directory DACL as the identity backstop. | ProcessSocketNotifications loses or duplicates events in M7 stress tests: WSAPoll stays the only engine for Tier 3. |
| R6 | PHP runtime behaviour: opcache reattach, worker start latency, the VS18 move for PHP 8.6. | The resident prototype; the file cache fallback; spare workers; tracking php-windows-builder's `vs.json`. | Reattach failures above 1% of worker starts (E11): ship with `opcache.file_cache_only=1`. Worker start p95 above 2,000 ms (E12): no on-demand scaling on Windows. |
| R7 | Security. Upstream rated Windows-only advisories (mio IOCP named pipes, wasmtime on Windows) as not affecting Unit and still merged the fixes ([PR 1170](https://github.com/nginx/unit/pull/1170), merged 2024-03-11; [PR 1184](https://github.com/nginx/unit/pull/1184); [PR 1226](https://github.com/nginx/unit/pull/1226)); a native build removes that reasoning. Pipes, socket files, the TCP control socket and Windows path rules add local attack surface. | Triage rule updated for Windows; secure defaults (AF_UNIX control, DACLs, `SO_EXCLUSIVEADDRUSE`); path tests (D19). | A Windows-only security issue left unfixed for 30 days: pull the Windows artefacts until it is fixed. |
| R8 | Antivirus, SmartScreen and Smart App Control. | SignPath signing; Defender submissions per release; documentation. | Smart App Control blocks the bundled PHP on default Windows 11 installs and no signing route exists: say so on the download page; if most testers cannot run the build, pause releases. |
| R9 | Demand does not appear. | The Phase 0 poll; the adoption metric S11. | Poll below its pass line: no M1 (G0 fails). Fewer than 5 outside users by 2028-03-31: maintenance only, then an exit review. |
| R10 | The bundled PHP falls behind PHP security releases. | A scheduled job watches https://windows.php.net/downloads/releases/releases.json and rebuilds the zip. | A PHP security release not repackaged within 14 days, twice in a row: stop shipping the PHP module in the zip until the job works. |
| R11 | Conflicts with ongoing port-layer work in FreeUnit. | Narrow Windows files; the overlap check of M3. | None; this is a process rule. |

## 10. Open decisions for the maintainer

These decisions need the maintainer, not an experiment.

- **O1. Organisation and name.** Where this planning repository lives (under freeunitorg or a personal account) and what the Windows build is called in release notes, for example "FreeUnit for Windows (development preview)".
- **O2. Development only, for good?** Keep Windows at Tier 3 permanently, or allow promotion to Tier 2 under the promotion rule of §2.4.
- **O3. Windows 10 and Server 2019.** Keep the best-effort range of builds 17763 to 20347 (Windows 10 1809 and later, LTSC 2019 and 2021, Server 2019) while it is serviced: Windows 10 22H2 extended updates end on 2027-10-12 for consumers and 2028-10-10 for organisations, LTSC 2021 support ends on 2027-01-12, and Server 2019 and LTSC 2019 support ends on 2029-01-09. Or raise the runtime floor to 20348 at the preview.
- **O4. Signing.** SignPath Foundation (free; the publisher shown is SignPath Foundation), Certum open-source certificates (25 to 69 EUR and at most 459 days per certificate, as listed on https://shop.certum.eu/code-signing.html in research/toolchain-packaging-demand.md §4.2 and not re-verified; the certificate names a person), or Artifact Signing (USD 119.88 a year; it needs an organisation in an eligible country, or an individual resident in the United States or Canada).
- **O5. libunit ABI on Windows.** Accept `nxt_unit_fd_t` in the public header (D20). It leaves the Unix API and ABI unchanged and makes the Windows ABI differ from Unix.

## 11. Evidence index

- docs/research-report.md: the synthesis; answers to the five questions; the settlement of the four conflicts between the notes and three further corrections; upstream's position answered on the merits.
- research/history-upstream-and-fork.md: upstream's statements on Windows with dates and quotes; demand in the nginx/unit and FreeUnit trackers; the dead Windows remnants in the tree; nginx's Windows history and its unfinished IOCP module.
- research/portability-audit.md: the Unix dependencies of the core and libunit at 872bf041 with grades; 13 architectural assumptions; the five hardest problems; the size estimate of 10,000 to 18,500 new lines.
- research/process-and-ipc-design.md: the verified process and IPC map; Windows designs D1 to D13 for spawning, the prototype, the port channel, wake-ups, descriptor passing, shared memory, supervision, signals and modules; the precedents of PostgreSQL, Apache, nginx, libuv and opcache; experiments Q1 to Q16.
- research/windows-io-model.md: the engine contract and the io_uring lessons; Windows I/O interfaces with minimum builds; the precedents of AFD poll, Pony and completion-native designs; options O1 to O6; the platform floor; the staged engine path; experiments 1 to 13.
- research/toolchain-packaging-demand.md: the `__vectorcall` evidence and the compiler choice; official PHP builds and the embed library; the build system; CI runners and costs; the test harness; packaging and signing; demand data; precedents; maintenance cost and support tiers.
- Two independent reviews on 2026-10-07 rechecked code citations, counts and primary sources against the notes; their findings were verified before they were applied.
- Checks made for this plan on 2026-10-07: the source at 872bf041 (`src/nxt_socketpair.c`, `auto/sockets`, `auto/feature`, `src/nxt_atomic.h`, `src/nxt_clang.h`, `src/nxt_php_sapi.c`, `src/nxt_app_queue.h`, `src/nxt_router.c`, `src/nxt_unit.h`, `src/nxt_unit.c`, `src/nxt_controller.c`, `src/nxt_main_process.c`, `src/nxt_service.c`, `src/nxt_unit_sptr.h`, `src/nxt_port_memory_int.h`, `src/nxt_port_socket.c`, `src/nxt_file.h`, `test/conftest.py`, `test/unit/http.py`, `auto/os/test`); php-src php-8.5.11 (`Zend/zend_stream.h`, `Zend/zend_execute_API.c` lines 1579, 1618 and 1622); Microsoft Learn pages for ProcessSocketNotifications, WSAPoll, SO_EXCLUSIVEADDRUSE, Object Namespaces and QueryUnbiasedInterruptTime; php-src php-8.5.11 `main/php_ini.c`; the import library of the PHP 8.5.11 non-thread-safe development package; PostgreSQL's `pg-ci.yml`; FreeUnit's commit history since 2026-04-03 for the velocity basis; the GitHub API for nginx/unit issue 604, FrankenPHP v1.12.0 and pull request 2119, Pony 0.66.0, CPython pull request 137420 and FreeUnit's star and fork counts; the io_uring results at commit 6ee2a59d.
