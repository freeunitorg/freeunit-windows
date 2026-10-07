# Native Windows FreeUnit: toolchain, build, CI, packaging, demand and precedents

Research notes for the freeunit-windows feasibility plan, dated 2026-10-07. FreeUnit source facts are cited as `path:line@872bf041`, meaning revision 872bf04170756e919211799ec5574000d628d521 of https://github.com/freeunitorg/freeunit. Every other fact ends with its source link. Section 0 summarises and lists where the briefing was wrong. Sections 1 to 7 follow the key questions: 1 toolchain and PHP embedding, 2 build system and dependencies, 3 CI and test harness, 4 packaging and signing, 5 demand, 6 precedents, 7 maintenance cost, support tiers and open questions. Each numbered section has a Takeaway, Cited Findings, Inferences (with a confidence label) and Gaps. Demand numbers carry a population label: [ALL DEVS], [PHP DEVS], [DDEV USERS] or [DOWNLOADS].

## 0. Summary and corrections to the briefing

### Takeaway
A native Windows FreeUnit for local PHP development can be built with current, free tools: an MSVC-ABI compiler (clang-cl from Visual Studio 2022 or 2026), the official PHP zips and devel packs without rebuilding PHP, vcpkg for OpenSSL, PCRE2, zlib, brotli and zstd, the existing configure script under the MSYS2 shell, free GitHub-hosted Windows runners, and a zip published through winget and an own Scoop bucket. Demand is real, but its key segment is unmeasured: 54% of professional Stack Overflow respondents who have worked with PHP use Windows at work and 35% use Windows without WSL (the author's tabulation of the public 2025 dataset, section 5.1), yet no source measures how many are blocked from WSL2 or Docker. The precedents warn that Windows ports remain limited unless the I/O and process model is designed for Windows, and that they end when their maintainers leave. FrankenPHP shipped the closest working precedent in March 2026.

### Cited Findings
Corrections to the briefing (details in the numbered sections):
- FrankenPHP is no longer WSL or Docker only. Native Windows support shipped in v1.12.0 (2026-03-06) and links the official MSVC-built TS PHP (section 6.1). Sources: [Windows support for FrankenPHP: it's finally alive](https://dunglas.dev/2026/03/windows-support-for-frankenphp-its-finally-alive/), [php/frankenphp#2119](https://github.com/php/frankenphp/pull/2119)
- php8embed.lib ships at the root of the official binary zips (8.4.26 TS, 8.5.11 TS and NTS), not in the devel pack. FreeUnit does not need it: src/nxt_php_sapi.c@872bf041 calls no `php_embed_*` function, and the configure probe only calls `php_module_startup()` (auto/modules/php:142-160@872bf041). The devel pack's import library and headers are enough (section 1.2). Source: [php-8.5.11-Win32-vs17-x64.zip](https://downloads.php.net/~windows/releases/php-8.5.11-Win32-vs17-x64.zip)
- The Visual Studio map in the briefing is right (vs16 for PHP 8.0 to 8.3, vs17 for 8.4 and 8.5). PHP 8.6, 8.7 and master move to vs18 (Visual Studio 2026), and 8.6.0RC3 vs18 QA builds exist (section 1.1). Sources: [php-windows-builder vs.json](https://github.com/php/php-windows-builder/blob/master/php/BuildPhp/config/vs.json), [QA listing](https://downloads.php.net/~windows/qa/)
- No PHP branch has an official ARM64 Windows build (section 1.1). Source: [PHP for Windows releases listing](https://windows.php.net/downloads/releases/)
- The deciding constraint on the PHP module compiler is the calling convention, not only CRT mixing: 340 of 6,116 exports of the 8.5.11 php8ts.lib carry `__vectorcall` names such as `_emalloc@@8`. PHP's headers request that convention only when `_MSC_VER` is defined, and GCC has no vectorcall attribute, so MinGW gcc cannot link them (section 1.2). Sources: [php-8.5.11-Win32-vs17-x64.zip](https://downloads.php.net/~windows/releases/php-8.5.11-Win32-vs17-x64.zip), [GCC x86 function attributes](https://gcc.gnu.org/onlinedocs/gcc/x86-Attributes.html)
- LTCG does not constrain an embedder: the official php8ts.lib and php8embed.lib contain no /GL objects (section 1.2). Source: [php-8.5.11-Win32-vs17-x64.zip](https://downloads.php.net/~windows/releases/php-8.5.11-Win32-vs17-x64.zip)
- lighttpd is not Cygwin only: an experimental native Windows build exists since 1.4.70 (2023-05-10) (section 6.1). Source: [lighttpd 1.4.70](https://www.lighttpd.net/2023/5/10/1.4.70/)
- Envoy is not maintained on Windows: official support ended on 2023-08-31 "due to a lack of resources" (section 6.1). Source: [Envoy: requirements to run on Windows](https://www.envoyproxy.io/docs/envoy/latest/faq/windows/win_requirements)
- php-cgi on Windows does honour PHP_FCGI_CHILDREN, as a static pool of at most 64 processes (section 5.7). Source: [php-src sapi/cgi/cgi_main.c at 20f4c4d6](https://github.com/php/php-src/blob/20f4c4d605203b95cf01e76e306c25bdff6678b2/sapi/cgi/cgi_main.c#L2092-L2210)
- `windows-11-arm` left preview: generally available for public repositories on 2025-08-07, and now also listed for private repositories. `windows-latest` is Windows Server 2025 with Visual Studio 2026 since June 2026 (section 3.1). Sources: [GitHub changelog 2025-08-07](https://github.blog/changelog/2025-08-07-arm64-hosted-runners-for-public-repositories-are-now-generally-available/), [GitHub changelog 2026-05-14](https://github.blog/changelog/2026-05-14-github-actions-upcoming-image-migrations)
- vcpkg removed its `x-gha` cache provider on 2025-04-29, not in 2024 (sections 2.3 and 3.3). Source: [microsoft/vcpkg-tool PR 1662](https://github.com/microsoft/vcpkg-tool/pull/1662)
- `php/setup-php-sdk` is archived; its successor is `php/php-windows-builder` (section 3.3). Source: [php/setup-php-sdk](https://github.com/php/setup-php-sdk)
- PostgreSQL left Cirrus CI on 2026-06-04 and runs its Windows jobs on GitHub Actions (section 3.4). Source: [postgres/postgres commit 68c8a365d4](https://github.com/postgres/postgres/commit/68c8a365d4dcefd7ec60f7a99a7ab7b557c36357)
- WiX is at v7.0.0 (2026-04-06). Its Open Source Maintenance Fee applies only to organizations with more than USD 10,000 annual revenue (section 4.1). Sources: [WiX Toolset releases](https://github.com/wixtoolset/wix/releases), [FireGiant docs: Open Source Maintenance Fee](https://docs.firegiant.com/wix/osmf/)
- Azure Trusted Signing is now called Artifact Signing. Individuals qualify only in the United States or Canada; the current quickstart states no minimum years of business history (section 4.2). Source: [Microsoft Learn: Quickstart: Set up Artifact Signing](https://learn.microsoft.com/en-us/azure/artifact-signing/quickstart)
- Consumer Windows 10 ESU was extended to 2027-10-12 (section 4.7). Source: [Windows Experience Blog, post of 2025-06-24 with editor's notes](https://blogs.windows.com/windowsexperience/2025/06/24/stay-secure-with-windows-11-copilot-pcs-and-windows-365-before-support-ends-for-windows-10/)
- Smart App Control can now be re-enabled without a clean installation (section 4.3). Source: [Microsoft Support: Smart App Control FAQ](https://support.microsoft.com/en-us/topic/what-is-smart-app-control-285ea03d-fa88-4d56-882e-6698afdb7003)
- Laragon did not simply turn paid in 2024: the line that needs a licence for commercial use started with v7.0.6 on 2024-12-16, and non-commercial use stays free (section 5.4). Source: [Laragon pricing](https://laragon.org/pricing)
- The pytest suite fails at import on Windows, before any control-socket question: `import fcntl` at test/conftest.py:2@872bf041, top-level `pwd` and `grp` imports in four test modules, and a `socket.AF_UNIX` lookup on every request in test/unit/http.py:64@872bf041 (section 3.5). Source: [Python docs: fcntl](https://docs.python.org/3/library/fcntl.html)
- The workflows at 872bf041 also use `macos-latest` (.github/workflows/unitctl.yml:55@872bf041) and `ubuntu-24.04-arm` (.github/workflows/release-docker.yml:103@872bf041), not only ubuntu and `macos-15`; none uses Windows.

### Inferences
- Recommendation: build a Tier 3 "development use" Windows x64 target for PHP first (section 7.1), with the toolchain in 1.0, the CI plan in 3.7 and the packaging plan in 4.8. Confidence: medium.
- Toolchain, CI, packaging and dependencies are not the hard part. The hard part is the core's process and IPC model: fork, socketpairs with descriptor passing and shared memory, which configure probes for (auto/sockets and auto/shmem at 872bf041, section 2.7) and which Winsock AF_UNIX cannot carry (stream sockets only, no socketpair, no descriptor passing; section 3.5). That decision belongs to the architecture chapter and gates everything in these notes. Confidence: high.
- The second risk is people. Envoy's Windows support and the Microsoft Redis port ended when their Windows maintainers left (section 6.1). Name the maintainers before committing. Confidence: medium-high.

### Gaps
Three experiments decide most of the plan (the full list is in 7.2):
- Run configure under the MSYS2 shell with `CC=clang-cl` on `windows-2025` and keep the log: count the failing probes and link lines.
- Link and load a minimal PHP SAPI module against the official php8ts.lib with clang-cl, serve one request, and confirm that MinGW gcc fails on the `@@N` names.
- Ask DDEV and Drupal community channels how many Windows developers have WSL2 or Docker blocked at work.

## 1.0 Toolchain and build-system recommendation (synthesis of 1.1 to 2.8)

### Takeaway
Use one MSVC-ABI compiler family: clang-cl with /MD (the dynamic Universal CRT). Build the core and libunit with the current Visual Studio (2026 on the `windows-2025` runner image). Build the PHP 8.4 and 8.5 module with the toolset PHP itself uses (v14.44, Visual Studio 2022 17.14) and the PHP 8.6 module with Visual Studio 2026; v14 binaries from these versions are compatible. Drive it with the existing configure script and auto/ under the MSYS2 shell and GNU make, take the libraries from vcpkg, and link the official PHP import library and headers (php8ts.lib for TS, php8.lib for NTS) without rebuilding PHP. Ship x64 only at first. MinGW gcc cannot build the PHP module, and plain cl.exe would need shims for FreeUnit's GCC-specific code.

### Cited Findings
- The PHP module must be compiled in MSVC mode. 340 of 6,116 exports of the 8.5.11 php8ts.lib are `__vectorcall` names (`name@@N`), and Zend/zend_portability.h defines ZEND_FASTCALL as `__vectorcall` only when `_MSC_VER` is defined (section 1.2). GCC's x86 attribute list has no vectorcall. Clang targeting MinGW accepts `__attribute__((vectorcall))` and emits the same `@@N` names (tested 2026-10-07 with clang 21.1.8 for x86_64-w64-mingw32), but only after patching ZEND_FASTCALL, which PHP does not support. Sources: [php-8.5.11-Win32-vs17-x64.zip](https://downloads.php.net/~windows/releases/php-8.5.11-Win32-vs17-x64.zip), [php-devel-pack-8.5.11-Win32-vs17-x64.zip](https://downloads.php.net/~windows/releases/php-devel-pack-8.5.11-Win32-vs17-x64.zip), [php-src PHP-8.5 Zend/zend_portability.h](https://github.com/php/php-src/blob/PHP-8.5/Zend/zend_portability.h), [GCC x86 function attributes](https://gcc.gnu.org/onlinedocs/gcc/x86-Attributes.html), [Clang attribute reference](https://clang.llvm.org/docs/AttributeReference.html)
- FreeUnit code that cl.exe rejects and clang-cl accepts: `nxt_inline` and `nxt_noinline` use `__attribute__` unconditionally (src/nxt_clang.h:11-12@872bf041); atomics exist only as GCC `__sync` builtins under `NXT_HAVE_GCC_ATOMIC`, with no other branch (src/nxt_atomic.h:17-89@872bf041); `__builtin_ffs` and `__builtin_ffsll` are called without a fallback (src/nxt_mp.c:129@872bf041, src/nxt_conf_validation.c:1786@872bf041, src/nxt_port_memory_int.h:267@872bf041). Already portable: `__builtin_clz` at src/nxt_mp.c:166@872bf041 has a table fallback (src/nxt_mp.c:163-172@872bf041), checked size arithmetic has portable functions (src/nxt_checked.h:44-66@872bf041), and the one `__typeof__` (src/nxt_http_chunk_parse.c:23@872bf041) is accepted by cl.exe since Visual Studio 17.9 ([Microsoft Learn: typeof, __typeof__](https://learn.microsoft.com/en-us/cpp/c-language/typeof-c)). Method: git grep over the 357 C sources and headers under src/ at 872bf041, outside the Rust crate directories src/otel and src/wasm-wasi-component.
- Clang "attempts to be ABI-compatible, meaning that Clang-compiled code should be able to link against MSVC-compiled code successfully", and Visual Studio ships it in clang-cl mode. Sources: [Clang MSVC compatibility](https://clang.llvm.org/docs/MSVCCompatibility.html), [Microsoft Learn: Clang/LLVM support in Visual Studio projects](https://learn.microsoft.com/en-us/cpp/build/clang-support-msbuild)
- Comparable projects chose clang in MSVC mode: "As of Node.js 24.0.0, ClangCL is required to compile on Windows" ([Node.js BUILDING.md](https://github.com/nodejs/node/blob/main/BUILDING.md)); FrankenPHP links the official PHP with Visual Studio's clang and lld-link after its MinGW build crashed on CRT mismatches ([php/frankenphp#2119](https://github.com/php/frankenphp/pull/2119), [release blog](https://dunglas.dev/2026/03/windows-support-for-frankenphp-its-finally-alive/)).
- A `FILE *` crosses the module/PHP boundary (src/nxt_php_sapi.c:1290@872bf041), so the module and php8ts.dll must share one CRT; the official PHP uses the dynamic UCRT and VCRUNTIME140.dll (section 1.4). The same line's mode string "re" is rejected by the UCRT mode parser. Sources: [Microsoft Learn: Potential errors passing CRT objects across DLL boundaries](https://learn.microsoft.com/en-us/cpp/c-runtime-library/potential-errors-passing-crt-objects-across-dll-boundaries), [UCRT corecrt_internal_stdio.h](https://doxygen.reactos.org/de/dec/corecrt__internal__stdio_8h_source.html)
- The PHP module runs one request loop per process: one `nxt_unit_init()` and one `nxt_unit_run()` (src/nxt_php_sapi.c:530-538@872bf041). ZTS is optional in configure (auto/modules/php:169-183@872bf041) and only adds a `php_tsrm_startup()` call (src/nxt_php_sapi.c:407-415@872bf041).
- windows.php.net: "If you want to use PHP as FastCGI with IIS, use the Non-Thread Safe (NTS) builds of PHP, or if you want to use the Apache HTTP Server, use the Thread Safe (TS) builds of PHP." Source: [PHP downloads for Windows](https://www.php.net/downloads.php?os=windows)
- Microsoft v14 toolsets stay binary compatible through Visual Studio 2026: "Any apps built by MSVC Build Tools v14.* available in Visual Studio 2017, 2019, 2022, or 2026 can use the latest Visual C++ v14 Redistributable." Source: [Microsoft Learn: Latest supported Visual C++ Redistributable downloads](https://learn.microsoft.com/en-us/cpp/windows/latest-supported-vc-redist?view=msvc-170)
- configure already expects this model: the MINGW branch selects `cl` (auto/os/test:77-88@872bf041). nginx builds the same way with about 270 lines of MSVC-specific shell and makefiles, but its MSVC path has no dynamic modules (section 2.1). Source: [nginx auto/os/win32](https://github.com/nginx/nginx/blob/master/auto/os/win32)
- PostgreSQL (17.0, 2024-09-26) and curl (8.17.0, 2025-11-05) both deleted their Windows-only build systems (section 2.2). Sources: [PostgreSQL 17.0 release notes](https://www.postgresql.org/docs/release/17.0/), [curl 8.17.0 changes](https://curl.se/ch/8.17.0.html)
- MSYS2 deprecated the msvcrt-based MINGW64 environment on 2026-03-15 (section 1.4). Source: [MSYS2 news](https://www.msys2.org/news/)
- Every needed library has a vcpkg port with x64-windows and arm64-windows triplets; Rust's x86_64-pc-windows-msvc and aarch64-pc-windows-msvc targets are Tier 1; njs has no Windows support (sections 2.3, 2.4, 2.6). Sources: [vcpkg ports](https://github.com/microsoft/vcpkg/tree/master/ports), [rustc platform support](https://doc.rust-lang.org/rustc/platform-support.html), [nginx/njs#629](https://github.com/nginx/njs/issues/629)

### Inferences

#### Recommendation

| Decision | Choice | Main evidence | Confidence |
|---|---|---|---|
| Compiler for the core, libunit and the PHP module | clang-cl (the Visual Studio component "C++ Clang tools for Windows"), with lld-link or link.exe; Visual Studio 2026 for the core, and for the module the toolset of the PHP build it links (row "Toolset per PHP branch") | `__vectorcall` exports; GCC-only code in src/nxt_clang.h and src/nxt_atomic.h; Node.js and FrankenPHP chose the same | medium-high |
| Fallback compiler | cl.exe, after a small shim: `__forceinline` for `nxt_inline`, Interlocked intrinsics for nxt_atomic.h, `_BitScanForward` and `_BitScanForward64` for the `ffs` builtins | grep above | medium |
| Not for shipped binaries | MinGW-w64 gcc; clang in GNU mode (MinGW target) | gcc has no vectorcall support; GNU-mode clang would need a patched ZEND_FASTCALL, which PHP does not support; FrankenPHP's MinGW build failed on CRT mismatches; MINGW64 deprecated | high |
| CRT | /MD (dynamic UCRT); VC++ v14 redistributable as a prerequisite, as PHP itself requires | `FILE *` crosses into PHP; one CRT per process | high |
| PHP inputs | official binary zip (runtime DLLs, ext/) plus the matching devel pack (headers, php8ts.lib or php8.lib); php8embed.lib not used; no PHP rebuild for x64 | section 1.2 | high |
| PHP variant | Build and test both TS and NTS. The default follows the Windows process model chosen by the architecture chapter: separate worker processes can use NTS; worker threads in one process (the Apache mpm_winnt and FrankenPHP model) need TS | src/nxt_php_sapi.c:530-538@872bf041; windows.php.net TS/NTS guidance | medium |
| Toolset per PHP branch | v14.44 (VS 2022 17.14) for PHP 8.4 and 8.5; v14.5x (VS 2026) for 8.6; the core may use the newer toolset because v14 binaries stay compatible | php-windows-builder VS map; Microsoft v14 compatibility statement | medium |
| Script open | On Windows replace `fopen(filename, "re")` and `zend_stream_init_fp()` with `zend_stream_init_filename()`, so no `FILE *` crosses and PHP handles wide paths | section 1.4 | medium |
| Build system | Keep configure and auto/ under the MSYS2 shell with GNU make; add an MSVC-ABI compiler case to auto/cc/test, a Windows case to auto/os/conf, `.exe` handling in auto/feature, DLL module linking against an import library of unitd.exe or a core DLL, and the `otel.lib` name for the Rust static library | sections 2.1, 2.7, 2.8 | medium |
| Dependencies | vcpkg manifest with a pinned baseline, triplet x64-windows (or x64-windows-static-md), OpenSSL pinned to the 3.5 LTS line | section 2.3 | medium |
| Rust | x86_64-pc-windows-msvc for src/otel; unitctl needs hyperlocal behind `cfg(unix)` and the TCP control address on Windows | section 2.4 | medium |
| Out of the first milestone | ARM64 (no official PHP), njs (no Windows support), the wasm modules, the Ruby and Perl modules (their Windows runtimes are MinGW UCRT builds), release builds cross-compiled from Linux (Microsoft toolchain licence, not redistributable) | sections 1.1, 1.5, 1.6, 2.5, 2.6 | high |

- The Linux mingw-w64 cross-compile job proposed in 3.7 is a cheap early warning for portability mistakes, but it builds a different ABI from the one shipped. Keep it optional and never treat it as evidence for the PHP module. Confidence: medium.
- Writing order: the compiler case in auto/cc/test and the header shims come first, because nothing compiles without them; the module DLL link is the riskiest build step, because nginx's MSVC path never needed it. Confidence: medium.

### Gaps
- Nothing has been compiled or linked on Windows yet. One CI job settles the core questions: run configure with `CC=clang-cl` under the MSYS2 shell on `windows-2025`, link a 20-line program that calls `php_module_startup()` and `_emalloc()` against php8ts.lib with clang-cl, cl.exe and UCRT64 gcc (gcc is expected to fail on the `@@N` names), and call `fopen("x.php", "re")` with and without PHP's invalid parameter handler.
- The licence terms for running the Microsoft Build Tools on Linux CI through msvc-wine or xwin were not read (https://go.microsoft.com/fwlink/?LinkId=2086102).
- Whether clang-cl from Visual Studio 2022 17.14 accepts FreeUnit's `--hardening=strict` flag set was not checked; auto/cc/hardening was not read for MSVC-mode equivalents.

## 1.1 Official Windows PHP builds: Visual Studio version, TS/NTS, architectures

### Takeaway
The briefing's map is correct: PHP 8.0 to 8.3 are built with VS16 (Visual Studio 2019 toolset), 8.4 and 8.5 with VS17 (Visual Studio 2022), and PHP 8.6 moves to VS18 (Visual Studio 2026), whose 8.6.0RC3 QA builds already exist. Every branch ships TS and NTS for x64 and x86 only; no official ARM64 build exists for any branch.

### Cited Findings
- Current builds in the releases listing on 2026-10-07 (dates are the listing's file times) - [PHP for Windows releases listing](https://windows.php.net/downloads/releases/); same data in [releases.json](https://windows.php.net/downloads/releases/releases.json):

  | Branch | Latest build | VS tag | Variants | Arch | File date |
  |---|---|---|---|---|---|
  | 8.1 | 8.1.34 | vs16 | TS, NTS | x64, x86 | 2025-12-16 |
  | 8.2 | 8.2.34 | vs16 | TS, NTS | x64, x86 | 2026-09-22 |
  | 8.3 | 8.3.35 | vs16 | TS, NTS | x64, x86 | 2026-09-22 |
  | 8.4 | 8.4.26 | vs17 | TS, NTS | x64, x86 | 2026-09-22 |
  | 8.5 | 8.5.11 | vs17 | TS, NTS | x64, x86 | 2026-09-22 |
  | 8.6 (QA) | 8.6.0RC3 | vs18 | TS, NTS | x64, x86 | QA only |

- releases.json has keys only for branches 7.4, 8.0, 8.1, 8.2, 8.3, 8.4 and 8.5, each with exactly four builds (ts/nts times x64/x86) - [releases.json](https://windows.php.net/downloads/releases/releases.json).
- The QA directory holds php-8.6.0RC3-Win32-vs18-x64.zip, php-8.6.0RC3-nts-Win32-vs18-x64.zip and x86 twins with vs18 devel and debug packs, plus 8.5.12RC1 and 8.4.27RC1 (vs17) and stale 8.3.34-dev, 8.2.27RC1 and 8.1.27RC1 (vs16) entries - [QA listing](https://downloads.php.net/~windows/qa/).
- The strings "arm64" and "aarch64" occur 0 times in the releases, archives and QA listings; "vs18" occurs only in the QA listing (24 times) - [releases](https://windows.php.net/downloads/releases/), [archives](https://windows.php.net/downloads/releases/archives/), [QA](https://downloads.php.net/~windows/qa/).
- The old host now redirects: https://windows.php.net/downloads/releases/ answers 302 to https://downloads.php.net/~windows/releases/, and https://windows.php.net/ answers 302 to the php.net download page - [PHP downloads for Windows](https://www.php.net/downloads.php?os=windows). https://windows.php.net/qa/ and https://windows.php.net/snapshots/ return the same php.net pre-release page (29,864 bytes), which links the 8.6.0RC3 vs18 zips.
- The php.net Windows page still says: "The builds below are built using Visual Studio 2019 (VS16) or Visual Studio 2022 (VS17) compiler. They require the Visual C++ Redistributable for Visual Studio 2015-2022 x64 or x86 installed." - [PHP downloads for Windows](https://www.php.net/downloads.php?os=windows).
- The builder's version map: "8.0" to "8.3": "vs16", "8.4" and "8.5": "vs17", "8.6", "8.7" and "master": "vs18"; vs17 means toolset 14.30 to 14.49, vs18 means 14.50 and later - [php-windows-builder vs.json](https://github.com/php/php-windows-builder/blob/master/php/BuildPhp/config/vs.json).
- The PHP build workflow matrix is `arch: [x64, x86]`; it runs on `windows-2025-vs2026` for 8.6, 8.7 and master, else on `windows-2022` - [php-windows-builder php.yml](https://github.com/php/php-windows-builder/blob/master/.github/workflows/php.yml).
- Both 8.4.26 and 8.5.11 devel packs define PHP_BUILD_COMPILER "Visual C++ 2022", PHP_COMPILER_ID "VS17", PHP_LINKER_MAJOR 14, PHP_LINKER_MINOR 44 in include/main/config.w32.h - [php-devel-pack-8.5.11-Win32-vs17-x64.zip](https://downloads.php.net/~windows/releases/php-devel-pack-8.5.11-Win32-vs17-x64.zip).
- PHP Foundation, 2026-06-26: "Historically, PHP for Windows has shipped x86 and x64 builds" and "Dropping x86 and investing in ARM64 may make more sense for future versions." - [Maintaining PHP Build infrastructure for Windows](https://thephp.foundation/blog/2026/06/26/maintaining-php-build-infrastructure-for-windows-tooling-for-builds-and-security-updates/).

### Inferences
- Build and test FreeUnit against PHP 8.4 and 8.5 with Visual Studio 2022 17.14 (toolset 14.44) or newer, and add Visual Studio 2026 when PHP 8.6 ships. Confidence: high.
- Windows on ARM64 is out of scope for the first target: there is no official PHP to embed. Confidence: high.
- 8.1.34 is the last 8.1 build in the listing; treat 8.1 as unsupported. Confidence: medium (EOL date not checked here).

### Gaps
- Table dates are file times, not announcement dates. Cross-check with https://www.php.net/releases/ if a date matters.
- The 8.1 end-of-life date was not read from https://www.php.net/supported-versions.php.

## 1.2 Embed SAPI availability, devel pack contents, PGO and LTCG

### Takeaway
php8embed.lib ships in every 8.4 and 8.5 binary zip, at the zip root; the devel pack does not contain it. It holds one plain x64 COFF object (php_embed.obj, /MD, Control Flow Guard), a resource object and a full import library for php8ts.dll; no member is a /GL (LTCG) object. FreeUnit's PHP module does not need php8embed.lib: it calls no php_embed_* symbol, so the import library php8ts.lib (TS) or php8.lib (NTS) from the devel pack plus its headers are enough. No PHP rebuild is needed for x64.

### Cited Findings
- Method: all PHP archives were downloaded fresh on 2026-10-07, checked against the published sha256 list and inspected without running anything (unzip -l, objdump -p and a small Python parser for the ar, COFF and PE structures). Source: [sha256sum.txt](https://windows.php.net/downloads/releases/sha256sum.txt)
- Downloads, all six sha256 values matched the published list - [sha256sum.txt](https://windows.php.net/downloads/releases/sha256sum.txt):

  | File | Bytes | sha256 |
  |---|---|---|
  | php-8.4.26-Win32-vs17-x64.zip | 35,138,257 | 6e56f0e932e92bfdce208d3a7e6068f6e2f7a19fc2922a5f856fb085f61673f3 |
  | php-devel-pack-8.4.26-Win32-vs17-x64.zip | 1,422,785 | 34de521ec706455e578b8a60343740056bb0532d3e1b82f440225354fd06e7b7 |
  | php-8.5.11-Win32-vs17-x64.zip | 36,180,745 | c83d5a1e0d760fb026ce695d8a70a73a4c81ac224b3e5c425406b23a7d53e1de |
  | php-devel-pack-8.5.11-Win32-vs17-x64.zip | 1,746,011 | 43f610861ced7b943437fe2808cfd720fde3341f461b1d441ea18bcd49ff711d |
  | php-8.5.11-nts-Win32-vs17-x64.zip | 36,048,897 | 0ea96e0d2b9b737a6036f05cf4e95c49313faa6d0f27bd97edb2742503f0c043 |
  | php-devel-pack-8.5.11-nts-Win32-vs17-x64.zip | 1,744,096 | d85e734238d45d02ddc35b5bca4ab47ad12e005fad970c792c520683ae3b0f33 |

- Exact paths and sizes from the full `unzip -l` listings - [php-8.4.26-Win32-vs17-x64.zip](https://downloads.php.net/~windows/releases/php-8.4.26-Win32-vs17-x64.zip), [php-8.5.11-Win32-vs17-x64.zip](https://downloads.php.net/~windows/releases/php-8.5.11-Win32-vs17-x64.zip), [php-8.5.11-nts-Win32-vs17-x64.zip](https://downloads.php.net/~windows/releases/php-8.5.11-nts-Win32-vs17-x64.zip) and the matching devel packs:

  | Item | 8.4.26 TS | 8.5.11 TS | 8.5.11 NTS |
  |---|---|---|---|
  | `php8embed.lib` (binary zip root) | 1,021,812 | 1,563,100 | 1,542,216 |
  | `dev/php8ts.lib` or `dev/php8.lib` (binary zip) | 949,690 | 1,489,082 | 1,469,030 (php8.lib) |
  | `php8ts.dll` or `php8.dll` (binary zip) | 11,745,280 | 14,140,928 | 13,688,832 (php8.dll) |
  | devel `lib/php8ts.lib` or `lib/php8.lib` | 949,690 | 1,489,082 | 1,469,030 |
  | devel `include/` entries (files and directories) | 406 | 430 | 430 |
  | devel `include/sapi/embed/php_embed.h` | 1,796 | 1,796 | 1,796 |
  | devel `phpize.bat` | 288 | 320 | 320 |
  | devel `script/confutils.js` | 104,459 | 105,280 | 105,280 |
  | devel `.lib` files in `lib/` | 38 | 37 | 37 |
  | entries in binary zip / devel pack (files and directories) | 86 / 460 | 85 / 483 | 84 / 483 |

- Devel pack paths are relative to one top directory per pack: `php-8.4.26-devel-vs17-x64/` and `php-8.5.11-devel-vs17-x64/` (the TS and NTS 8.5.11 packs use the same name) - [php-devel-pack-8.4.26-Win32-vs17-x64.zip](https://downloads.php.net/~windows/releases/php-devel-pack-8.4.26-Win32-vs17-x64.zip).
- The devel packs also hold script/phpize.js, script/config.phpize.js, script/config.w32.phpize.in, script/Makefile.phpize, script/ext_deps.js, build/gen_stub.php, build/template.rc and build/default.manifest. No devel pack contains php8embed.lib, config.nice.bat or a Makefile other than Makefile.phpize - [php-devel-pack-8.5.11-Win32-vs17-x64.zip](https://downloads.php.net/~windows/releases/php-devel-pack-8.5.11-Win32-vs17-x64.zip).
- sapi/embed/config.w32 is the same in PHP-8.4 and PHP-8.5: `ARG_ENABLE('embed', 'Embedded SAPI library', 'no');`, `var PHP_EMBED_PGO = false;`, `SAPI('embed', 'php_embed.c', 'php' + PHP_VERSION + 'embed.lib', '/DZEND_ENABLE_STATIC_TSRMLS_CACHE=1');` - [PHP-8.4 config.w32](https://github.com/php/php-src/blob/PHP-8.4/sapi/embed/config.w32), [PHP-8.5 config.w32](https://github.com/php/php-src/blob/PHP-8.5/sapi/embed/config.w32).
- On master (8.6) the embed target also compiles the CLI sources (php_cli.c, php_http_parser.c, php_cli_server.c, ps_title.c, php_cli_process_title.c) and adds ws2_32.lib and shell32.lib - [master config.w32](https://github.com/php/php-src/blob/master/sapi/embed/config.w32).
- The official configure command, from CONFIGURE_COMMAND in include/main/config.w32.h, is the same for 8.4.26 and 8.5.11 TS: `configure.js "--enable-snapshot-build" "--enable-debug-pack" "--enable-object-out-dir=../obj/" "--enable-com-dotnet=shared" "--without-analyzer" "--with-pgo"`; NTS adds `"--disable-zts"`. There is no explicit --enable-embed; the snapshot build turns it on - [php-devel-pack-8.4.26-Win32-vs17-x64.zip](https://downloads.php.net/~windows/releases/php-devel-pack-8.4.26-Win32-vs17-x64.zip).
- The builder's first pass uses the same options with `"--enable-pgi"` (instrumented build) - [config.nts.bat for vs17 x64](https://github.com/php/php-windows-builder/blob/master/php/BuildPhp/config/vs17/x64/config.nts.bat).
- confutils.js lines 1175-1263 (copy in the 8.5.11 devel pack): `is_pgo_desired()` returns the value of `PHP_<NAME>_PGO` when defined; with --with-pgo each PGO-enabled SAPI gets `/GL /O2` in CFLAGS and `/LTCG /USEPROFILE` in LDFLAGS; extensions get the same - [php-devel-pack-8.5.11-Win32-vs17-x64.zip](https://downloads.php.net/~windows/releases/php-devel-pack-8.5.11-Win32-vs17-x64.zip).
- Member scan of php8embed.lib (8.5.11 TS): 6,116 short import records for php8ts.dll, 3 import descriptor objects (__IMPORT_DESCRIPTOR_php8ts, __NULL_IMPORT_DESCRIPTOR, php8ts_NULL_THUNK_DATA), 1 resource object made by CVTRES, and 1 compiled x64 COFF object of 70,408 bytes that defines php_embed_init. 8.4.26 TS: 4,181 short imports, same layout, compiled object 68,933 bytes. 8.5.11 NTS: 6,079 short imports for php8.dll. No member starts with the anonymous-object header that marks a /GL object (LLVM names its class ID ClGlObjMagic) - [php-8.5.11-Win32-vs17-x64.zip](https://downloads.php.net/~windows/releases/php-8.5.11-Win32-vs17-x64.zip), [LLVM COFF.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/BinaryFormat/COFF.h).
- dev/php8ts.lib (8.5.11) is a pure import library: 6,116 short import records plus the 3 descriptor objects - [php-8.5.11-Win32-vs17-x64.zip](https://downloads.php.net/~windows/releases/php-8.5.11-Win32-vs17-x64.zip).
- The compiled php_embed object carries `.drectve` = `/alternatename:_Avx2WmemEnabled=_Avx2WmemEnabledWeakValue /DEFAULTLIB:"MSVCRT" /DEFAULTLIB:"OLDNAMES"` (dynamic CRT, /MD), references `__guard_dispatch_icall_fptr` (/guard:cf), and imports UCRT stdio (`__imp___acrt_iob_func`, `__imp___stdio_common_vfprintf`, `__imp__fileno`, `__imp__setmode`, `__imp_fwrite`, `__imp_fflush`) and PHP entry points (`php_tsrm_startup`, `sapi_startup`, `php_module_startup`, `php_request_startup`) - [php-8.5.11-Win32-vs17-x64.zip](https://downloads.php.net/~windows/releases/php-8.5.11-Win32-vs17-x64.zip).
- php8ts.dll (8.5.11 TS) imports api-ms-win-crt-convert, environment, filesystem, heap, locale, math, runtime, stdio, string, time and utility (the UCRT) and VCRUNTIME140.dll; linker version 14.44 - [php-8.5.11-Win32-vs17-x64.zip](https://downloads.php.net/~windows/releases/php-8.5.11-Win32-vs17-x64.zip).
- Of 6,116 names exported through php8ts.lib (8.5.11 TS), 340 carry __vectorcall decoration (`name@@N`), for example `_emalloc@@8`, `_efree@@8`, `_estrndup@@16`, `zend_hash_str_find@@24`, `zend_hash_update@@24`, `zend_hash_index_find@@16`. 8.4.26: 332 of 4,181 - [php-8.5.11-Win32-vs17-x64.zip](https://downloads.php.net/~windows/releases/php-8.5.11-Win32-vs17-x64.zip).
- Zend/zend_portability.h in the 8.5.11 devel pack: `#elif defined(_MSC_VER)` then `# define ZEND_FASTCALL __vectorcall`, `#else` `# define ZEND_FASTCALL` (empty). 8.4.26 uses `defined(_MSC_VER) && _MSC_VER >= 1800` - [php-devel-pack-8.5.11-Win32-vs17-x64.zip](https://downloads.php.net/~windows/releases/php-devel-pack-8.5.11-Win32-vs17-x64.zip).
- FreeUnit side: auto/modules/php:142@872bf041 probes "PHP embed SAPI" with a test program that only calls `php_module_startup()` - [auto/modules/php@872bf041](https://github.com/freeunitorg/freeunit/blob/872bf041/auto/modules/php#L142). src/nxt_php_sapi.c@872bf041 contains no `php_embed` reference (grep, no match) - [src/nxt_php_sapi.c@872bf041](https://github.com/freeunitorg/freeunit/blob/872bf041/src/nxt_php_sapi.c).
- Of 29 php/zend/sapi-style calls in src/nxt_php_sapi.c@872bf041, 19 match an export name exactly and 5 are header macros or inlines. The other PHP names (`php_end_ob_buffers` at line 257, `zend_hash_quick_find` and `zend_hash_func` at 462-463, 4-argument `zend_hash_find` at 1783) are PHP 5 APIs that cannot compile against PHP 8 headers, so they sit in old-version branches - [src/nxt_php_sapi.c@872bf041](https://github.com/freeunitorg/freeunit/blob/872bf041/src/nxt_php_sapi.c).
- Microsoft on /GL: "The format of files produced with /GL in the current version often isn't readable by later versions of Visual Studio and the MSVC toolset. Unless you're willing to ship copies of the .lib file for all versions of Visual Studio you expect your users to use, now and in the future, don't ship a .lib file made up of .obj files produced by /GL." - [/GL (Whole program optimization)](https://learn.microsoft.com/en-us/cpp/build/reference/gl-whole-program-optimization).

### Inferences
- The PHP module can link the official import library directly; php8embed.lib is optional, and FreeUnit does not need to build PHP for x64. Confidence: high.
- The PHP module must be compiled in MSVC mode, meaning a compiler that defines `_MSC_VER` and supports `__vectorcall`: cl.exe, clang-cl, or clang with a `*-windows-msvc` target. Under MinGW gcc `_MSC_VER` is undefined, ZEND_FASTCALL becomes empty, and calls to `_emalloc` and similar functions get undecorated names that php8ts.lib does not export. GCC has no vectorcall attribute at all; clang targeting MinGW has one but would still need a patched ZEND_FASTCALL. Confidence: high for the name mismatch; the failing link itself was not run.
- The PGO and /GL build of php8ts.dll does not leak into what FreeUnit links: the import records and the php_embed object are plain COFF, readable by any recent link.exe or lld-link. Confidence: high.
- Both variants link: TS through php8ts.lib keeps the existing ZTS path (src/nxt_php_sapi.c:407-415@872bf041 calls php_tsrm_startup() under ZTS), NTS links php8.lib. The module runs one request loop per process (src/nxt_php_sapi.c:530-538@872bf041), so ZTS adds no concurrency today; section 1.0 ties the choice to the Windows process model. Confidence: medium.
- A self-built PHP would only be needed for ARM64 or sanitizer and debug builds. That path needs php-sdk-binary-tools, Visual Studio and the dependency series that the builder reads from `https://downloads.php.net/~windows/php-sdk/deps/series/packages-<ver>-<vs>-<arch>-staging.txt` ([php.yml](https://github.com/php/php-windows-builder/blob/master/.github/workflows/php.yml)). Confidence: medium.

### Gaps
- Nothing was linked or run on Windows. Experiment: on a Windows runner, link a 20-line program that calls `php_module_startup()` and `_emalloc()` against php8ts.lib with cl.exe, clang-cl and MSYS2 UCRT64 gcc; expect gcc to fail on the `@@N` names.
- The PHP 8.6 php8embed.lib will grow the CLI objects (master config.w32). Check the 8.6.0RC3 vs18 zip before targeting 8.6.

## 1.3 Who builds the official binaries, Visual Studio policy, PECL DLLs

### Takeaway
GitHub Actions workflows in php/php-windows-builder build the official binaries on php-src tags, test them and deploy them to downloads.php.net/~windows. The same repository builds PECL DLLs and hands them to the downloads server. The official policy is Visual Studio 2026 (vs18) for PHP 8.6 and later.

### Cited Findings
- Shivam Mathur, PHP Foundation, 2026-06-26: "we now have automated workflows in php/php-windows-builder that build PHP on new tags in php/php-src, run tests, deploy them"; "PHP 8.0 through PHP 8.3 use the Visual Studio 2019 toolset, while newer versions such as PHP 8.4 and PHP 8.5 use the Visual Studio 2022 toolset. PHP 8.6 will use the Visual Studio 2026 toolset."; "we also revived Windows builds for extension releases on PECL" with "builds for more than 100 actively maintained extensions" - [Maintaining PHP Build infrastructure for Windows](https://thephp.foundation/blog/2026/06/26/maintaining-php-build-infrastructure-for-windows-tooling-for-builds-and-security-updates/).
- Builder README table: "8.6 / master | 2026 (vs18) | windows-2025-vs2026, github-hosted"; 8.4 and 8.5: "2022 (vs17)"; 8.0 to 8.3: "2019 (vs16)" on windows-2022 runners - [php-windows-builder README](https://github.com/php/php-windows-builder/blob/master/README.md).
- For vs18 the SDK starter passes no `-s <toolset>` argument; older VS versions pin a toolset: `$toolsetArgs = if ($VsConfig.vs -eq 'vs18') { @() } else { @('-s', $VsConfig.toolset) }` - [Invoke-PhpSdkStarter.ps1](https://github.com/php/php-windows-builder/blob/master/php/BuildPhp/private/Invoke-PhpSdkStarter.ps1).
- Latest builder commit seen: 2026-09-23 "Add PHP 8.7 support and update PHP SDK to 2.8.4" - [php-windows-builder commits](https://github.com/php/php-windows-builder/commits/master).
- PECL: workflow "Build PHP Extension From PECL"; its upload job runs `gh workflow run pecl.yml -R php/web-downloads` - [pecl.yml](https://github.com/php/php-windows-builder/blob/master/.github/workflows/pecl.yml). The newest files in the PECL release directory are dated 2026-10-03 - [PECL Windows releases](https://downloads.php.net/~windows/pecl/releases/).
- Each release zip has SBOM files beside it, for example php-8.4.26-Win32-vs17-x64.zip.cdx.json (299K), .spdx.json (195K) and .openvex.json (31K) - [releases listing](https://windows.php.net/downloads/releases/).

### Inferences
- FreeUnit should consume the official zips and devel packs and track the builder's VS map rather than build PHP. Confidence: high.
- FreeUnit's Windows CI can use the same GitHub-hosted images: windows-2022 for 8.4 and 8.5, windows-2025-vs2026 for 8.6. Confidence: medium.

### Gaps
- When the old windows.php.net PECL build machine stopped, and the date of the PECL "DLLs are back" news (search snippets say June 2024), were not confirmed from a fetched primary page. Read the archive at https://pecl.php.net/news/.

## 1.4 CRT mixing hazards and the fopen "e" flag

### Takeaway
A FILE* can cross the module/PHP boundary only when both sides use the same CRT DLL; with /MD everything shares ucrtbase.dll, which the official php8ts.dll uses. The msvcrt-based MINGW64 environment is out (MSYS2 deprecated it on 2026-03-15). Separately, the UCRT rejects the glibc "e" flag, so `fopen(filename, "re")` fails on Windows and the module needs a Windows branch.

### Cited Findings
- Microsoft (page updated 2025-06-19): "Each copy of the CRT library has a separate and distinct state"; "CRT objects such as file handles, environment variables, and locales are only valid for the copy of the CRT in the app or DLL where these objects were allocated or set."; "The DLL and its clients normally use the same copy of the CRT library only if both are linked at load time to the same version of the CRT DLL. Because the DLL version of the Universal CRT library used by Visual Studio 2015 and later is now a centrally deployed Windows component (ucrtbase.dll), it's the same for apps built with Visual Studio 2015 and later versions. However, even when the CRT code is identical, you can't give memory allocated in one heap to a component that uses a different heap." - [Potential errors passing CRT objects across DLL boundaries](https://learn.microsoft.com/en-us/cpp/c-runtime-library/potential-errors-passing-crt-objects-across-dll-boundaries).
- fopen documents the mode modifiers t, b, x, c, n, N, S, R, T, D and ccs=; there is no "e". "N: Specifies that the file isn't inherited by child processes" (equivalent oflag `_O_NOINHERIT`). "If filename or mode is NULL or an empty string, these functions trigger the invalid parameter handler ... these functions return NULL and set errno to EINVAL." "By default, a narrow filename string is interpreted using the ANSI codepage (CP_ACP)." "If t or b isn't given in mode, the default translation mode is defined by the global variable _fmode." - [fopen, _wfopen](https://learn.microsoft.com/en-us/cpp/c-runtime-library/reference/fopen-wfopen).
- UCRT source (corecrt_internal_stdio.h, "Copyright (c) Microsoft Corporation", as mirrored by ReactOS): the modifier loop of `__acrt_stdio_parse_mode` accepts '+', 'b', 't', 'c', 'n', 'S', 'R', 'T', 'D', 'N', 'x', ' ' and ','; any other character hits `default: _VALIDATE_RETURN(("Invalid file open mode", 0), EINVAL, result);` - [corecrt_internal_stdio.h source](https://doxygen.reactos.org/de/dec/corecrt__internal__stdio_8h_source.html).
- Given facts: src/nxt_php_sapi.c:1290@872bf041 opens the script with `fopen(filename, "re")` and hands the FILE* to PHP through zend_stream_init_fp, aliased at src/nxt_php_sapi.c:1271@872bf041 - [src/nxt_php_sapi.c@872bf041](https://github.com/freeunitorg/freeunit/blob/872bf041/src/nxt_php_sapi.c#L1271-L1290).
- MSYS2 environments: MSYS (cygwin), UCRT64 (gcc, ucrt, libstdc++), CLANG64 (llvm, ucrt, libc++), CLANGARM64 (llvm, aarch64, ucrt, libc++), MINGW64 and MINGW32 (gcc, msvcrt). "If you are unsure, go with UCRT64." UCRT gives "Better compatibility with MSVC, both at build time and at run time." - [MSYS2 environments](https://www.msys2.org/docs/environments/).
- MSYS2 news, 2026-03-15 "Deprecating the MINGW64 Environment": "As support for Windows 8.1 has been dropped, there is no longer a need for non-UCRT environments such as MINGW64. Consequently, we are beginning to phase out the MINGW64 environment. To start, no new packages will be added to this ..." and "please consider switching to UCRT64 or CLANG64 instead." Earlier items: 2026-02-28 "Dropping support for Windows 8.1", 2022-10-29 "Changing the default environment from MINGW64 to UCRT64" - [MSYS2 news](https://www.msys2.org/news/).
- php8ts.dll imports the api-ms-win-crt-* UCRT sets and VCRUNTIME140.dll, and php_embed.obj is built with /DEFAULTLIB:"MSVCRT" (/MD) - [php-8.5.11-Win32-vs17-x64.zip](https://downloads.php.net/~windows/releases/php-8.5.11-Win32-vs17-x64.zip).
- Microsoft on longjmp: "In Microsoft C++ code on Windows, longjmp uses the same stack-unwinding semantics as exception-handling code." - [longjmp](https://learn.microsoft.com/en-us/cpp/c-runtime-library/reference/longjmp).

### Inferences
- `fopen(filename, "re")` will fail on the UCRT. Whether the process also dies depends on the invalid parameter handler in force at that moment. Confidence: high that it fails, medium on the crash.
- Fix on Windows: use `zend_stream_init_filename()` and let PHP open the file. PHP then reads the path through its own Windows file layer, which avoids the ANSI code page problem of a narrow fopen, and no FILE* crosses the boundary. If a FILE* is kept, use `_wfopen()` with a UTF-16 path and mode "rbN" (binary, not inheritable). Confidence: medium (PHP's internal path handling not read here).
- In the recommended design (module compiled in MSVC mode with /MD, see question 2) FILE* sharing and zend_try/zend_bailout across php8ts.dll stay within one CRT and one compiler family. Confidence: high.
- A MinGW UCRT64 module also binds to ucrtbase.dll, so sharing a FILE* with php8ts.dll should work in principle, but this is moot for PHP (vectorcall). It matters only for Ruby and Perl, whose stacks are MinGW end to end. Confidence: medium.

### Gaps
- No run on Windows. Experiment: call `fopen("x.php", "re")` with the default handler and with `_set_invalid_parameter_handler()`, and record the return value, errno and process exit code. Read which handler php_module_startup installs (php-src main/main.c).
- setjmp/longjmp between gcc-built frames and MSVC's SEH-based longjmp was not tested; no primary source was found for mixed-compiler unwinding.

## 1.5 clang-cl, lld-link, MinGW, cross-compiling, Wine

### Takeaway
clang-cl is the practical compiler: Visual Studio ships it, it defines `_MSC_VER`, emits MSVC-ABI COFF and links with the official PHP import libraries. FrankenPHP's merged Windows build already links official php8ts.lib and php8embed.lib with clang and lld. lld-link refuses /GL objects, which only matters when rebuilding PHP. Cross-compiling from Linux (msvc-wine, xwin) needs the Microsoft license and is not redistributable; build and test on real Windows runners and use Wine 11.0 for smoke tests at most.

### Cited Findings
- lld-link rejects cl.exe /GL objects: `case file_magic::coff_cl_gl_object:` reports `": is not a native COFF file. Recompile without /GL"` (Driver.cpp lines 354-356, and again at 478-480) - [lld/COFF/Driver.cpp](https://github.com/llvm/llvm-project/blob/main/lld/COFF/Driver.cpp).
- "First, Clang attempts to be ABI-compatible, meaning that Clang-compiled code should be able to link against MSVC-compiled code successfully." - [Clang MSVC compatibility](https://clang.llvm.org/docs/MSVCCompatibility.html).
- Microsoft (page updated 2025-12-11): "Visual Studio by default invokes Clang in clang-cl mode. It links with the Microsoft implementation of the Standard Library." and "This LLVM toolset is provided as-is, sourced directly from the LLVM Foundation's release page without modifications by Microsoft." - [Clang/LLVM support in Visual Studio projects](https://learn.microsoft.com/en-us/cpp/build/clang-support-msbuild).
- PHP's own build system knows a clang toolset: confutils.js sets `CLANG_TOOLSET` when `PHP_TOOLSET` is "clang" and then looks up `clang-cl` - [php-devel-pack-8.5.11-Win32-vs17-x64.zip](https://downloads.php.net/~windows/releases/php-devel-pack-8.5.11-Win32-vs17-x64.zip).
- FrankenPHP "feat: Windows support", merged 2026-02-26: `CC: clang`, `CXX: clang++`, Go `-ldflags=-extldflags=-fuse-ld=lld`, and `CGO_LDFLAGS=... -L$phpBin -L$phpDevel\lib -lphp8ts -lphp8embed`, where `$phpBin` is the unpacked `php-<version>-Win32-vs17-x64.zip` (php8embed.lib sits at its root) and `$phpDevel` the unpacked devel pack (php8ts.lib in lib/). The workflow picks the newest version that matches `php-\d+\.\d+\.\d+-Win32-vs17-x64\.zip` - [FrankenPHP PR 2119](https://github.com/php/frankenphp/pull/2119).
- FrankenPHP issue 2314 (opened 2026-03-26, closed): 2026-03-27 "the GL compiled dependencies make Clang fail when using Ltcg. The PGO profile is also only for msvc." (about rebuilding php8ts.dll with clang against the roughly 40 prebuilt dependencies); 2026-03-28 the default `/GUARD:CF` costs much of the speed; 2026-05-27 "Merged in 8.6 so closing here. Will need to update the build CI to Clang-cl then." - [FrankenPHP issue 2314](https://github.com/php/frankenphp/issues/2314).
- llvm-mingw: the "primary target is UCRT"; it supports ARM and ARM64; "The toolchain defaults to using the Universal CRT and targeting Windows 7." Latest release 20260922, published 2026-09-22 - [llvm-mingw](https://github.com/mstorsjo/llvm-mingw).
- msvc-wine: "Downloading and installing it requires accepting the license, available at https://go.microsoft.com/fwlink/?LinkId=2086102 for the currently latest version. As Visual Studio isn't redistributable, the installed toolchain isn't either." It provides x86, x64, arm and arm64 tools and can also feed clang-cl - [msvc-wine](https://github.com/mstorsjo/msvc-wine).
- xwin downloads the Microsoft CRT and Windows SDK headers and libraries; `--accept-license` skips "the prompt to accept the license"; it documents clang-cl as compiler and lld-link as linker; xwin itself is Apache-2.0 or MIT - [xwin](https://github.com/Jake-Shadle/xwin).
- Wine: the annotated tag wine-11.0 ("Release 11.0") is dated 2026-01-13; on 2026-10-07 there is no wine-11.0.x maintenance tag, and development tags reach wine-11.19 - [wine-11.0 tag](https://github.com/wine-mirror/wine/tree/wine-11.0).
- Given facts: auto/os/test:77-88@872bf041 has a `MINGW*)` branch (comment: "MinGW /bin/sh builtin echo omits newline under Wine for some reason, so use a portable echo.c program built using MinGW GCC with only msvcrt.dll dependence") that sets echo=auto/echo/echo.exe, CC=${CC:-cl}, AR=${AR:-ar}, NXT_WINDOWS=YES; auto/echo/Makefile:2-3@872bf041 builds echo.exe with `mingw32-gcc -o echo.exe -O2 echo.c`; NXT_WINDOWS is used nowhere else - [auto/os/test@872bf041](https://github.com/freeunitorg/freeunit/blob/872bf041/auto/os/test#L77-L88), [auto/echo/Makefile@872bf041](https://github.com/freeunitorg/freeunit/blob/872bf041/auto/echo/Makefile#L2-L3).

### Inferences
- Use clang-cl as the one compiler for the core, libunit and the MSVC-ABI modules. It keeps the GNU C extensions that a GCC-first code base tends to use while matching the MSVC ABI; cl.exe is the fallback. Confidence: medium-high (FreeUnit's use of GNU extensions was not audited here).
- Keep the nginx-style model the MINGW branch already assumes: a POSIX sh (MSYS2 MSYS environment) drives configure, CC=clang-cl, archiver llvm-lib or lib.exe instead of `ar`. The auto/echo workaround (mingw32-gcc, msvcrt.dll) is a legacy of an msvcrt-era MinGW and can go. Confidence: medium.
- Linux cross builds with xwin plus clang-cl and lld-link are technically viable for C, but the toolchain bits fall under Microsoft's license and cannot be redistributed; GitHub-hosted Windows runners avoid the question. Confidence: medium.
- Wine is fine for a "does unitd start and answer one request" smoke check; do not treat it as evidence for socket, IOCP or process-model behaviour. Confidence: low-medium (no Wine test done).

### Gaps
- The Microsoft license at https://go.microsoft.com/fwlink/?LinkId=2086102 was not read; whether running the Build Tools on Linux CI for an open-source project is allowed stays open.
- Wine 11.0 fidelity for FreeUnit's event engine and AF_UNIX use was not tested.
- php-src PR 21563 (security flags, from issue 2314) and the 8.6 "preserve_none" work were not read.

## 1.6 Other language runtimes on Windows

### Takeaway
Python, Node.js and Java can share the MSVC-ABI compiler with the core and PHP. Ruby (RubyInstaller2) and Perl (Strawberry Perl) are MinGW-w64 UCRT builds, so those modules would need a second toolchain (MSYS2 UCRT64) or another interpreter build.

### Cited Findings
- Python: CPython's Windows build instructions start with "Install Microsoft Visual Studio 2017 or later with Python workload" and also describe an optional clang-cl build (clang+llvm-18.1.8-x86_64-pc-windows-msvc) - [CPython PCbuild/readme.txt](https://github.com/python/cpython/blob/main/PCbuild/readme.txt).
- Ruby: "This project provides an Installer for Ruby-2.4 and newer on Windows based on the MSYS2 toolchain"; binary gems target "x64-mingw-ucrt or aarch64-mingw-ucrt"; latest release RubyInstaller-4.0.7-1, published 2026-09-15 - [RubyInstaller2](https://github.com/oneclick/rubyinstaller2).
- Perl: release "Strawberry Perl 5.42.3.1 64-bit UCRT" (tag SP_54231_64bit, published 2026-08-17): "Compiled using GCC 13.2 with UCRT." - [Strawberry Perl release](https://github.com/StrawberryPerl/Perl-Dist-Strawberry/releases/tag/SP_54231_64bit).
- Node.js: "As of Node.js 24.0.0, ClangCL is required to compile on Windows."; prerequisites "Visual Studio 2022 or 2026 with the Windows 11 SDK on a 64-bit host"; official win-x64 and win-arm64 binaries are built on Windows Server 2022 with Visual Studio 2022 - [Node.js BUILDING.md](https://github.com/nodejs/node/blob/main/BUILDING.md). node.lib is published for addons (3,029,908 bytes on 2026-10-07) - [node.lib](https://nodejs.org/dist/latest/win-x64/node.lib).
- Java: `#define JNIEXPORT __declspec(dllexport)` and `#define JNIIMPORT __declspec(dllimport)`, attributes that both MSVC and MinGW accept - [OpenJDK jni_md.h (Windows)](https://github.com/openjdk/jdk/blob/master/src/java.base/windows/native/include/jni_md.h).

### Inferences
- Python module: compile with clang-cl and link the python3XX.lib import library from a python.org install. Confidence: medium-high (libs/ layout not re-checked).
- Node.js: FreeUnit's Node support is an npm native addon built by node-gyp with Visual Studio against node.lib; no extra compiler. Confidence: medium.
- Java: JNI works with either compiler; link jvm.lib from the JDK. Confidence: medium.
- Ruby and Perl: their headers and config are generated for MinGW gcc, so build those modules with MSYS2 UCRT64. They share ucrtbase.dll with an MSVC-built core, and the module to core boundary is plain C, so this is workable but doubles the CI toolchains. Defer both. Confidence: medium.

### Gaps
- python3XX.lib in libs/, jvm.lib in the JDK lib/ directory, and MSVC-built (mswin) Ruby or Perl distributions were not checked.

## 1.7 Conclusion: which compiler for the core and for the modules

### Takeaway
Build the core, libunit and the PHP module with clang-cl (MSVC ABI, /MD, UCRT) from Visual Studio 2022 17.14 or Visual Studio 2026. The PHP module must be MSVC-mode because 340 php8ts.dll exports are __vectorcall and PHP's headers select that convention only under `_MSC_VER`; it links php8ts.lib or php8.lib from the official devel pack, with no PHP rebuild. One compiler serves the core, PHP, Python, Node.js and Java; Ruby and Perl would need MSYS2 UCRT64 or other interpreter builds.

### Cited Findings
- 340 of 6,116 php8ts.dll exports are `name@@N` (__vectorcall) in 8.5.11 TS - [php-8.5.11-Win32-vs17-x64.zip](https://downloads.php.net/~windows/releases/php-8.5.11-Win32-vs17-x64.zip).
- ZEND_FASTCALL is __vectorcall only when `_MSC_VER` is defined - [php-devel-pack-8.5.11-Win32-vs17-x64.zip](https://downloads.php.net/~windows/releases/php-devel-pack-8.5.11-Win32-vs17-x64.zip).
- Official 8.4 and 8.5 builds use the VS 2022 toolset, linker 14.44; 8.6 moves to VS 2026 - [Maintaining PHP Build infrastructure for Windows](https://thephp.foundation/blog/2026/06/26/maintaining-php-build-infrastructure-for-windows-tooling-for-builds-and-security-updates/).
- clang with lld links the official php8ts.lib and php8embed.lib in a shipped product - [FrankenPHP PR 2119](https://github.com/php/frankenphp/pull/2119).
- msvcrt MinGW (MINGW64) is deprecated since 2026-03-15 - [MSYS2 news](https://www.msys2.org/news/).

### Inferences
- Recommended toolchain: Visual Studio 2022 17.14 Build Tools plus its "C++ Clang tools for Windows" (clang-cl, lld-link), with the MSYS2 MSYS shell only to run configure and make. Add Visual Studio 2026 when PHP 8.6 lands. Confidence: medium-high.
- Import libraries do not tie FreeUnit to PHP's exact MSVC version, so a newer Visual Studio links an older PHP fine. Confidence: high for import records; the general "use the newest linker" rule was not re-read.
- First target is x64 only. Confidence: high.
- Rust components fit this plan through the x86_64-pc-windows-msvc target. Confidence: medium (section 2.4).

### Gaps
- Microsoft's binary compatibility page (https://learn.microsoft.com/en-us/cpp/porting/binary-compat-2015-2017) was not read.
- No end-to-end Windows link of the PHP module yet; the experiment in question 2 settles it in one CI job.

## 1.8 Authenticode status of the official PHP binaries

### Takeaway
None of the 64 .exe and .dll files in php-8.5.11-Win32-vs17-x64.zip carries an embedded Authenticode signature, and the sampled 8.4.26 files are unsigned too. A FreeUnit bundle built on official PHP would ship unsigned PHP binaries.

### Cited Findings
- PE data directory 4 (IMAGE_DIRECTORY_ENTRY_SECURITY) has offset 0 and size 0 for php.exe, php-cgi.exe, php8ts.dll, ext/php_openssl.dll, ext/php_curl.dll, ext/php_mbstring.dll and ext/php_sodium.dll (8.5.11 TS), and for php.exe, php8ts.dll, ext/php_openssl.dll and ext/php_opcache.dll (8.4.26 TS). All 64 PE files in the 8.5.11 TS zip have size 0 - [php-8.5.11-Win32-vs17-x64.zip](https://downloads.php.net/~windows/releases/php-8.5.11-Win32-vs17-x64.zip), [php-8.4.26-Win32-vs17-x64.zip](https://downloads.php.net/~windows/releases/php-8.4.26-Win32-vs17-x64.zip).
- The 8.5.11 TS zip has no ext/php_opcache.dll and its devel pack has no lib/php_opcache.lib; the 8.4.26 zip and devel pack have both - [php-8.5.11-Win32-vs17-x64.zip](https://downloads.php.net/~windows/releases/php-8.5.11-Win32-vs17-x64.zip).
- FrankenPHP's Windows workflow finds php8embed.lib through `-L$phpBin` (the binary zip directory) and php8ts.lib through `-L$phpDevel\lib`; the devel pack's lib/ has no php8embed.lib - [FrankenPHP PR 2119](https://github.com/php/frankenphp/pull/2119).

### Inferences
- Windows 11 Smart App Control may block these unsigned DLLs, as a Laravel Herd user reported on 2026-09-28 ([herd-community#1757](https://github.com/beyondcode/herd-community/issues/1757), section 6.1). FreeUnit cannot sign PHP's own files; a local-dev bundle should document this or ship a PHP build it signs itself. Confidence: medium.
- The missing php_opcache.dll in 8.5 matches OPcache being built into PHP from 8.5. Confidence: medium (RFC not read).

### Gaps
- Catalog (.cat) signatures were not checked; only embedded signatures were.

## 2.1 How nginx builds on Windows

### Takeaway
nginx builds natively on Windows with its own POSIX-sh configure under MSYS or MSYS2, `--with-cc=cl`, and
`nmake`. A small MSVC layer does the work: auto/cc/msvc (167 lines), auto/os/win32 (39 lines) and three
makefile.msvc files. On win32 it skips the Unix probe scripts and uses a hand-written config header. The
official Windows zip in 2026 is a 32-bit x86 binary built with cl 19.44 (Visual Studio 2022 toolset).
freenginx also ships Windows zips, built with cl 19.16. Neither includes njs, and the MSVC path has no
dynamic modules.

### Cited Findings
- Method: links to nginx source files point at nginx master as fetched on 2026-10-07 - [nginx/nginx](https://github.com/nginx/nginx)
- Required tools: "Microsoft Visual Studio 8, 10, 17 are known to work", MSYS or MSYS2, Perl (ActivePerl or Strawberry Perl) for OpenSSL, a Git client, and PCRE, zlib and OpenSSL sources. Mercurial is no longer listed - [Building nginx on the Win32 platform with Visual C](https://nginx.org/en/docs/howto_build_on_win32.html)
- Documented command: `auto/configure --with-cc=cl --with-debug --prefix= --conf-path=conf/nginx.conf --pid-path=logs/nginx.pid --http-log-path=logs/access.log --error-log-path=logs/error.log --sbin-path=nginx.exe --http-client-body-temp-path=temp/client_body_temp --http-proxy-temp-path=temp/proxy_temp --http-fastcgi-temp-path=temp/fastcgi_temp --http-scgi-temp-path=temp/scgi_temp --http-uwsgi-temp-path=temp/uwsgi_temp --with-cc-opt=-DFD_SETSIZE=1024 --with-pcre=objs/lib/pcre2-10.39 --with-zlib=objs/lib/zlib-1.3.1 --with-openssl=objs/lib/openssl-3.0.14 --with-openssl-opt=no-asm --with-http_ssl_module`, then `nmake` - [Building nginx on the Win32 platform with Visual C](https://nginx.org/en/docs/howto_build_on_win32.html)
- configure maps `uname -s` values `MINGW32_* | MINGW64_* | MSYS_*` to `NGX_PLATFORM=win32` (lines 39-40) and skips `auto/headers` and `auto/unix`, the Unix feature probes, for win32 (lines 52-60) - [nginx auto/configure](https://github.com/nginx/nginx/blob/master/auto/configure#L27-L60)
- `--crossbuild=*` sets `NGX_PLATFORM` directly (line 212). For win32 the `WINE` variable becomes `NGX_WINE` (line 645). configure then skips `uname` and sets `NGX_MACHINE=i386` (auto/configure lines 44-47) - [nginx auto/options](https://github.com/nginx/nginx/blob/master/auto/options#L212)
- MSVC is selected only when `$CC` is exactly `cl`; the version comes from the cl banner, run through `$NGX_WINE` when cross-building - [nginx auto/cc/name lines 27-36](https://github.com/nginx/nginx/blob/master/auto/cc/name#L27-L36)
- auto/cc/msvc: target machine from the cl banner (`*ARM64` to arm64, `*x64` to amd64, otherwise i386, lines 19-33); `-W4` (94) and `-WX` (97); static CRT `-MT` (106); `kernel32.lib user32.lib` (113); `-Zi` PDB unless under Wine (121-124); precompiled header (134-138); resource compiler `rc` (142-144); `-Fo`, `-Fe` and `.obj` output naming (153-155); nmake inline response files `@<<` for long link lines (157-163) - [nginx auto/cc/msvc](https://github.com/nginx/nginx/blob/master/auto/cc/msvc)
- auto/os/win32: `OS_CONFIG="$WIN32_CONFIG"` (line 11) and `ngx_binext=".exe"` (17). For gcc or clang it links `-ladvapi32 -lws2_32` and sets `MODULE_LINK`, `--export-all-symbols` and `--out-implib` (21-25). For MSVC it only adds `advapi32.lib ws2_32.lib` (28-30) and sets no module link - [nginx auto/os/win32](https://github.com/nginx/nginx/blob/master/auto/os/win32)
- Dependencies are built from source, statically, by nmake makefiles. OpenSSL: `perl Configure $(OPENSSL_TARGET) no-shared no-threads` (line 9) - [makefile.msvc](https://github.com/nginx/nginx/blob/master/auto/lib/openssl/makefile.msvc). zlib: plain `cl -c` and `link -lib` - [makefile.msvc](https://github.com/nginx/nginx/blob/master/auto/lib/zlib/makefile.msvc). PCRE2: a source list used when `NGX_CC_NAME = msvc` (line 10) - [auto/lib/pcre/make](https://github.com/nginx/nginx/blob/master/auto/lib/pcre/make). auto/lib/pcre/makefile.msvc still names PCRE1 files (`pcre_*.c`) - [makefile.msvc](https://github.com/nginx/nginx/blob/master/auto/lib/pcre/makefile.msvc)
- On 2026-10-07 the download page offers nginx/Windows-1.31.6 (mainline), nginx/Windows-1.30.5 (stable) and legacy zips, and says nothing about compiler or architecture - [nginx: download](https://nginx.org/en/download.html)
- The 1.31.6 zip holds `nginx.exe`, a PE32 image for machine 0x014c (32-bit x86). Its strings read "built by cl 19.44.35211 for x86" and configure arguments `--with-cc=cl --builddir=objs.msvc8 ... --with-pcre=objs.msvc8/lib/pcre2-10.48 --with-zlib=objs.msvc8/lib/zlib-1.3.2 --with-http_v2_module ... --with-mail --with-stream ... --with-openssl=objs.msvc8/lib/openssl-3.5.8 --with-openssl-opt='no-asm no-tests no-makedepend -D_WIN32_WINNT=0x0501' --with-http_ssl_module ...`. Embedded OpenSSL is "3.5.8 25 Aug 2026". Read from the PE header and strings with Python; the binary was not run - [nginx-1.31.6.zip](https://nginx.org/download/nginx-1.31.6.zip)
- nginx changes: 1.29.5 (2026-02-04) "fixed warning when compiling with MSVC 2022 x86"; 1.15.10 (2019-03-26) "nginx/Windows could not be built with Visual Studio 2015 or newer"; 1.13.2 (2017-06-27) "could not be built under MSYS2 / MinGW 64-bit"; 1.11.8 (2016-12-27) "could not be built with 64-bit Visual Studio" - [nginx changes.xml](https://github.com/nginx/nginx/blob/master/docs/xml/nginx/changes.xml)
- Older official zips used Visual Studio 2010: nginx 1.13.3 reported "built by cl 16.00.40219.01 for 80x86" - [nginx trac ticket 1390](https://trac.nginx.org/nginx/ticket/1390)
- freenginx offers freenginx/Windows-1.31.4 (mainline) and freenginx/Windows-1.30.1 (stable) - [freenginx: download](https://freenginx.org/en/download.html). Its 1.31.4 `nginx.exe` is also PE32 x86, "built by cl 19.16.27051 for x86", with pcre2 10.48, zlib 1.3.2 and OpenSSL 3.5.8 - [freenginx-1.31.4.zip](https://freenginx.org/download/freenginx-1.31.4.zip)
- Neither zip has njs or dynamic modules: no `--add-module` or `--add-dynamic-module` in the configure arguments, and no DLL or modules directory among the 45 (nginx) and 41 (freenginx) archive entries - [nginx-1.31.6.zip](https://nginx.org/download/nginx-1.31.6.zip), [freenginx-1.31.4.zip](https://freenginx.org/download/freenginx-1.31.4.zip)

### Inferences
- cl 19.44 is the Visual Studio 2022 version 17.14 toolset; cl 19.16 is Visual Studio 2017 version 15.9. The mapping is from Microsoft's `_MSC_VER` table and was not re-fetched for these notes. Confidence high.
- nginx's Windows build avoids nearly all compile-and-run probes and relies on a hand-written header. That is why about 270 lines of MSVC-specific shell and makefiles are enough. FreeUnit cannot copy that approach unchanged, because its probes also compute values (struct sizes, see section 7). Confidence medium-high.
- The official zip is a compatibility build: 32-bit, no assembler in OpenSSL, `_WIN32_WINNT=0x0501` passed to OpenSSL. It shows the toolchain works with current MSVC. It is not a model for a 64-bit production build. Confidence high.
- The nginx MSVC path has no dynamic module support. FreeUnit's language modules are dynamic objects loaded by unitd. This is the largest gap between "copy nginx" and what FreeUnit needs. Confidence high.

### Gaps
- nginx does not publish the exact Visual Studio edition or Windows SDK of its release builder. Reading the PE linker version and the Rich header of nginx.exe would narrow it.
- `--crossbuild=win32` under Wine was not tested; its last fix in changes.xml was not searched.

## 2.2 Precedents for a second build system

### Takeaway
PostgreSQL and curl each had a Windows-only build system beside the main one, and both removed it.
PostgreSQL added Meson in 16 (2023-09-14) and removed the MSVC-specific build in 17 (2024-09-26); autoconf
remains. curl dropped winbuild in 8.17.0 (2025-11-05) and keeps autotools plus CMake. The reported cost was
neglect and drift of the Windows build, plus slow, hard-to-read test runs, not the tooling itself.

### Cited Findings
- PostgreSQL 16.0, released 2023-09-14: "Add meson build system (Andres Freund, Nazir Bilal Yavuz, Peter Eisentraut). This eventually will replace the Autoconf and Windows-based MSVC build systems." - [PostgreSQL 16.0 release notes](https://www.postgresql.org/docs/release/16.0/)
- PostgreSQL 17.0, released 2024-09-26: "Remove the Microsoft Visual Studio-specific PostgreSQL build option (Michael Paquier). Meson is now the only available method for Visual Studio builds." - [PostgreSQL 17.0 release notes](https://www.postgresql.org/docs/release/17.0/)
- RFC by Andres Freund, 2021-10-12: the msvc project generator is "something most of us unix-y folks do not like to touch", and that, "combined with there being no easy way to run all tests, and it being just different, really hurt our windows developer appeal (and subsequently the quality of postgres on windows)". Autoconf is "*barely* in maintenance mode". Recursive make needed `.NOTPARALLEL`. Windows CI build 202 s (old) against 140 s (meson+ninja) and 206 s (meson+msbuild); Windows CI tests 1323 s against 903 s. Local build 40.46 s against 7.31 s - [[RFC] building postgres with meson](https://www.postgresql.org/message-id/20211012083721.hvixq4pnh2pixr3j@alap3.anarazel.de)
- curl's past removals: "winbuild build system (removed in 8.17.0)" and "CMake 3.17 and older (removed in 8.20.0)" - [curl docs/DEPRECATE.md](https://github.com/curl/curl/blob/master/docs/DEPRECATE.md)
- curl 8.17.0, released 2025-11-05: "build: drop the winbuild build system" - [curl 8.17.0 changes](https://curl.se/ch/8.17.0.html)
- Daniel Stenberg, 2018-01-29: "We have at least three different ways (down from four a few years ago) to build on windows (winbuild makefiles, cmake and autotools)" - [curl-library mail 2018-01/0115](https://curl.se/mail/lib-2018-01/0115.html)
- libuv and nghttp2 keep both autotools (`configure.ac`, `Makefile.am`) and `CMakeLists.txt` at the top level; libgit2 keeps only CMake (top-level listings on 2026-10-07) - [libuv](https://github.com/libuv/libuv), [nghttp2](https://github.com/nghttp2/nghttp2), [libgit2](https://github.com/libgit2/libgit2)

### Inferences
- A Windows-only second build system lags the main one because the main developers do not run it. PostgreSQL and curl both ended by deleting theirs. Both still run two cross-platform systems (autoconf and Meson; autotools and CMake). Confidence high.
- PostgreSQL needed about three years from RFC (2021-10) to the MSVC removal (2024-09), led by a full-time developer, and autoconf is still there. A small fork should not expect a faster full migration. Confidence medium.

### Gaps
- The Meson commit date (2022-09-22) comes from a search summary; it was not read from git.
- curl's 2016 plan to drop `Makefile.vc*` in favour of winbuild appeared only in a search snippet ([curl-library 2016-08/0057](https://curl.se/mail/lib-2016-08/0057.html)). No written curl, libuv, libgit2 or nghttp2 statement on the cost of two build systems was found.

## 2.3 Dependencies on Windows

### Takeaway
Every library FreeUnit links has a current vcpkg port with no platform restriction, and the x64-windows,
x64-windows-static and arm64-windows triplets are built in. OpenSSL from source needs Perl and NASM; nginx
avoids NASM with `no-asm`. Prebuilt OpenSSL is also available from Shining Light and FireDaemon. vcpkg's
GitHub Actions cache provider `x-gha` was removed in April 2025, not 2024.

### Cited Findings
- vcpkg master at commit f451d04d49 (2026-10-07): openssl 3.6.5 (port-version 1, Apache-2.0), pcre2 10.49 (BSD-3-Clause), zlib 1.3.2 (port-version 2, Zlib), zlib-ng 2.3.3 (Zlib), brotli 1.2.0 (MIT), zstd 1.5.7 (BSD-3-Clause OR GPL-2.0-only). None declares a `supports` restriction - [openssl](https://github.com/microsoft/vcpkg/blob/master/ports/openssl/vcpkg.json), [pcre2](https://github.com/microsoft/vcpkg/blob/master/ports/pcre2/vcpkg.json), [zlib](https://github.com/microsoft/vcpkg/blob/master/ports/zlib/vcpkg.json), [zlib-ng](https://github.com/microsoft/vcpkg/blob/master/ports/zlib-ng/vcpkg.json), [brotli](https://github.com/microsoft/vcpkg/blob/master/ports/brotli/vcpkg.json), [zstd](https://github.com/microsoft/vcpkg/blob/master/ports/zstd/vcpkg.json)
- Built-in triplets include x64-windows, x64-windows-static, x64-windows-static-md and arm64-windows. arm64-windows-static exists only as a community triplet - [vcpkg triplets](https://github.com/microsoft/vcpkg/tree/master/triplets), [community triplets](https://github.com/microsoft/vcpkg/tree/master/triplets/community)
- vcpkg has no wasmtime port (`ports/wasmtime/vcpkg.json` returned 404) - [vcpkg ports](https://github.com/microsoft/vcpkg/tree/master/ports)
- OpenSSL NOTES-WINDOWS.md: Strawberry Perl recommended; "NASM is the only supported assembler"; targets `VC-WIN32`, `VC-WIN64A`, `VC-WIN64-ARM`, `VC-WIN64-CLANGASM-ARM`, and `-HYBRIDCRT` variants that depend on the Universal CRT; build with nmake from a Visual Studio Developer Command Prompt - [OpenSSL NOTES-WINDOWS.md](https://github.com/openssl/openssl/blob/master/NOTES-WINDOWS.md)
- Shining Light Productions: the page loads its table by script; its hash file lists Win64OpenSSL, Win32OpenSSL, Win64ARMOpenSSL and WinUniversalOpenSSL installers for 3.5.9, 3.6.5 and 4.0.3, plus Light editions - [Win32/Win64 OpenSSL](https://slproweb.com/products/Win32OpenSSL.html), [win32_openssl_hashes.json](https://github.com/slproweb/opensslhashes/blob/master/win32_openssl_hashes.json)
- FireDaemon OpenSSL: free x86, x64 and ARM64 binaries, built with Visual Studio Community 2026 using `VC-WIN32-HYBRIDCRT` and `VC-WIN64A-HYBRIDCRT`, covering 3.5.x LTS and 4.0.x; use "governed by the OpenSSL License" - [FireDaemon OpenSSL](https://www.firedaemon.com/firedaemon-openssl)
- vcpkg GitHub Actions cache page: "The GitHub Actions Cache backend for binary caching has been removed. This tutorial is no longer maintained." (page dated 2025-05-01) - [Microsoft Learn: vcpkg binary cache with GitHub Actions Cache](https://learn.microsoft.com/en-us/vcpkg/consume/binary-caching-github-actions-cache)
- vcpkg-tool PR 1662 "Remove x-gha binary cache provider", merged 2025-04-29. Reason: the GitHub Actions cache internal APIs were sunset and were never meant for outside use. Replacements: NuGet on GitHub Packages, which the vcpkg team recommends and which keeps per-port granularity; or `actions/cache` over the `installed` directory, one entry for the whole tree - [microsoft/vcpkg-tool PR 1662](https://github.com/microsoft/vcpkg-tool/pull/1662)

### Inferences
- Use vcpkg in manifest mode with a pinned baseline. Start with x64-windows-static-md (static libraries, dynamic CRT) or x64-windows; add arm64-windows later. The CRT choice must match the PHP build the module embeds: /MD, dynamic UCRT (section 1.4). Confidence medium.
- Prefer the OpenSSL 3.5 LTS line for a long-lived product, as nginx, freenginx and FireDaemon do; vcpkg's default is 3.6.5, so pin with a version override. Confidence medium.
- For CI, start with `actions/cache` keyed on the vcpkg baseline, triplet and manifest hash; move to NuGet on GitHub Packages if per-port reuse matters. The brief's "files provider with actions/cache" is a community pattern; PR 1662 does not name it. Confidence medium.
- FireDaemon's "OpenSSL License" wording conflicts in form with vcpkg's Apache-2.0 for OpenSSL 3.x. OpenSSL 3.0 and later are Apache-2.0, so this is a wording difference. Confidence medium (OpenSSL's license page not fetched).

### Gaps
- vcpkg.io package pages were not fetched; versions come from port files at one commit.
- Not checked: whether the vcpkg pcre2 and openssl ports install `.pc` files that FreeUnit's `pcre2-config` or pkg-config paths could use, and the exact `.lib` names. Settle with `vcpkg install openssl pcre2 zlib --triplet x64-windows-static-md` and a listing of `installed/<triplet>/lib`.

## 2.4 Rust on Windows

### Takeaway
The needed Rust targets are Tier 1: x86_64-pc-windows-msvc, and aarch64-pc-windows-msvc since Rust 1.91.0
(2025-10-30). The crates src/otel uses run CI on Windows. unitctl is the blocker: its client crate depends
on hyperlocal, which wraps a Unix-only tokio type, and its CI builds no Windows target.

### Cited Findings
- Tier 1 with host tools: x86_64-pc-windows-msvc, aarch64-pc-windows-msvc, i686-pc-windows-msvc, x86_64-pc-windows-gnu. Tier 2 with host tools: x86_64-pc-windows-gnullvm, aarch64-pc-windows-gnullvm, i686-pc-windows-gnu. Minimum Windows 10 or Windows Server 2016 - [rustc book: Platform Support](https://doc.rust-lang.org/rustc/platform-support.html)
- aarch64-pc-windows-msvc became Tier 1 in Rust 1.91.0, released 2025-10-30 - [Announcing Rust 1.91.0](https://blog.rust-lang.org/2025/10/30/Rust-1.91.0/), [RFC 3817](https://rust-lang.github.io/rfcs/3817-promote-aarch64-pc-windows-msvc-to-tier-1.html)
- unitctl has the comment "Sockets on Windows are not supported" - tools/unitctl/unit-client-rs/src/control_socket_address.rs:316-318@872bf041 (verified)
- The unitctl workflow builds no Windows target. Its `test` job has two legs, `x86_64-unknown-linux-gnu` and `aarch64-apple-darwin` (.github/workflows/unitctl.yml:49-56@872bf041); a musl job builds `x86_64-unknown-linux-musl` (.github/workflows/unitctl.yml:87-121@872bf041); the `build` job builds `aarch64-unknown-linux-gnu`, `x86_64-unknown-linux-gnu`, `aarch64-apple-darwin` and `x86_64-apple-darwin` (.github/workflows/unitctl.yml:192-205@872bf041)
- unit-client-rs depends on `hyper` 1 (line 24), `hyperlocal = "0.9"` (27), `tokio` (36) and `bollard = "0.21"` (46) - tools/unitctl/unit-client-rs/Cargo.toml:24-46@872bf041
- hyperlocal's client wraps `tokio::net::UnixStream` (src/client.rs lines 23-27); its modules are gated only on cargo features, not on `cfg(unix)` (src/lib.rs lines 21-28) - [hyperlocal client.rs](https://github.com/softprops/hyperlocal/blob/main/src/client.rs), [hyperlocal lib.rs](https://github.com/softprops/hyperlocal/blob/main/src/lib.rs)
- tokio: `UnixStream` is "Available on Unix and crate feature net only"; `tokio::net::windows::named_pipe` is "Available on Windows and crate feature net only" - [tokio UnixStream](https://docs.rs/tokio/latest/tokio/net/struct.UnixStream.html), [tokio named_pipe](https://docs.rs/tokio/latest/tokio/net/windows/named_pipe/index.html)
- The uds_windows crate is at 1.2.1, updated 2026-03-14, repository haraldh/rust_uds_windows - [crates.io: uds_windows](https://crates.io/crates/uds_windows)
- src/otel is a `staticlib` (src/otel/Cargo.toml:8@872bf041) using opentelemetry-otlp 0.33 with `reqwest-blocking-client` and `grpc-tonic` (src/otel/Cargo.toml:33-34@872bf041) and tonic 0.14 (src/otel/Cargo.toml:37@872bf041). The core links it as `$NXT_BUILD_DIR/lib/libotel.a` (auto/make:46@872bf041), built by `cargo rustc` (auto/make:689@872bf041)
- opentelemetry-rust, tonic and hyper run CI on `windows-latest` - [opentelemetry-rust ci.yml, lines 29 and 135](https://github.com/open-telemetry/opentelemetry-rust/blob/HEAD/.github/workflows/ci.yml), [tonic CI.yml, lines 65, 167, 249](https://github.com/hyperium/tonic/blob/HEAD/.github/workflows/CI.yml), [hyper CI.yml, line 84](https://github.com/hyperium/hyper/blob/HEAD/.github/workflows/CI.yml)

### Inferences
- unitctl should fail to compile for a Windows target as it stands, because hyperlocal names `tokio::net::UnixStream` without a `cfg(unix)` gate. Confidence medium (not compiled).
- Cheapest unitctl path on Windows: gate hyperlocal behind `cfg(unix)` and use the TCP control address, which unitctl already parses as a URI. The controller applies its peer-credential check only to AF_UNIX peers and lets other peers through (src/nxt_controller.c:891-906@872bf041), so a TCP control socket has no caller check. That may be acceptable on loopback for a developer machine, not for production. Confidence medium.
- For one transport on both platforms, Windows AF_UNIX through uds_windows (with a tokio adapter) beats a named pipe, which needs a new server side in unitd. The choice should follow what the core port uses for its own sockets (architecture chapter). Confidence low-medium.
- On MSVC targets Rust names a static library `otel.lib`, not `libotel.a`, and its system libraries must be taken from `--print native-static-libs`. auto/make:46@872bf041 hard-codes the Unix name. Confidence high for the name, medium for the library list.
- No Windows blocker was found for opentelemetry-otlp, tonic or hyper; all three test on Windows. Confidence medium.

### Gaps
- Run `cargo check --target x86_64-pc-windows-msvc` for tools/unitctl and src/otel on a Windows runner. That settles the hyperlocal claim and any C build-script issues (TLS crates).
- Issue trackers of opentelemetry-rust, tonic and reqwest were not searched for Windows bugs. bollard's Windows named-pipe support was not checked.

## 2.5 wasmtime on Windows

### Takeaway
wasmtime supports x86_64-pc-windows-msvc at Tier 1 and ships prebuilt C API archives for x86_64, aarch64 and
i686 Windows and x86_64 MinGW. aarch64-pc-windows-msvc is only Tier 3. The FreeUnit wasm module would need
`.lib` and `.dll` naming; module loading is the same problem as for PHP. Both wasm modules can wait.

### Cited Findings
- Tier 1: x86_64-pc-windows-msvc. Tier 2: x86_64-pc-windows-gnu, missing "Clear owner of the target". Tier 3: aarch64-pc-windows-msvc, missing "CI testing, full-time maintainer" - [Wasmtime stability tiers](https://docs.wasmtime.dev/stability-tiers.html)
- Release v49.0.2, published 2026-10-02, has `wasmtime-v49.0.2-x86_64-windows-c-api.zip`, `-aarch64-windows-c-api.zip`, `-i686-windows-c-api.zip` and `-x86_64-mingw-c-api.zip` - [wasmtime v49.0.2](https://github.com/bytecodealliance/wasmtime/releases/tag/v49.0.2)
- The x86_64-windows C API archive contains `lib/wasmtime.dll` (22,389,760 bytes), `lib/wasmtime.dll.lib` (import library), `lib/wasmtime.lib` (static, 84,072,486 bytes) and a smaller `min/` variant - [wasmtime-v49.0.2-x86_64-windows-c-api.zip](https://github.com/bytecodealliance/wasmtime/releases/download/v49.0.2/wasmtime-v49.0.2-x86_64-windows-c-api.zip)
- FreeUnit's wasm module documents `libwasmtime.so` and links `-L${NXT_WASM_LIB_PATH} -lwasmtime` - auto/modules/wasm:30,64,99@872bf041

### Inferences
- The wasm module (C API) can plausibly build on x86_64 Windows once link names change. Confidence medium.
- wasm-wasi-component (Rust, wasmtime crate) should compile on x86_64-pc-windows-msvc, which is Tier 1 for both Rust and wasmtime. It shares the module-loading and libunit port work. Confidence low-medium (not compiled).
- Leave both modules out of the first PHP milestone. Confidence medium.

### Gaps
- The wasmtime version FreeUnit pins and whether wasm-wasi-component's bindgen step works against MSVC headers were not checked.

## 2.6 njs on Windows

### Takeaway
njs does not support Windows, and its maintainers declined to add it in 2020 and 2023. The nginx and
freenginx Windows zips contain no njs. A Windows FreeUnit must build without njs.

### Cited Findings
- njs auto/os has cases for Linux, FreeBSD/NetBSD/OpenBSD, SunOS, Darwin and a default, and no Windows or MINGW case. src/ has `njs_unix.h` and no Windows header - [njs auto/os](https://github.com/nginx/njs/blob/master/auto/os), [njs src](https://github.com/nginx/njs/tree/master/src)
- Issue 320, maintainer reply on 2020-06-17: "njs project does not support Windows, (only POSIX-compatible OSs: linux, freebsd, Solaris, macOs,..)". A 2025-04-01 comment describes a Cygwin build - [nginx/njs issue 320](https://github.com/nginx/njs/issues/320)
- Issue 629 "RFC: Support compilation on Windows", 2023-03-22: "we have no plans for Windows support. Because it is not only requires modifying code, but also maintenance." Closed 2023-04-09 - [nginx/njs issue 629](https://github.com/nginx/njs/issues/629)
- The nginx 1.31.6 and freenginx 1.31.4 Windows configure arguments include no njs module (section 1) - [nginx-1.31.6.zip](https://nginx.org/download/nginx-1.31.6.zip)

### Inferences
- Treat njs as unsupported on Windows. `--njs` should stop configure with a clear message there. Confidence high.

### Gaps
- None that changes the plan.

## 2.7 What in FreeUnit's auto/ breaks under MSVC

### Takeaway
FreeUnit's auto/ (54 files, 8,503 lines at 872bf041) assumes a GCC-style driver throughout: `-o`, `-l`,
`-shared`, `-fPIC`, `ar`, `.o`, `.a` and `.so`, `*-config` scripts and GNU make syntax. The only Windows
code is a MINGW stub that picks `cl` and sets a flag nobody reads. nginx's auto/cc/msvc solves compiler
identification, flags, output naming, CRT, PDB and long command lines. It does not solve DLL modules,
run-time value probes or Rust static libraries, which FreeUnit needs.

### Cited Findings
- MINGW* branch: `echo=auto/echo/echo.exe`, `CC=${CC:-cl}`, `AR=${AR:-ar}`, `NXT_WINDOWS=YES` - auto/os/test:77-88@872bf041 (verified). Note `AR` defaults to `ar`, not `lib`
- echo.exe is built with `mingw32-gcc -o echo.exe -O2 echo.c` - auto/echo/Makefile:2-3@872bf041 (verified)
- `NXT_WINDOWS` occurs only where it is set - auto/os/test:87@872bf041 (git grep over the whole tree, verified)
- The compiler check uses `which $CC` - auto/cc/test:16@872bf041. Identification greps `$CC -v` output for "gcc version", "clang version" or "Apple LLVM version" - auto/cc/test:24-45@872bf041. Any other compiler falls into an empty `*)` case - auto/cc/test:146-147@872bf041. gcc and clang add `-fPIC` (63, 107), `-fvisibility=hidden` (66, 110), `-Wall -Wextra` (75, 119) and `-Werror` (97) - auto/cc/test@872bf041
- Feature probes compile with `$CC ... -o $NXT_AUTOTEST $NXT_AUTOTEST.c $nxt_feature_libs $NXT_LD_OPT $NXT_TEST_LIBS`, test `[ -x $NXT_AUTOTEST ]`, run the program, and for `run=value` write its output into the generated config header - auto/feature:38-76@872bf041
- auto/os/conf has cases Linux, FreeBSD, SunOS, Darwin, NetBSD, OpenBSD, DragonFly, AIX, HP-UX, QNX and `*)`, and none for Windows - auto/os/conf:21-259@872bf041. Link templates: `$(AR) -r -c`, `$(CC) -shared -Wl,-soname,libnxt.so`, `$(CC) -Wl,-E`, `libnxt.a`, `libnxt.so`, `-lm` - auto/os/conf:15-64@872bf041
- The generated Makefile uses GNU make syntax (`:=`, `ifeq`) - auto/make:13-18,63-88@872bf041; detects GNU make with `make --version | grep GNU` and calls `uname -s` - auto/make:53-57@872bf041; names objects `${nxt_src%.c}.o` - auto/make:114,128@872bf041; links with `-o \$@` - auto/make:156-170@872bf041
- Libraries are named with `-l` or found through `*-config`: `-lrt` for shm_open (auto/shmem:44@872bf041); `-lssl -lcrypto` (auto/ssltls:16@872bf041); `pcre2-config --libs8` (auto/pcre:13@872bf041); `php-config`, `-lphp` or `-lphp${major}`, `libphp*.a` (auto/modules/php:56-108@872bf041); `-lwasmtime` (auto/modules/wasm:64@872bf041)
- Probes test Unix facilities by name: auto/sockets checks `socketpair(AF_UNIX, SOCK_SEQPACKET)`, `struct msghdr.msg_control`, `sockopt SO_PASSCRED`, `struct ucred`, `struct sockaddr_un size` and `accept4()`; auto/shmem checks `shm_open()` and `memfd_create()` (the `nxt_feature=` lines in auto/sockets@872bf041 and auto/shmem@872bf041)
- pkg-config is used in auto/make (1 line) and auto/njs (3 lines) - git grep over auto/@872bf041

### Inferences
- Portable from nginx with little change: compiler detection from the cl banner, target machine, warning and CRT flags, `-Fo`, `-Fe`, `.obj`, `.exe`, `.lib` system libraries, PDB, resource file, nmake-style response files for long link lines. nginx keeps this working against current MSVC (its 2026 zip uses cl 19.44). Confidence high.
- Not solved by nginx, and needed by FreeUnit. (a) DLL modules: unitd.exe or a core DLL must export what modules call, with an import library; nginx's MSVC branch has no module link at all (nginx auto/os/win32 lines 28-30). (b) Run-time value probes: nginx skips probes on win32; FreeUnit's auto/feature must name outputs `-Fe$NXT_AUTOTEST.exe` and run them. That works for native builds and fails for cross builds without Wine. (c) The Rust static library name and its native libraries (section 4). Confidence high.
- Keep GNU make. nginx emits nmake syntax, but MSYS2 ships GNU make, which can drive cl.exe. Rewriting auto/make for nmake gains nothing. Confidence medium.
- MSYS2 rewrites arguments that look like POSIX paths when it starts a native program. Using dash-style cl options, as nginx does, avoids mangled `/Fo`-style options. Confidence low-medium (MSYS2 documentation not fetched).
- With MinGW-w64 (gcc or clang driver), `-o`, `-l` and `-shared` keep working and the port shrinks to OS cases and DLL export flags; nginx's gcc/clang branch shows the pattern (`--export-all-symbols`, `--out-implib`, nginx auto/os/win32 lines 21-25). Section 1.2 rules this out for the PHP module: PHP's headers request the `__vectorcall` names of the official exports only under `_MSC_VER`, and gcc has no vectorcall support. Confidence medium-high.
- Most of the Unix-facility probes will just report "not found". The work they reveal (AF_UNIX SEQPACKET, descriptor passing, shm_open) is source porting, not build system work. Confidence medium.

### Gaps
- `./configure` was not run under MSYS2 with `CC=clang-cl` (or `CC=cl`). One `windows-2025` CI job with MSYS2 would list every failing probe in the autoconf error log.
- Not tested: whether `[ -x name ]` finds `name.exe` under MSYS2 bash, and how `which cl` behaves there.

## 2.8 Recommendation: one build system or two

### Takeaway
Keep one build system. Extend configure and auto/ to run under MSYS2 bash with an MSVC-mode compiler (clang-cl, section 1.0; nginx uses cl.exe), and
keep GNU make. Take OpenSSL, PCRE2, zlib, brotli and zstd from vcpkg instead of building them in the
Makefile. Do not add a Windows-only CMake or Meson build. Consider a full move to Meson only if the spike
shows Windows special cases spreading across many probe files, or if Windows CI time becomes a measured
problem.

### Cited Findings
- FreeUnit's auto/ descends from nginx's and already has a Windows hook that selects `cl` - auto/os/test:77-88@872bf041
- nginx's MSVC layer is small and current: auto/cc/msvc 167 lines, auto/os/win32 39 lines, three makefile.msvc files of 17 to 23 lines, and the 2026 official zip is built with cl 19.44 - [nginx auto/cc/msvc](https://github.com/nginx/nginx/blob/master/auto/cc/msvc), [nginx-1.31.6.zip](https://nginx.org/download/nginx-1.31.6.zip)
- PostgreSQL removed its Windows-only MSVC build after Meson arrived and still keeps autoconf - [PostgreSQL 17.0 release notes](https://www.postgresql.org/docs/release/17.0/)
- curl removed its Windows-only winbuild in 8.17.0 (2025-11-05) - [curl 8.17.0 changes](https://curl.se/ch/8.17.0.html)
- PostgreSQL's stated reason: a separate Windows generator that Unix developers did not touch hurt Windows quality - [[RFC] building postgres with meson](https://www.postgresql.org/message-id/20211012083721.hvixq4pnh2pixr3j@alap3.anarazel.de)
- The dependencies exist in vcpkg for the needed triplets - [vcpkg triplets](https://github.com/microsoft/vcpkg/tree/master/triplets)

### Inferences
- Why not a Windows-only CMake or Meson build: it repeats the failure both precedents removed. Every module option, probe and hardening flag would have to be kept in sync by people who do not build on Windows. Confidence high.
- Why not move everything to Meson now: it rewrites 8,503 lines of auto/ and breaks the nginx lineage that the Windows layer would be copied from. FreeUnit's per-runtime module configure (`./configure php --config=...`, one module per runtime version) maps less directly to Meson options. PostgreSQL's migration took about three years with a full-time lead. The first target is a developer machine, not a CI-time problem. Confidence medium.
- Why auto/ under MSYS2 with an MSVC-mode compiler: one source of truth; nginx proves the model with current MSVC; FreeUnit needs roughly nginx's layer plus three additions (DLL module linking with an exported core, `.exe` handling in auto/feature, Rust library naming), plus vcpkg include and library paths passed through existing `--cc-opt`/`--ld-opt` style options. Confidence medium.
- First steps: (1) a `windows-2025` CI job that runs configure under MSYS2 with `CC=clang-cl` and keeps the configure log (`build/autoconf.err`, configure:25@872bf041) as an artifact; (2) an `auto/cc/msvc`-style case in auto/cc/test and a MINGW/MSYS case in auto/os/conf; (3) a link test of one module DLL against unitd.exe exports; (4) `cargo check --target x86_64-pc-windows-msvc` for src/otel and tools/unitctl; (5) build with `--njs` and the wasm modules disabled. Confidence medium.
- A MinGW-w64 build would have kept auto/'s GCC-style driver assumptions, but section 1.2 rules MinGW out for the PHP module, so the MSVC-mode work in this section is needed. Confidence medium-high.

### Gaps
- The spike in step (1) is the deciding experiment. Its output (count and spread of failing probes and link lines) should decide between "auto/ under MSYS2" and "move to Meson".
- The cost of keeping MSYS2 as a build prerequisite for Windows contributors was not measured. nginx accepts it.

## 3.1 Hosted Windows runners in October 2026

### Takeaway
Three x64 and arm64 Windows labels matter: `windows-2025` (also `windows-latest`), `windows-2022` and `windows-11-arm`.
Since June 2026 `windows-latest` and `windows-2025` carry Visual Studio 2026 (18.x). `windows-2022` keeps Visual Studio 2022 (17.14).
Public repositories get 4 vCPU, 16 GB RAM and 14 GB SSD on every standard Windows label, x64 and arm64. `windows-2019` is gone since 2025-06-30.

### Cited Findings
- `windows-latest` moved from Windows Server 2022 to Windows Server 2025 between 2025-09-02 and 2025-09-30 - [GitHub Actions: New APIs and windows-latest migration notice (2025-07-31)](https://github.blog/changelog/2025-07-31-github-actions-new-apis-and-windows-latest-migration-notice/)
- `windows-latest` and `windows-2025` moved to Visual Studio 2026 by default between 2026-06-08 and 2026-06-15. `windows-2025-vs2026` was the test label and "will point to the windows-2025 image" after the move. To stay on VS 2022 the post says to use `windows-2022` - [GitHub Actions: Upcoming image migrations (2026-05-14)](https://github.blog/changelog/2026-05-14-github-actions-upcoming-image-migrations)
- The runner-images table today maps `windows-latest`, `windows-2025` and `windows-2025-vs2026` to the Windows2025-VS2026 image, `windows-2022` to Windows2022, `windows-11-arm` to Windows11-Arm64 and `windows-11-vs2026-arm` to Windows11-VS2026-Arm64 - [actions/runner-images README](https://github.com/actions/runner-images/blob/main/README.md)
- Windows2025-VS2026 image 20260925.250.1: OS 10.0.26100, Visual Studio Enterprise 2026 18.10.12217.157, the side-by-side VS 2022 toolset component `Microsoft.VisualStudio.Component.VC.14.44.17.14.x86.x64`, Windows 11 SDK 26100, Python 3.12.10, Rust 1.98.1, rustup 1.29.1, CMake 4.4.3, LLVM 20.1.8, PHP 8.5.11, vcpkg in `C:\vcpkg` (`VCPKG_INSTALLATION_ROOT`) - [Windows2025-VS2026-Readme.md](https://github.com/actions/runner-images/blob/main/images/windows/Windows2025-VS2026-Readme.md)
- Windows2022 image 20260927.320.1: OS 10.0.20348, Visual Studio Enterprise 2022 17.14.37710.0, Python 3.12.10, Rust 1.98.1, CMake 3.31.6, vcpkg in `C:\vcpkg` - [Windows2022-Readme.md](https://github.com/actions/runner-images/blob/main/images/windows/Windows2022-Readme.md)
- Windows11-Arm64 image 20260927.180.1: OS 10.0.26200, Visual Studio Enterprise 2022 17.14.37710.0, Python 3.13.15, LLVM 22.1.8, PHP 8.4.26, Rust 1.98.1, vcpkg in `C:\vcpkg` - [Windows11-Arm64-Readme.md](https://github.com/actions/runner-images/blob/main/images/windows/Windows11-Arm64-Readme.md)
- A Windows2025-Readme.md (VS 2022 17.14, image 20260927.275.1) still exists in the repository but the README table no longer links it - [Windows2025-Readme.md](https://github.com/actions/runner-images/blob/main/images/windows/Windows2025-Readme.md)
- The Windows 11 arm64 image with VS 2026 became generally available on standard and larger runners (label `windows-11-vs2026-arm`) on 2026-08-20. `windows-11-arm` was to move to VS 2026 between 2026-09-21 and 2026-09-30 - [Windows 11 arm64 VS2026 image generally available (2026-08-20)](https://github.blog/changelog/2026-08-20-windows-11-arm64-vs2026-image-generally-available)
- CONFLICT: the runner-images table and the `windows-11-arm` image readme dated 2026-09-27 still show Visual Studio 2022 17.14 for `windows-11-arm`. Either the move slipped or the documentation lags - [actions/runner-images README](https://github.com/actions/runner-images/blob/main/README.md)
- `windows-2019` was fully retired by 2025-06-30, after brownouts on 3, 10, 17 and 24 June 2025 from 13:00 to 21:00 UTC. Retirement follows the N-1 OS policy - [Upcoming breaking changes and releases for GitHub Actions (2025-04-15)](https://github.blog/changelog/2025-04-15-upcoming-breaking-changes-and-releases-for-github-actions/)
- Public repositories: Windows x64 runners have 4 CPU, 16 GB RAM, 14 GB SSD (`windows-latest`, `windows-2025`, `windows-2025-vs2026`, `windows-2022`); Windows arm64 runners have 4 CPU, 16 GB, 14 GB (`windows-11-arm`, `windows-11-vs2026-arm`). Private repositories: 2 CPU and 8 GB for both architectures - [GitHub-hosted runners reference](https://docs.github.com/en/actions/reference/runners/github-hosted-runners)
- Linux and Windows arm64 standard runners became generally available for public repositories on 2025-08-07 with 4 vCPU. The post said they "are only available in public repositories" and pointed private repositories to arm64 larger runners - [arm64 hosted runners for public repositories are now generally available](https://github.blog/changelog/2025-08-07-arm64-hosted-runners-for-public-repositories-are-now-generally-available/)
- CONFLICT (superseded): the current runners reference lists `windows-11-arm` under private repositories (2 CPU, 8 GB), and the pricing page has a "Windows arm64 2-core" rate. The public-only restriction from 2025-08-07 no longer holds - [GitHub-hosted runners reference](https://docs.github.com/en/actions/reference/runners/github-hosted-runners), [Actions runner pricing](https://docs.github.com/en/billing/reference/actions-runner-pricing)

### Inferences
- Pin explicit labels in required checks and never use `windows-latest`. It changed twice in nine months (Server 2025 in September 2025, VS 2026 in June 2026). Confidence: high.
- Use `windows-2025` (VS 2026) for the core build and `windows-2022` (VS 2022) for PHP 8.4 and 8.5 module legs, because php-windows-builder builds those PHP versions with VS 2022 on `windows-2022` (section 3). The VS 2026 image also carries the v14.44 toolset, so `vcvarsall.bat x64 -vcvars_ver=14.44` is a fallback on one image. Confidence: medium.
- Treat `windows-11-arm` as a nightly leg until the VS 2026 move settles, and pin `windows-11-vs2026-arm` if the leg needs a known compiler. Confidence: medium.
- 14 GB of SSD is enough for the core tree, vcpkg packages and a PHP devel pack. This is not measured. Confidence: medium.

### Gaps
- Whether `windows-11-arm` now runs VS 2026: print `vswhere -latest -property catalog_productLineVersion` in a scratch job.
- The Windows on Arm public preview start date (a Windows Developer Blog post of 2025-04-14 appeared in search) was not fetched.
- Concurrent job limits for a GitHub Free organization were not read. Read the "Actions limits" page before sizing a matrix.

## 3.2 Billing in 2026

### Takeaway
FreeUnit is a public repository, so standard Windows runners cost nothing. The binding limits are wall time and concurrency, not money.
GitHub cut hosted runner prices by up to 39% on 2026-01-01. The $0.002 per minute platform charge for self-hosted runners, planned for 2026-03-01, was postponed, not cancelled. No new date was found.

### Cited Findings
- "GitHub Actions usage is free for self-hosted runners and for public repositories that use standard GitHub-hosted runners." Included minutes: Free 2,000, Pro and Team 3,000, Enterprise Cloud 50,000 per month - [GitHub Actions billing](https://docs.github.com/en/billing/concepts/product-billing/github-actions)
- Per-minute rates for standard runners: Linux 1-core $0.002, Linux 2-core $0.006, Windows 2-core $0.010, Linux arm64 2-core $0.005, Windows arm64 2-core $0.010, macOS $0.062. The page states rates, not OS multipliers - [Actions runner pricing](https://docs.github.com/en/billing/reference/actions-runner-pricing)
- Larger Windows runners: x64 4, 8, 16 cores at $0.022, $0.042, $0.082 per minute; arm64 at $0.014, $0.026, $0.050. "The larger runners are not free for public repositories." - [Actions runner pricing](https://docs.github.com/en/billing/reference/actions-runner-pricing)
- The December 2025 announcement: hosted runner prices down "up to 39%" from 2026-01-01; a "$0.002 per minute GitHub Actions cloud platform charge" for self-hosted runners from 2026-03-01; "Runner usage in public repositories will remain free". The post now opens with: "postponing the announced billing change for self-hosted GitHub Actions to take time to re-evaluate our approach" - [Update to GitHub Actions pricing (2025-12-16)](https://github.blog/changelog/2025-12-16-coming-soon-simpler-pricing-and-a-better-experience-for-github-actions/)
- The GitHub resources page (2025-12-15) says the platform charge is already included in the reduced hosted prices, that the self-hosted part is postponed, and that "Standard GitHub-hosted or self-hosted runner usage on public repositories will remain free." - [Pricing changes for GitHub Actions](https://github.com/resources/insights/2026-pricing-changes-for-github-actions)

### Inferences
- The Windows to Linux price ratio for 2-core x64 is 0.010 / 0.006 = 1.67. For arm64 it is 0.010 / 0.005 = 2.0. This matters only for private forks. Confidence: high (arithmetic on cited rates).
- A private fork running a 20-minute Windows leg on a 2-core runner pays $0.20 per run. Confidence: high.
- Do not plan on larger runners; they are billed even for public repositories. Confidence: high.

### Gaps
- Whether GitHub set a new date or a new design for the self-hosted charge after December 2025. A changelog search on "self-hosted" for 2026 found nothing; this is absence of evidence only.
- The pre-2026 Windows rate and the old "2x minute multiplier" were not re-verified from a primary source.

## 3.3 Tooling: MSVC environment, MSYS2, vcpkg, PHP SDK, setup-php, Rust

### Takeaway
`ilammy/msvc-dev-cmd` is not archived, but it has had no push since 2024-04, and it declares `node20`, which GitHub retired in September 2026. Use the maintained fork `TheMrMilchmann/setup-msvc-dev` or call `vcvarsall.bat` directly.
vcpkg is preinstalled on every Windows image, but its `x-gha` cache provider was removed in April 2025. Cache with the `files` provider plus `actions/cache`, or use NuGet on GitHub Packages.
`php/setup-php-sdk` is archived. Its successor is `php/php-windows-builder`, which supports x64 and x86 only. `shivammathur/setup-php` installs PHP and prebuilt DLL extensions on Windows, including arm64, but no devel pack.

### Cited Findings
- `ilammy/msvc-dev-cmd`: not archived; last push 2024-04-01; latest release v1.13.0 of 2024-01-01 (GitHub API). Its action.yml declares `using: node20` - [ilammy/msvc-dev-cmd](https://github.com/ilammy/msvc-dev-cmd), [action.yml](https://github.com/ilammy/msvc-dev-cmd/blob/HEAD/action.yml)
- Open issues there include "Add support for Visual Studio 2026 (vsversion: "2026")" (2026-03-04) and "Taking on more maintainers?" (2026-04-13) - [issue 98](https://github.com/ilammy/msvc-dev-cmd/issues/98), [issue 103](https://github.com/ilammy/msvc-dev-cmd/issues/103)
- Node 20 is no longer available in GitHub Actions; maintainers must set `runs.using` to `node24`; "The temporary ACTIONS_ALLOW_USE_UNSECURE_NODE_VERSION opt-out is no longer available." - [Node 20 is no longer available in GitHub Actions (2026-09-23)](https://github.blog/changelog/2026-09-23-node-20-is-no-longer-available-in-github-actions)
- `TheMrMilchmann/setup-msvc-dev` is a fork of ilammy's action, release v4.1.0 of 2026-07-22, last push 2026-10-05 (GitHub API). Inputs include `arch`, `vs-path`, `sdk`, `toolset` - [TheMrMilchmann/setup-msvc-dev](https://github.com/TheMrMilchmann/setup-msvc-dev)
- `msys2/setup-msys2` is active: release v2.33.0 on 2026-09-27, not archived (GitHub API). curl pins it at v2.33.0 - [setup-msys2 v2.33.0](https://github.com/msys2/setup-msys2/releases/tag/v2.33.0), [curl windows.yml](https://github.com/curl/curl/blob/master/.github/workflows/windows.yml)
- vcpkg removed the GitHub Actions cache backend: "The GitHub Actions Cache backend for binary caching has been removed. This tutorial is no longer maintained." - [Microsoft Learn: binary caching with GitHub Actions Cache](https://learn.microsoft.com/en-us/vcpkg/consume/binary-caching-github-actions-cache)
- The binary caching reference marks `x-gha` as "Removed" and documents a Files provider, a NuGet provider and a GitHub Packages quickstart - [Microsoft Learn: binary caching configuration](https://learn.microsoft.com/en-us/vcpkg/reference/binarycaching)
- vcpkg-tool PR 1662 "Remove `x-gha` binary cache provider" was merged on 2025-04-29. It lists two migrations: NuGet on GitHub Packages ("the method recommended by the vcpkg team", per-port granularity) and `actions/cache` on the `installed` directory ("endorsed by the GitHub Actions team", whole-tree invalidation) - [microsoft/vcpkg-tool PR 1662](https://github.com/microsoft/vcpkg-tool/pull/1662)
- curl uses the preinstalled vcpkg: `vcpkg x-set-installed ... --triplet=...` and `-DCMAKE_TOOLCHAIN_FILE=$VCPKG_INSTALLATION_ROOT/scripts/buildsystems/vcpkg.cmake` - [curl windows.yml](https://github.com/curl/curl/blob/master/.github/workflows/windows.yml)
- `php/setup-php-sdk` is archived (GitHub API `archived: true`, last release v0.12 of 2026-01-09). Its README says to migrate to `php/php-windows-builder` - [php/setup-php-sdk](https://github.com/php/setup-php-sdk)
- `php/php-windows-builder` (release 1.9.0 of 2026-07-27) builds PHP and extensions on Windows. The `extension` action takes `php-version`, `arch`, `ts`, `args`, `libs`, `run-tests`, `test-runner`. `arch` supports `x64` and `x86`. Toolsets: PHP 8.0 to 8.3 use VS16 (2019), PHP 8.4 and 8.5 use VS17 (2022) on `windows-2022`, PHP 8.6 and master use VS18 (2026) on `windows-2025-vs2026` - [php/php-windows-builder](https://github.com/php/php-windows-builder)
- `shivammathur/setup-php` lists `windows-2025` and `windows-2022` (PHP 8.5 preinstalled) and `windows-11-arm` (PHP 8.4 preinstalled). `phpts` accepts `nts` or `zts`/`ts`; default is `nts`. "On Windows, extensions available on PECL which have the DLL binary can be set up." - [shivammathur/setup-php](https://github.com/shivammathur/setup-php)
- Rust 1.98.1, Cargo 1.98.1 and rustup 1.29.1 are preinstalled on all three Windows images - [Windows2025-VS2026-Readme.md](https://github.com/actions/runner-images/blob/main/images/windows/Windows2025-VS2026-Readme.md), [Windows11-Arm64-Readme.md](https://github.com/actions/runner-images/blob/main/images/windows/Windows11-Arm64-Readme.md)

### Inferences
- Do not adopt `ilammy/msvc-dev-cmd` for new work. Either pin `TheMrMilchmann/setup-msvc-dev` by SHA or run `vswhere` and `vcvarsall.bat` in one pwsh step and export the environment. The second has no third-party dependency. Confidence: high.
- For OpenSSL, PCRE2, zlib, brotli and zstd use the preinstalled vcpkg with a manifest and a pinned baseline. Set `VCPKG_BINARY_SOURCES=clear;files,<cache dir>,readwrite` and save that directory with `actions/cache`, keyed on vcpkg commit, triplet and manifest hash. The files provider keeps per-package archives, so a partial hit still saves time. Confidence: medium (design, not measured).
- Fork pull requests cannot write to GitHub Packages, so NuGet caching only helps pushes to the main repository. Files plus `actions/cache` works for both. Confidence: medium.
- The PHP module is an embed SAPI consumer, not a PECL extension, so the php-windows-builder `extension` action does not fit directly. The likely path is its `php` action (or the official devel pack) to get headers and the embed library, then FreeUnit's own build. Confidence: medium; see Gaps.
- arm64 PHP module CI cannot use php-windows-builder (x64 and x86 only). setup-php gives a PHP binary on `windows-11-arm` but no devel pack. Confidence: high for the facts, medium for the conclusion that arm64 PHP needs a self-built PHP.
- Use `dtolnay/rust-toolchain` or rustup directly and cache `~/.cargo` plus `target` with `actions/cache` for unitctl and other Rust parts. Not researched further. Confidence: low.

### Gaps
- What happens today to an action declaring `node20` (fails, or forced onto Node 24). Run `ilammy/msvc-dev-cmd@v1` in a scratch workflow.
- Settled in 1.2: `php8embed.lib` ships at the root of the official PHP 8.4 and 8.5 release zips (TS and NTS), and the matching devel pack carries the headers (including `include/sapi/embed/php_embed.h`) and the import library. CI needs both zips, not a self-built PHP, for x64.
- Whether `actions/setup-python` offers CPython 3.13 or 3.14 for Windows arm64 was not checked.

## 3.4 CI leg durations for comparable C projects

### Takeaway
Windows build plus test legs of comparable C projects take 2 to 9 minutes on standard runners (curl, libuv). PostgreSQL splits its Visual Studio test leg into two slices of about 16 and 17 minutes.
PostgreSQL removed Cirrus CI on 2026-06-04 and now runs its Windows jobs on GitHub Actions, so the Cirrus task named in the brief no longer exists.

### Cited Findings
- Method: job durations come from the GitHub REST API (job `started_at` to `completed_at`, queue time excluded); the run pages linked below show the same values - [GitHub REST API: workflow jobs](https://docs.github.com/en/rest/actions/workflow-jobs)
- PostgreSQL commit 68c8a365d4 "ci: Remove support for cirrus-ci based CI", committed 2026-06-04. CI now lives in `.github/workflows/pg-ci.yml` - [postgres/postgres commit 68c8a365d4](https://github.com/postgres/postgres/commit/68c8a365d4dcefd7ec60f7a99a7ab7b557c36357)
- CONFLICT with the brief: just before removal the Cirrus Windows task was named "Windows - Server 2022, VS 2019 - Meson & ninja", not "Server 2019" - [.cirrus.tasks.yml at the parent commit, line 769](https://github.com/postgres/postgres/blob/9c126063b19adc53a0ce15ac1ae1a70979e3c12e/.cirrus.tasks.yml)
- PostgreSQL's `windows-vs` job runs on `windows-2022` with `timeout-minutes: 60`, a matrix of 2 slices, a step that disables Windows Defender real-time monitoring, and `PG_TEST_USE_UNIX_SOCKETS: 1` - [pg-ci.yml](https://github.com/postgres/postgres/blob/master/.github/workflows/pg-ci.yml)
- PostgreSQL run 37558281186 on master (2026-10-07): "Windows - Visual Studio - Slice 1/2" 17m18s, "Slice 2/2" 15m58s, "Windows - MinGW - Meson" 15m55s, all on `windows-2022`. For comparison "Linux - Meson (64-bit)" 20m19s and "macOS - Meson" 19m19s - [postgres run 37558281186](https://github.com/postgres/postgres/actions/runs/37558281186)
- curl run 34119299124 on master (2026-09-07): "msvc, CM x64-windows openssl +examples" 7m12s on `windows-2025`; "msvc, CM arm64-windows schannel U" 8m56s on `windows-11-arm`; "msvc, CM x64-uwp !ssl +examples" 2m11s; "mingw, CM ucrt-x86_64 schannel U torture 1" and "torture 2" 9m38s and 8m8s on `windows-2022`; msys2 legs 4m30s to 6m51s; "linux-mingw" cross builds on `ubuntu-26.04` 0m53s to 4m5s - [curl run 34119299124](https://github.com/curl/curl/actions/runs/34119299124)
- libuv run 37394064082 on v1.x (2026-10-06): "Visual Studio 17 2022" x64 4m41s, x64 ASAN 6m2s, arm64 2m28s on `windows-2022`; "Visual Studio 18 2026" x64 5m12s on `windows-2025`; mingw cross builds on `ubuntu-26.04` 0m55s and 1m57s; the tests of those builds on `windows-2022` 2m27s and 2m28s - [libuv run 37394064082](https://github.com/libuv/libuv/actions/runs/37394064082)
- libuv builds with mingw-w64 on Ubuntu, uploads the binaries as an artifact, and runs the tests on a Windows runner - [libuv CI-win.yml](https://github.com/libuv/libuv/blob/v1.x/.github/workflows/CI-win.yml)

### Inferences
- A FreeUnit core build with MSVC plus the C test table should land at 5 to 10 minutes on `windows-2025` once vcpkg packages are cached. That sits between libuv (4 to 6 minutes) and curl's MSVC leg (7 minutes). Confidence: medium-low (no FreeUnit measurement).
- A Windows pytest subset should be sharded once it passes about 20 minutes, as PostgreSQL does. Confidence: medium.
- Copy PostgreSQL's Defender step for test legs that create many small files. Its effect size is not measured. Confidence: medium.
- Linux mingw cross builds take about 1 to 4 minutes and give fast compile-error feedback (curl, libuv). Confidence: high for the precedent.

### Gaps
- Cold vcpkg build time for openssl, pcre2, zlib, brotli and zstd on `windows-2025`. Measure once without cache.
- Queue latency for Windows and arm64 runners was not measured.

## 3.5 The pytest suite's Unix assumptions and their Windows equivalents

### Takeaway
On Windows CPython the suite fails before the first test. `test/conftest.py` imports `fcntl` at module level, and `test/unit/http.py` evaluates `socket.AF_UNIX` on every request, TCP requests included.
CPython 3.13 and 3.14 on Windows have no `socket.AF_UNIX`. The feature request gh-77589 has been open since 2018, and its PR 137420 awaits review. Winsock AF_UNIX exists (since Insider build 17063) but is SOCK_STREAM only, with no socketpair and no descriptor passing.
A Windows harness needs four replacements: TCP loopback for control, Job Objects for process trees, psutil for process scans and handle counts, and a graceful stop that does not depend on POSIX signals.

### Cited Findings
Repository facts at 872bf041:
- `import fcntl` at module level, used to reset stdout flags - test/conftest.py:2@872bf041, test/conftest.py:121@872bf041. `fcntl` availability is "Unix, not WASI" - [Python docs: fcntl](https://docs.python.org/3/library/fcntl.html)
- Top-level `import pwd` and `import grp` in four test modules - test/test_control_socket_peer.py:1@872bf041, test/test_controller_conn_teardown.py:2@872bf041, test/test_go_isolation.py:1@872bf041, test/test_python_application.py:1@872bf041. `pwd` is "Unix, not WASI, not iOS"; `grp` is "Unix, not WASI, not Android, not iOS" - [Python docs: pwd](https://docs.python.org/3/library/pwd.html), [Python docs: grp](https://docs.python.org/3/library/grp.html)
- `os.killpg(pgid, sig)` and the probe `os.killpg(pgid, 0)` - test/conftest.py:336@872bf041, test/conftest.py:355@872bf041. Also in tests - test/test_capget_fallback.py:295@872bf041, test/test_control_socket_peer.py:159@872bf041, test/test_state_store.py:188@872bf041. `os.killpg` availability is "Unix, not WASI, not iOS" - [Python docs: os.killpg](https://docs.python.org/3/library/os.html#os.killpg)
- `/proc` scan and `/proc/<pid>/stat` reads - test/conftest.py:369@872bf041, test/conftest.py:380@872bf041. `/proc/` appears 48 times in 14 files under test/ (git grep at 872bf041).
- SIGTERM then SIGKILL to the group - test/conftest.py:409@872bf041, test/conftest.py:417@872bf041. `signal.SIGKILL` availability is "Unix", so the attribute does not exist on Windows. "On Windows, signal() can only be called with SIGABRT, SIGFPE, SIGILL, SIGINT, SIGSEGV, SIGTERM, or SIGBREAK." - [Python docs: signal](https://docs.python.org/3/library/signal.html)
- A SIGTERM handler that reaps groups is installed at import - test/conftest.py:465@872bf041 to test/conftest.py:472@872bf041.
- unitd starts with `--control unix:<tmp>/control.unit.sock` - test/conftest.py:511@872bf041, test/conftest.py:525@872bf041. The control client uses `sock_type 'unix'` - test/unit/control.py:52@872bf041. `'unix:'` appears 28 times in 17 files under test/ (git grep at 872bf041).
- Inside `http()` (test/unit/http.py:43@872bf041) a dict maps `'unix'` to `socket.AF_UNIX` - test/unit/http.py:64@872bf041. On Windows CPython that line raises AttributeError for every request.
- `subprocess.Popen(..., start_new_session=True)` - test/conftest.py:553@872bf041. `start_new_session` and `process_group` have "availability: POSIX" - [Python docs: subprocess.Popen](https://docs.python.org/3/library/subprocess.html#subprocess.Popen)
- `ps ax -o state -o ppid` for zombies, `ps -ax -o pid -o ppid -o command`, and a third call `ps ax -O ppid` not listed in the brief - test/conftest.py:627@872bf041, test/conftest.py:777@872bf041, test/conftest.py:925@872bf041.
- SIGQUIT for a graceful stop - test/conftest.py:649@872bf041. `SIGQUIT` availability is "Unix" - [Python docs: signal](https://docs.python.org/3/library/signal.html)
- On Windows, `Popen.send_signal(SIGTERM)` is an alias for `terminate()`, which calls `TerminateProcess`. "CTRL_C_EVENT and CTRL_BREAK_EVENT can be sent to processes started with a creationflags parameter which includes CREATE_NEW_PROCESS_GROUP." - [Python docs: Popen.send_signal](https://docs.python.org/3/library/subprocess.html#subprocess.Popen.send_signal)
- `findmnt` and `waitforunmount` - test/conftest.py:739@872bf041 - are guarded by `is_findmnt` (test/conftest.py:95@872bf041), and `check_findmnt()` returns False on FileNotFoundError (test/unit/utils.py:75@872bf041 to :81). This path is already inert on Windows.
- `_count_fds()` reads `/proc/<pid>/fd`, then tries `procstat -f`, then `lsof -n -p` - test/conftest.py:870@872bf041 to test/conftest.py:887@872bf041.
- `nxt_test_in_child()` uses `fork()` and `waitpid()` - src/test/nxt_tests.c:36@872bf041, fork at src/test/nxt_tests.c:42@872bf041, waitpid at src/test/nxt_tests.c:55@872bf041.
- No workflow uses a Windows runner. CORRECTION to the brief: runners are `ubuntu-latest`, `ubuntu-24.04`, `ubuntu-24.04-arm` (.github/workflows/release-docker.yml:103@872bf041), `macos-15` (.github/workflows/build-test-macos.yml:30@872bf041) and also `macos-latest` for the unitctl `aarch64-apple-darwin` target (.github/workflows/unitctl.yml:55@872bf041). unitctl's `test` job builds `x86_64-unknown-linux-gnu` and `aarch64-apple-darwin` (.github/workflows/unitctl.yml:49-56@872bf041), a musl job builds `x86_64-unknown-linux-musl` (:109), and its `build` job adds `aarch64-unknown-linux-gnu` and `x86_64-apple-darwin` (:192-205). None is a Windows target.

Windows and Python facts:
- "If the AF_UNIX constant is not defined then this protocol is unsupported." - [Python docs: socket.AF_UNIX](https://docs.python.org/3/library/socket.html#socket.AF_UNIX)
- On Windows, `loop.create_unix_connection()` and `loop.create_unix_server()` are not supported; "The socket.AF_UNIX socket family is specific to Unix." - [Python docs: asyncio platform support](https://docs.python.org/3/library/asyncio-platforms.html#windows)
- gh-77589 "Enable AF_UNIX support in Windows" (formerly bpo-33408) is open since 2018-05-02. The first PR (14823) closed unmerged on 2019-07-17; its author wrote in 2023 that the work stopped because Windows had no AF_UNIX datagram support - [cpython issue 77589](https://github.com/python/cpython/issues/77589), [cpython PR 14823](https://github.com/python/cpython/pull/14823)
- PR 137420 "gh-77589: Add unix domain socket for Windows" opened 2025-08-05, still open with label "awaiting review", last updated 2026-07-31 - [cpython PR 137420](https://github.com/python/cpython/pull/137420)
- Winsock AF_UNIX arrived in Insider build 17063 (blog of 2017-12-19). Only SOCK_STREAM is supported. Not supported: SOCK_DGRAM, SOCK_SEQPACKET, ancillary data (descriptor and credential passing), autobind, and `socketpair`. The socket file must be deleted before a rebind - [AF_UNIX comes to Windows](https://devblogs.microsoft.com/commandline/af_unix-comes-to-windows/)
- CONFLICT: the same blog lists abstract addresses as supported, but a 2020 comment on it and a 2024 comment on gh-77589 say abstract sockets do not work - [AF_UNIX comes to Windows](https://devblogs.microsoft.com/commandline/af_unix-comes-to-windows/), [cpython issue 77589](https://github.com/python/cpython/issues/77589)
- PostgreSQL's Windows CI sets `PG_TEST_USE_UNIX_SOCKETS: 1`, so a C server and its test clients use Winsock AF_UNIX on `windows-2022` runners today - [pg-ci.yml](https://github.com/postgres/postgres/blob/master/.github/workflows/pg-ci.yml)
- `multiprocessing.connection` accepts family `'AF_PIPE'` "for a Windows named pipe", with addresses of the form `\\.\pipe\{PipeName}` - [Python docs: multiprocessing.connection.Listener](https://docs.python.org/3/library/multiprocessing.html#multiprocessing.connection.Listener)
- `GenerateConsoleCtrlEvent`: CTRL_C "cannot be limited to a specific process group"; a group made with CREATE_NEW_PROCESS_GROUP includes all descendants of the root; "Only those processes in the group that share the same console as the calling process receive the signal." The default handler calls ExitProcess - [Microsoft Learn: GenerateConsoleCtrlEvent](https://learn.microsoft.com/en-us/windows/console/generateconsolectrlevent)
- `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE` "Causes all processes associated with the job to terminate when the last handle to the job is closed." Children escape the job only with `CREATE_BREAKAWAY_FROM_JOB` under `JOB_OBJECT_LIMIT_BREAKAWAY_OK`, or under `JOB_OBJECT_LIMIT_SILENT_BREAKAWAY_OK` - [Microsoft Learn: JOBOBJECT_BASIC_LIMIT_INFORMATION](https://learn.microsoft.com/en-us/windows/win32/api/winnt/ns-winnt-jobobject_basic_limit_information)
- `taskkill /t` "Ends the specified process and any child processes started by it"; `/f` ends processes forcefully - [Microsoft Learn: taskkill](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/taskkill)
- psutil: `num_handles()` returns handles in use, availability Windows; `num_fds()` is UNIX only; `children(recursive=True)` returns all descendants - [psutil docs/api.rst](https://github.com/giampaolo/psutil/blob/master/docs/api.rst)

### Inferences
- Import hygiene comes first: move `import fcntl` and its use under `sys.platform != 'win32'`, use `pytest.importorskip` or platform markers for the pwd/grp modules, and build the http.py socket table with `getattr(socket, 'AF_UNIX', None)`. Without this nothing runs. Confidence: high.
- Control channel for CI: run unitd with `--control 127.0.0.1:<free port>` and set the client `sock_type` to `ipv4`. This needs no new client code. The TCP control socket has no peer-credential check, so bind loopback only and pick a random port per run. Confidence: high for CI use; the product's own channel is a separate decision.
- If the product ships an AF_UNIX control socket on Windows, the stock Python client cannot reach it until CPython ships gh-77589. Python 3.15 is past its feature freeze, so 3.16 (late 2027) is the earliest stdlib option. Test it with a small C client or ctypes in the meantime. Confidence: medium (release-cadence reasoning, schedule not fetched).
- If the product ships a named pipe, a Python test client can open `\\.\pipe\<name>` with `open(..., 'r+b', buffering=0)` for a byte stream. `multiprocessing.connection` expects its own message-mode framing, so it is not a raw HTTP transport. Confidence: medium; verify by experiment.
- Process trees: put unitd and all descendants in a Job Object with KILL_ON_JOB_CLOSE. Children join by default, so a job assigned before unitd forks its workers covers the whole tree. Use a small launcher that is assigned to the job before it starts unitd, to avoid a race. Use psutil `children(recursive=True)` for liveness checks and `taskkill /T /F` only as a fallback, because `/T` follows parent links that break when an intermediate parent exits. Confidence: medium.
- Better still, the Windows port of unitd should create its own job for its workers. Then the harness has one process to stop. Confidence: medium.
- Graceful stop: do not rely on CTRL_BREAK. It needs a shared console, and a service has none. Use a stop request on the control API or a named event that the main process waits on, and keep job close as the hard stop. Confidence: medium-high.
- The SIGTERM reaper at test/conftest.py:465 has no Windows trigger: a runner cancel uses TerminateProcess, which no handler sees. A job with KILL_ON_JOB_CLOSE held by pytest covers the same case. Confidence: medium.
- The zombie wait at test/conftest.py:627 has no Windows meaning; skip it there. Confidence: medium-high.
- Descriptor leak checks: use `psutil.Process.num_handles()` deltas with a tolerance. Handle counts include threads, events and keys, so absolute values do not match Linux descriptor counts. Keep exact-count tests Linux-only. Confidence: medium.
- C test table: replace `fork()` in `nxt_test_in_child()` with a re-run of the test binary through `CreateProcess`, passing the test name, and read the exit code. Confidence: medium.

### Gaps
- Abstract AF_UNIX addresses on current Windows: bind one on `windows-2025` and record the result.
- Whether `curl.exe` on the images supports `--unix-socket` on Windows was not checked; it would be a cheap AF_UNIX test client.
- The CPython 3.15 schedule (PEP 790) was not fetched; the "past feature freeze" claim rests on the annual cadence.
- CTRL_BREAK delivery inside a GitHub Actions Windows step was not tested.

## 3.6 Wine in CI

### Takeaway
Running a mingw-w64 build of FreeUnit's tests under Wine is possible but not worth it as a gate. Upstream Wine has no AF_UNIX (only a Wine Staging patchset), and the server projects closest to FreeUnit test on real Windows runners.
Wine 11.0 is the current stable release (tagged 2026-01-13). Development is at 11.19 (2026-10-02).

### Cited Findings
- Wine release tags: wine-10.0 on 2025-01-21, wine-11.0 on 2026-01-13, development wine-11.19 on 2026-10-02 (WineHQ GitLab API) - [wine-11.0 tag](https://gitlab.winehq.org/wine/wine/-/tags/wine-11.0), [wine-11.19 tag](https://gitlab.winehq.org/wine/wine/-/tags/wine-11.19), [wine-10.0 tag](https://gitlab.winehq.org/wine/wine/-/tags/wine-10.0)
- Wine Staging 11.16 (article of 2026-08-23) adds a ws2_32 AF_UNIX patchset of nine patches covering bind, connect, listen, accept, sendto and recvfrom. The article says upstream Wine does not have it - [Wine Staging 11.16 Adds AF_UNIX Socket Support](https://www.linuxcompatible.org/story/wine-staging-1116-adds-afunix-socket-support-and-shader-improvements/)
- GnuTLS runs its mingw test suite under Wine in GitLab CI: it registers Wine for `MZ` binaries through binfmt_misc, runs `wineboot --init`, then `make -C tests check`, with a 3-hour job timeout. The build step passes `--disable-full-test-suite` - [gnutls .gitlab-ci.yml, lines 717 to 737](https://gitlab.com/gnutls/gnutls/-/blob/master/.gitlab-ci.yml)
- cross-rs runs `cargo test` binaries for `x86_64-pc-windows-gnu` through Wine (`CARGO_TARGET_X86_64_PC_WINDOWS_GNU_RUNNER` set to `wine`) - [cross-rs Dockerfile.x86_64-pc-windows-gnu](https://github.com/cross-rs/cross/blob/main/docker/Dockerfile.x86_64-pc-windows-gnu)
- curl cross-builds with mingw on Linux ("linux-mingw" jobs, 0m53s to 4m5s) and runs its tests on Windows runners; its Windows workflow does not mention Wine - [curl windows.yml](https://github.com/curl/curl/blob/master/.github/workflows/windows.yml), [curl run 34119299124](https://github.com/curl/curl/actions/runs/34119299124)
- libuv cross-builds with mingw on Ubuntu and tests the binaries on `windows-2022`, not under Wine - [libuv CI-win.yml](https://github.com/libuv/libuv/blob/v1.x/.github/workflows/CI-win.yml)

### Inferences
- Wine adds little here. Real Windows runners are free for this public repository, and the close precedents (curl, libuv, PostgreSQL) test on them. Confidence: high.
- A mingw-w64 build cannot cover the PHP module, because the official PHP Windows builds use MSVC (section 3). Wine would test a different toolchain from the one shipped. Confidence: medium.
- Keep one optional Linux job that cross-compiles the core with `x86_64-w64-mingw32` and `-Werror`. It gives compile feedback in 1 to 4 minutes, as curl and libuv show. Do not run the server tests under Wine. Confidence: medium-high.

### Gaps
- Wine's IOCP, AcceptEx and named pipe behaviour under a server workload was not researched from primary sources. A one-hour experiment: run a cross-built unitd under Wine 11.0 with a TCP control socket and the C test table.
- Whether a Wine 11.0.x maintenance release exists was not checked.

## 3.7 CI plan sketch and a Windows-capable harness

### Takeaway
Add one `windows.yml` with two pull-request jobs (core build plus C tests on `windows-2025`; PHP module plus a pytest smoke subset on `windows-2022`) and nightly jobs for arm64, a wider pytest subset, packaging and a mingw cross-compile. Keep the Windows jobs non-required until they are stable.
Start the harness with TCP loopback control and import hygiene. Then add Job Objects, psutil and a control-API stop.

### Cited Findings
- `windows-2025` has VS 2026 and also the v14.44 (VS 2022) toolset component; `windows-2022` has VS 2022 17.14 - [Windows2025-VS2026-Readme.md](https://github.com/actions/runner-images/blob/main/images/windows/Windows2025-VS2026-Readme.md), [Windows2022-Readme.md](https://github.com/actions/runner-images/blob/main/images/windows/Windows2022-Readme.md)
- php-windows-builder builds PHP 8.4 and 8.5 with VS17 on `windows-2022` and has `run-tests` and `test-runner` inputs - [php/php-windows-builder](https://github.com/php/php-windows-builder)
- vcpkg's `x-gha` is removed; files, NuGet and GitHub Packages remain - [Microsoft Learn: binary caching configuration](https://learn.microsoft.com/en-us/vcpkg/reference/binarycaching)
- PostgreSQL shards its Windows test leg into 2 slices with a 60-minute timeout - [pg-ci.yml](https://github.com/postgres/postgres/blob/master/.github/workflows/pg-ci.yml)
- Candidate modules for a first Windows subset exist at 872bf041, for example test/test_configuration.py:1@872bf041, test/test_routing.py:1@872bf041, test/test_return.py:1@872bf041, test/test_rewrite.py:1@872bf041, test/test_static.py:1@872bf041, test/test_variables.py:1@872bf041, test/test_access_log.py:1@872bf041, test/test_php_basic.py:1@872bf041, test/test_php_application.py:1@872bf041, test/test_tls.py:1@872bf041. Of 110 `test_*.py` files, those with isolation, chroot, users, Unix listeners or abstract sockets in their names are Linux-only by design (for example test/test_asgi_application_unix_abstract.py:1@872bf041, test/test_static_chroot.py:1@872bf041, test/test_php_isolation.py:1@872bf041).

### Inferences
Proposed jobs (durations are estimates, not measurements):

| Job | Runner | Trigger | What runs | Cache | Est. minutes |
|---|---|---|---|---|---|
| win-core | `windows-2025` (VS 2026) | every PR touching src/, auto/, configure, the workflow | clang-cl (MSVC ABI, section 1.0) build of unitd and libunit with `-WX`; `build/tests` C table | vcpkg files cache | 5 to 10 |
| win-php | `windows-2022` (VS 2022) | every PR touching src/, the PHP module, test/ | PHP 8.4 or 8.5 NTS x64 module (the TS variant builds in win-pytest-wide, section 1.0); pytest smoke: configuration, routing, return, static, PHP basic | vcpkg + PHP devel pack or self-built PHP keyed on version, arch, TS | 10 to 20 |
| win-arm64 | `windows-11-arm` or `windows-11-vs2026-arm` | nightly | core build plus C table | vcpkg arm64 triplet cache | 8 to 12 |
| win-pytest-wide | `windows-2025` | nightly, 2 to 4 shards | every module not marked Linux-only, TLS included; PHP module built against the TS zip | as win-core | 15 to 20 per shard |
| win-package | `windows-2025` | nightly and tags | zip (later MSI) with unitd, modules, vcpkg DLLs; smoke start of the packaged binary | as win-core | 5 |
| mingw-cross | `ubuntu-24.04` | every PR | `x86_64-w64-mingw32` compile of the core, warnings as errors, no tests | apt only | 1 to 4 |

- Triggers: run win-core and mingw-cross on every relevant PR from day one, but as non-required checks for about two weeks of green runs, then make win-core required. Keep win-php non-required until the PHP build path is settled. Confidence: medium.
- Stages of tests: stage 0 is compile only (mingw-cross, win-core build). Stage 1 adds the C table once `nxt_test_in_child()` no longer forks. Stage 2 adds the pytest smoke over TCP control. Stage 3 adds PHP tests. Stage 4 adds TLS, proxy and access log modules. Confidence: medium.
- Caching: vcpkg with `VCPKG_BINARY_SOURCES=clear;files,<dir>,readwrite` plus `actions/cache` keyed on the vcpkg commit, triplet and manifest hash; separate keys for x64 and arm64 (curl's windows.yml notes that the cache cannot be shared between arm and intel, citing https://github.com/actions/cache/issues/1622). Cargo: cache `~/.cargo/registry`, `~/.cargo/git` and `target`. PHP: cache the devel pack or the self-built PHP keyed on PHP version, arch, TS flag and toolset. Confidence: medium.
- Harness plan, in order:
  1. Import hygiene (fcntl, pwd, grp, `socket.AF_UNIX` lookups) and a `windows` or `unix_only` pytest marker applied by file. Confidence: high.
  2. One platform module in test/unit/ with `spawn()`, `stop_graceful()`, `kill_tree()`, `alive()`, `children()`, `count_handles()` and `control_address()`. conftest.py calls only these. Confidence: high.
  3. Control over TCP loopback on Windows (`--control 127.0.0.1:<port>`, client `sock_type 'ipv4'`). Add a dedicated test for the product's own local channel (AF_UNIX or named pipe) once it exists. Confidence: high.
  4. Process trees through a Job Object with KILL_ON_JOB_CLOSE (ctypes, or pywin32 if the image has it), psutil for scans, `taskkill /T /F` as a last resort. Confidence: medium.
  5. Graceful stop through a control API request or a named event; hard stop by closing the job. Confidence: medium.
  6. Handle-count deltas through psutil in place of descriptor counts; exact-count tests stay Linux-only. Confidence: medium.
  7. Collect unit.log and any crash dumps as artifacts on failure; set `DIE_ON_UNHANDLED_EXCEPTION` on the job so a crash ends the process instead of opening a dialog. Confidence: medium.
- Budget: with free standard runners, a full PR run adds two Windows jobs of about 5 to 20 minutes each and no cost. Confidence: medium.

### Gaps
- The FreeUnit-specific durations in the table are guesses. One scratch run of win-core and win-php settles them.
- The product's choice of local control channel on Windows (AF_UNIX, named pipe or TCP loopback) belongs to the architecture part and changes step 3.
- Whether pywin32 is preinstalled on the Windows images was not checked; ctypes needs no install.

## 4.1 Package formats and package managers

### Takeaway
Ship a portable zip first. Caddy, PHP for Windows, Apache Lounge and nginx all ship zips, and winget, Scoop and Chocolatey all consume them. winget accepts a zip with a nested portable exe, requires InstallerSha256, has no signing requirement, and scans every submission with Microsoft Defender and other antivirus engines. Scoop Main is not possible today (FreeUnit has 57 stars; Main asks for 500 stars and 150 forks), so start with an own bucket. An MSI built with WiX v7 costs nothing for a project without revenue; defer it until a service mode exists. Skip MSIX.

### Cited Findings
- The current WiX release is v7.0.0 (2026-04-06). Earlier: v7.0.0-rc.2 (2026-03-05), v7.0.0-rc.1 (2026-02-06), v6.0.2 (2025-08-28), v6.0.1 (2025-06-06) ([WiX Toolset releases](https://github.com/wixtoolset/wix/releases)).
- "WiX v3, v4 and v5 are out of community support." wixtoolset.org now redirects to FireGiant ([FireGiant WiX Toolset page](https://www.firegiant.com/wixtoolset/)).
- Open Source Maintenance Fee (OSMF): "organizations that generate more than $10,000 in annual revenue ... are required to sponsor" the wixtoolset GitHub organization. The fee was "first introduced in WiX v6", and "EULA acceptance enforcement" was "implemented in WiX v7". The source code "remains freely available under the terms of the LICENSE". The fetched page states no amount ([FireGiant docs: Open Source Maintenance Fee](https://docs.firegiant.com/wix/osmf/)).
- WiX release notes: "use of this project requires an Open Source Maintenance Fee" for revenue-generating use; v7 releases point to an OSMF EULA ([WiX Toolset releases](https://github.com/wixtoolset/wix/releases)).
- MSIX supports services from Windows 10 version 2004. A package with a service "will require admin privileges to install". "We currently do not support services with dependencies outside the package." Adding a service needs a restricted capability ([Microsoft Learn: Converting an installer with services](https://learn.microsoft.com/en-us/windows/msix/packaging-tool/convert-an-installer-with-services)).
- The desktop6:Service element allows StartAccount values "localSystem", "localService" or "networkService" only, and "requires the packagedServices or localSystemServices restricted capability" ([Microsoft Learn: desktop6:Service](https://learn.microsoft.com/en-us/uwp/schemas/appxpackage/uapmanifestschema/element-desktop6-service)).
- The MSIX manifest schema has a desktop2:FirewallRules element, so a package can declare firewall rules ([Microsoft Learn: desktop2:FirewallRules](https://learn.microsoft.com/en-us/uwp/schemas/appxpackage/uapmanifestschema/element-desktop2-firewallrules)).
- winget installer schema 1.12.0: portable packages since v1.3, zip since v1.5. NestedInstallerType gives "the installer type of the file within the archive"; NestedInstallerFiles needs RelativeFilePath; PortableCommandAlias is valid only "when NestedInstallerType is 'portable'". Architecture values: "x86, x64, arm, arm64, neutral". "InstallerSha256 - Sha256 is required." Package dependencies "must come from the same source" ([winget-pkgs installer schema 1.12.0](https://github.com/microsoft/winget-pkgs/blob/master/doc/manifest/schema/1.12.0/installer.md)).
- winget submission: an automated pipeline validates the manifest and runs "tests against the installer and installed binaries"; then "your submission will be manually reviewed by a moderator". Expectations: installer "virus free", installs and uninstalls "for both administrators and non-administrators", "supports non-interactive modes", "comes directly from the publisher's website" ([Microsoft Learn: Submit your manifest to the repository](https://learn.microsoft.com/en-us/windows/package-manager/package/repository)).
- winget validation labels include Validation-Defender-Error ("During dynamic testing, Microsoft Defender reported a problem"), Binary-Validation-Error ("Each submission ... is run through several antivirus programs"), URL-Validation-Error (HTTP error or failed URL reputation test), Validation-Indirect-URL (redirectors not allowed) and Validation-VCRuntime-Dependency ("dependency on the C++ runtime that could not be resolved") ([Microsoft Learn: Submit your manifest to the repository](https://learn.microsoft.com/en-us/windows/package-manager/package/repository)).
- winget policy 1.1.4: "The InstallerUrl must be the ISV's release location for the Product. Products from download websites will not be allowed." Policy 1.2.2 bans malware. The policies contain no requirement that installers be Authenticode-signed ([Microsoft Learn: Windows Package Manager repository policies](https://learn.microsoft.com/en-us/windows/package-manager/package/windows-package-manager-policies); [Microsoft Learn: Create your package manifest](https://learn.microsoft.com/en-us/windows/package-manager/package/manifest)).
- Caddy in winget: CaddyServer.Caddy 2.11.7 is InstallerType zip, NestedInstallerType portable, caddy.exe, with x64 and arm64 zips from GitHub Releases ([winget-pkgs CaddyServer.Caddy 2.11.7](https://github.com/microsoft/winget-pkgs/blob/master/manifests/c/CaddyServer/Caddy/2.11.7/CaddyServer.Caddy.installer.yaml)).
- Apache Lounge in winget: ApacheLounge.httpd 2.4.68 is zip plus portable Apache24/bin/httpd.exe, x86 and x64, with PackageDependencies Microsoft.VCRedist.2015+.x64 (and .x86) ([winget-pkgs ApacheLounge.httpd 2.4.68](https://github.com/microsoft/winget-pkgs/blob/master/manifests/a/ApacheLounge/httpd/2.4.68/ApacheLounge.httpd.installer.yaml)).
- PHP in winget: PHP.PHP.8.5 (manifest for 8.5.8) is zip plus portable php.exe with PortableCommandAlias php, x86 and x64 zips from downloads.php.net, and the same VCRedist dependencies ([winget-pkgs PHP.PHP.8.5 8.5.8](https://github.com/microsoft/winget-pkgs/blob/master/manifests/p/PHP/PHP/8/5/8.5.8/PHP.PHP.8.5.installer.yaml)).
- nginx is not in winget under the nginx publisher folder, which holds only nginx-prometheus-exporter. RoadRunner was not found under the RoadRunner, spiral or SpiralScout folders (GitHub API listing, 2026-10-07) ([winget-pkgs manifests/n/nginx](https://github.com/microsoft/winget-pkgs/tree/master/manifests/n/nginx)).
- Scoop Main criteria: "a reasonably well-known and widely used developer tool (e.g. if it's a GitHub project, it should have at least 500 stars and 150 forks)", "the latest stable version", "the full version", "a fairly standard install", "a non-GUI tool" ([Scoop wiki: Criteria for including apps in the main bucket](https://github.com/ScoopInstaller/Scoop/wiki/Criteria-for-including-apps-in-the-main-bucket)).
- freeunitorg/freeunit has 57 stars and 8 forks; the archived nginx/unit has 5,538 stars and 382 forks (GitHub API, 2026-10-07) ([freeunitorg/freeunit](https://github.com/freeunitorg/freeunit)).
- Scoop Main holds caddy 2.11.7 (64bit and arm64 zips), nginx 1.31.6 (https://nginx.org/download/nginx-1.31.6.zip), php 8.5.11 (64bit and 32bit; it suggests extras/vcredist2022) and apache 2.4.69 (a Win64 VS18 zip fetched from a fossies.org mirror). There is no roadrunner.json in Main or Extras ([Scoop Main caddy.json](https://github.com/ScoopInstaller/Main/blob/master/bucket/caddy.json); [nginx.json](https://github.com/ScoopInstaller/Main/blob/master/bucket/nginx.json); [php.json](https://github.com/ScoopInstaller/Main/blob/master/bucket/php.json); [apache.json](https://github.com/ScoopInstaller/Main/blob/master/bucket/apache.json)).
- Chocolatey community repository: "all new versions of packages are human reviewed prior to approval", except trusted packages, which "only go through automated moderation review". Reviewers check whether the maintainer "has distribution rights" for embedded binaries ([Chocolatey docs: Moderation](https://docs.chocolatey.org/en-us/community-repository/moderation/)).
- Chocolatey "is also verified against VirusTotal - 60-70 ramped up anti-virus scanners", covering "packages and any binaries they contain or download" ([Chocolatey docs: Security](https://docs.chocolatey.org/en-us/information/security)).
- Package Internalizer is a Chocolatey for Business (C4B) feature ([Chocolatey docs: Package Internalizer](https://docs.chocolatey.org/en-us/features/package-internalizer)).
- Chocolatey package pages exist for caddy, nginx, php, apache-httpd and roadrunner (HTTP 200; a missing id returns 404). Their maintainers and contents were not reviewed ([Chocolatey: caddy](https://community.chocolatey.org/packages/caddy); [roadrunner](https://community.chocolatey.org/packages/roadrunner)).
- nginx for Windows ships as a zip; PHP for Windows lists "Zip" downloads per build ([PHP downloads for Windows](https://www.php.net/downloads.php?os=windows); [Scoop Main nginx.json](https://github.com/ScoopInstaller/Main/blob/master/bucket/nginx.json)).

### Inferences
- High: a zip with a nested portable unitd.exe is the established pattern. Copy the ApacheLounge.httpd manifest shape: zip, NestedInstallerType portable, PortableCommandAlias for unitd and unitctl, PackageDependencies on Microsoft.VCRedist.2015+.x64 (and .arm64 later).
- High: GitHub Releases of freeunitorg/freeunit satisfy winget policy 1.1.4, as Caddy shows. Do not use a redirector or a mirror URL.
- High: Scoop Main will refuse FreeUnit at 57 stars. Publish an own bucket (a repository with bucket/freeunit.json) and suggest extras/vcredist2022 as php.json does. Re-apply to Main or Extras once the criteria are met.
- Medium: WiX v7 is free for FreeUnit while the project has no revenue. Risk: if a maintainer builds release MSIs as part of work for an employer with more than $10,000 revenue, that employer may count as the user. Build MSIs only in the project's own CI.
- Medium: MSIX does not fit a developer tool that should install without admin rights. A packaged service needs admin install and a restricted capability, and services cannot depend on anything outside the package.
- Medium: Chocolatey can wait. Its human review adds latency per release, and the free path (winget plus Scoop) already covers developers.

### Gaps
- The OSMF FAQ. A search index showed FAQ lines ("based on whether the organization generates revenue, not whether a specific product is sold"; "per organization"), but the fetched page had no FAQ. Read https://opensourcemaintenancefee.org/ (consumers section) and the WiX v7 OSMF EULA before shipping an MSI.
- How WiX v7 EULA acceptance works in unattended CI builds. Read the v7 docs.
- Whether MSIX file-system virtualization breaks a server that writes state, logs and sockets next to its binaries. Read the MSIX "prepare to package" limitations.
- RoadRunner's Windows packaging and winget presence. Search winget-pkgs for its PackageIdentifier.
- Contents of the Chocolatey packages (maintainer, version, embedded or downloaded binaries).

## 4.2 Code signing in 2026

### Takeaway
Microsoft renamed Trusted Signing to Artifact Signing. Basic costs $9.99 per month for 5,000 signatures. Public Trust certificates go only to organizations in a list of countries and to individuals in the United States or Canada, and a paid Azure subscription is required. A volunteer project with no legal entity cannot use it directly. SignPath Foundation signs OSI-licensed projects for free when CI builds them from a public repository, but the certificate names SignPath Foundation as publisher. Certum sells open-source certificates from 25 EUR. Since 2026-03-01 public code signing certificates last at most 460 days, and since 2023-06-01 keys must live in hardware. EV gives no SmartScreen benefit.

### Cited Findings
- Prices below are as published on 2026-10-07 ([Microsoft Learn: Change the account SKU](https://learn.microsoft.com/en-us/azure/artifact-signing/how-to-change-sku)).
- The service is now "Artifact Signing (formerly Trusted Signing)" ([Microsoft Learn: SmartScreen reputation for Windows app developers](https://learn.microsoft.com/en-us/windows/apps/package-and-deploy/smartscreen-reputation)); the old pricing URL now shows "Artifact Signing" ([Azure pricing: Artifact Signing](https://azure.microsoft.com/en-us/pricing/details/trusted-signing/)).
- SKUs: Basic "$9.99 per account" per month, 5,000 signatures; Premium "$99.99 per account", 100,000 signatures; "$0.005 per signature" after the quota; Basic has "1 of each available type" of certificate profile, Premium 10 ([Microsoft Learn: Change the account SKU](https://learn.microsoft.com/en-us/azure/artifact-signing/how-to-change-sku)). The Azure pricing page itself rendered "$-" placeholders and calls its prices estimates ([Azure pricing: Artifact Signing](https://azure.microsoft.com/en-us/pricing/details/trusted-signing/)).
- Billing "isn't calculated on a pro rata basis". Free, trial and sponsored Azure subscriptions are not supported ([Microsoft Learn: Artifact Signing FAQ](https://learn.microsoft.com/en-us/azure/artifact-signing/faq)).
- Eligibility: "Public Trust certificates are available to organizations in the United States, Canada, the European Union, the United Kingdom, Australia, New Zealand, Japan, South Korea, Singapore, Switzerland, Norway, and Israel. Individual developers must be located in the United States or Canada." ([Microsoft Learn: Quickstart: Set up Artifact Signing](https://learn.microsoft.com/en-us/azure/artifact-signing/quickstart)).
- Individual validation uses the Azure billing account, which "must have an Account Type of Individual". The legal name and address must match a government ID, and the city, state and country appear on the certificate. Processing takes "from 1 to 20 business days". Documents must be "issued within the previous 12 months". The quickstart states no minimum years of business history and no open-source path ([Microsoft Learn: Quickstart: Set up Artifact Signing](https://learn.microsoft.com/en-us/azure/artifact-signing/quickstart)).
- No custom CN or O: the CN "must always be the legal entity's validated name". "Artifact Signing doesn't issue Extended Validation (EV) certificates. There's no plan to issue EV certificates in the future." ([Microsoft Learn: Artifact Signing FAQ](https://learn.microsoft.com/en-us/azure/artifact-signing/faq)).
- Certificates "are renewed daily and are valid for only 72 hours". Time stamp countersigning keeps signatures valid after expiry; the TSA is http://timestamp.acs.microsoft.com. The Artifact Signing CAs "are scheduled for removal from the Common CA Database (CCADB)" but "remain on the Windows Platform Trust List" ([Microsoft Learn: Artifact Signing certificate management](https://learn.microsoft.com/en-us/azure/artifact-signing/concept-certificate-management)).
- SmartScreen with Artifact Signing: "SmartScreen reputation builds up automatically. The prompt stops appearing once the file hash has sufficient download history." ([Microsoft Learn: Artifact Signing FAQ](https://learn.microsoft.com/en-us/azure/artifact-signing/faq)).
- The GitHub Action Azure/trusted-signing-action now resolves to Azure/artifact-signing-action (GitHub API, 2026-10-07). Microsoft lists "SignTool, PowerShell, GitHub Actions, or the Artifact Signing SDK" for signing outside Azure Pipelines ([Azure/artifact-signing-action](https://github.com/Azure/artifact-signing-action); [Microsoft Learn: Artifact Signing FAQ](https://learn.microsoft.com/en-us/azure/artifact-signing/faq)).
- EV: "EV certificates no longer bypass SmartScreen. Years ago, signing files with an Extended Validation (EV) code signing certificate would result in positive SmartScreen reputation by default, but this behavior no longer exists. ... Paying a premium for EV solely to avoid SmartScreen warnings is no longer justified." The page gives no date for the change ([Microsoft Learn: SmartScreen reputation for Windows app developers](https://learn.microsoft.com/en-us/windows/apps/package-and-deploy/smartscreen-reputation)).
- Ballot CSC-31: "reduce the maximum certificate validity, from 39 months to 460 days, effective March 1st, 2026". Discussion 2025-09-16 to 2025-10-06, voting 2025-10-06 to 2025-10-13, giving CSBR v3.10.0 ([CA/Browser Forum: Ballot CSC-31](https://cabforum.org/2025/11/17/ballot-csc-31-maximum-validity-reduction)).
- CSBR 6.3.2: "For Code Signing Certificates issued on or after March 1st, 2026, the validity period MUST NOT exceed 460 days." CSBR 6.2.7.4.2: "Effective June 1, 2023, for Code Signing Certificates, CAs SHALL ensure that the Subscriber's Private Key is generated, stored, and used in a suitable Hardware Crypto Module" ([CA/Browser Forum Code Signing Baseline Requirements](https://github.com/cabforum/code-signing/blob/main/docs/CSBR.md)).
- Certum shop: Open Source Code Signing in the Cloud from 49.00 EUR, Open Source set from 69.00 EUR, Open Source code from 25.00 EUR; Standard from 139 to 209 EUR; EV from 329 to 379 EUR. Certum applies a "459 days" maximum from 2026-02-27, with free reissues for 2 to 3 year purchases. The page does not state open-source eligibility rules ([Certum shop: Code Signing](https://shop.certum.eu/code-signing.html)).
- Reseller prices are indicative only: one 2026 comparison quotes OV at about $64.50 (SSL.com), $220 (Sectigo) and $439 (DigiCert) per year, and EV at $290 to $500 (Sectigo) and about $644 (DigiCert). These figures come from a search summary, not vendor pages ([sslinsights: DigiCert vs Sectigo code signing](https://sslinsights.com/digicert-vs-sectigo-code-signing/)).
- SignPath Foundation terms: "an OSI-approved Open Source license without commercial dual-licensing"; "no proprietary, non open-source component"; binaries must be "a valid, automated build resulting from the source code at the noted source code repository"; "The code signing certificate is issued to SignPath Foundation. This means that SignPath Foundation is the publisher of the OSS project."; a "Code signing policy" page with the attribution "Free code signing provided by SignPath.io, certificate by SignPath Foundation"; MFA for all team members; Authors, Reviewers and Approvers roles; no hacking tools; software must "provide uninstallation capabilities" ([SignPath Foundation terms](https://signpath.org/terms)).
- SignPath's GitHub Action SignPath/github-action-submit-signing-request is active: latest release v2 (2025-10-23), last push 2026-09-10 (GitHub API, 2026-10-07) ([SignPath/github-action-submit-signing-request](https://github.com/SignPath/github-action-submit-signing-request)).

### Inferences
- High: FreeUnit cannot use Artifact Signing today unless a legal entity in a listed country (or a fiscal host) applies, or a maintainer resident in the United States or Canada applies as an individual. The individual route prints that person's legal name and city on every binary.
- Medium-high: apply to SignPath Foundation first. Apache-2.0 qualifies, GitHub Actions builds qualify, and the cost is zero. The project must publish a code signing policy, enforce MFA and name approvers. Users will see "SignPath Foundation" as publisher, not FreeUnit.
- Medium: the cheapest paid route is Certum open source (25 to 69 EUR). It names an individual, and the 459-day limit means a yearly renewal.
- High: do not buy EV. Microsoft says it no longer changes SmartScreen behaviour.
- Medium: whichever route, sign every PE file (exe and dll) and the MSI, and timestamp every signature. Short-lived Artifact Signing certificates and 460-day CA certificates both depend on the timestamp.

### Gaps
- Whether SignPath will sign third-party binaries that FreeUnit redistributes but does not build (PHP DLLs, OpenSSL DLLs), and how long approval takes. Ask through the SignPath Foundation application.
- Whether the shared SignPath Foundation certificate already carries SmartScreen reputation. Test with a signed preview on a clean VM.
- The date of the EV change. Secondary sources say 2024; no Microsoft page fetched here gives a date.
- Whether Artifact Signing still needs a minimum business history for organizations. The current quickstart does not say. Read the identity validation docs or ask Azure support.
- Certum open-source eligibility (individual only? project URL?) and whether its cloud signing (SimplySign) can run unattended in CI.
- Vendor list prices for OV and EV certificates (SSL.com, Sectigo, DigiCert pricing pages).

## 4.3 SmartScreen, Smart App Control and Defender

### Takeaway
No certificate buys instant SmartScreen trust any more. Signing still pays in two ways: reputation can carry over between releases signed by one identity, and Smart App Control on Windows 11 lets validly signed files run when its cloud service has no verdict. Unsigned files show "Windows protected your PC" and start from zero on every release. Go toolchain binaries have repeatedly hit Defender false positives; plan a submission step per release.

### Cited Findings
- SmartScreen weighs "Publisher reputation" and "File hash reputation". "Even when signed, a newly created binary could still show a SmartScreen warning". "When a file is not signed, SmartScreen reputation must build for each new version of your files, starting with zero reputation. Reputation cannot transfer from previous versions unless both were signed using the same publisher identity." ([Microsoft Learn: SmartScreen reputation for Windows app developers](https://learn.microsoft.com/en-us/windows/apps/package-and-deploy/smartscreen-reputation)).
- Unsigned or self-signed: "Windows protected your PC"; "User must choose 'Run anyway'". "Enterprise policy can prevent continuation entirely." ([same page](https://learn.microsoft.com/en-us/windows/apps/package-and-deploy/smartscreen-reputation)).
- "There is no exact threshold, but it can take several weeks and hundreds of clean installs from a wide audience." "There is no need (or mechanism) to manually submit a file for SmartScreen reputation review for consumer endpoints." Enterprise admins may submit files through https://www.microsoft.com/en-us/wdsi/filesubmission ([same page](https://learn.microsoft.com/en-us/windows/apps/package-and-deploy/smartscreen-reputation)).
- "Smart App Control will block execution of unsigned files unless the file has a positive reputation. Smart App Control signature checks apply to all executable files, not just those downloaded from the Internet." ([same page](https://learn.microsoft.com/en-us/windows/apps/package-and-deploy/smartscreen-reputation)).
- Smart App Control first asks its cloud service. If the service cannot decide, it checks the signature: "If the app is unsigned, or the signature is invalid, Smart App Control will consider it untrusted and block it". "Recent Windows updates allow Smart App Control to be re-enabled without requiring a clean installation." Listed reasons for it being off include "Your device is enterprise-managed or developer-mode has been configured" ([Microsoft Support: Smart App Control FAQ](https://support.microsoft.com/en-us/topic/what-is-smart-app-control-285ea03d-fa88-4d56-882e-6698afdb7003)).
- Go FAQ: virus scanner alarms on Go binaries are "a common occurrence, especially on Windows machines, and is almost always a false positive. Commercial virus scanning programs are often confused by the structure of Go binaries" ([Go FAQ](https://go.dev/doc/faq)).
- Antivirus tools flagged Go toolchain files several times. Issue titles: link.exe "detected as trojan" (2019); the Go 1.15.3 x86 MSI "detected as malware by Windows Defender" (2021); go.exe from the 1.17 MSI "classified as malware" (2021); buildid.exe "marked as Win32/Uwamson.A!ml by Windows Defender" (2021); pack.exe "PUA:Win32/Caypnamer.A!ml" (2022) ([golang/go#30644](https://github.com/golang/go/issues/30644); [issue 45415](https://github.com/golang/go/issues/45415); [issue 47807](https://github.com/golang/go/issues/47807); [issue 48224](https://github.com/golang/go/issues/48224); [issue 51514](https://github.com/golang/go/issues/51514)).
- A GitHub issue search for "Defender" in caddyserver/caddy and roadrunner-server/roadrunner found no false-positive reports (2026-10-07). Absence in search is not proof ([caddyserver/caddy issues](https://github.com/caddyserver/caddy/issues)).
- winget runs Defender during dynamic testing and several antivirus engines on the installer; false positives go to the Defender submission portal ([Microsoft Learn: Submit your manifest to the repository](https://learn.microsoft.com/en-us/windows/package-manager/package/repository)). Chocolatey checks packages and their binaries against VirusTotal ([Chocolatey docs: Security](https://docs.chocolatey.org/en-us/information/security)).

### Inferences
- High: sign every exe and dll, not only unitd.exe. Smart App Control checks all executables, including files installed by winget or Scoop and files that never had a download mark.
- Medium: expect SmartScreen prompts for the first weeks of each signing identity. Say so in the Windows install docs, as Microsoft recommends for early adopters.
- Medium: add a release checklist step: submit new binaries to the Microsoft Security Intelligence portal as a software developer and check VirusTotal before announcing. A new unsigned server that spawns processes, loads DLLs and opens sockets fits common heuristics.

### Gaps
- Whether PHP for Windows binaries are Authenticode-signed. If not, Smart App Control may block php8*.dll loaded by a signed unitd.exe. Test: Get-AuthenticodeSignature on the official zip contents, then load them on a Smart App Control VM.
- Whether winget and Scoop downloads carry the Mark of the Web, which triggers SmartScreen. Test on a clean Windows 11 VM.
- How Smart App Control treats an unsigned DLL loaded by a signed exe. Test on a VM with Smart App Control on.

## 4.4 Running model: console process, service, firewall, ports, Ctrl+C

### Takeaway
Start as a console process that the developer starts in a terminal and stops with Ctrl+C. nginx for Windows works only this way; Apache httpd adds a native service with `httpd.exe -k install`. Bind to 127.0.0.1 by default. Windows documents no root-only ports. Add a native Service Control Manager mode later instead of depending on a wrapper; WinSW has had no stable release since January 2023.

### Cited Findings
- "nginx/Windows runs as a standard console application (not a service)", managed with commands such as nginx -s stop, nginx -s quit and nginx -s reload. "Running as a service" is listed under possible future enhancements, and nginx for Windows "is considered to be a beta version" ([nginx for Windows](https://nginx.org/en/docs/windows.html)).
- Apache httpd: "You can install httpd as a Windows NT service as follows ...: httpd.exe -k install". The default service name is Apache2.4; -n names it, -f picks a config, and `httpd.exe -n "MyServiceName" -t` tests the config ([Apache httpd 2.4: Using Apache HTTP Server on Microsoft Windows](https://httpd.apache.org/docs/2.4/platform/windows.html)).
- WinSW: latest stable v2.12.0 (2023-01-28); newest pre-release v3.0.0-alpha.11 (2023-01-29); not archived; last push 2026-07-30; 14,343 stars (GitHub API, 2026-10-07) ([winsw/winsw](https://github.com/winsw/winsw)).
- Servy: MIT license, 2,042 stars, last push 2026-10-07 (GitHub API) ([aelassas/servy](https://github.com/aelassas/servy)).
- Firewall: apps "issue a listen call"; with no allow rule, "a dialog box prompts the user to either allow or block". An admin who answers No or cancels gets block rules for TCP and UDP. For a non-admin, "block rules are created. It doesn't matter what option is selected." Rules must be deleted to see the prompt again. Microsoft recommends adding allow rules "before the application's first launch". The page does not mention loopback ([Microsoft Learn: Windows Firewall Rules](https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/rules)).
- Winsock bind(): WSAEACCES is documented for a datagram bind to the broadcast address without SO_BROADCAST. The reference lists no privilege requirement for ports below 1024 ([Microsoft Learn: bind function](https://learn.microsoft.com/en-us/windows/win32/api/winsock/nf-winsock-bind)).
- Windows can exclude TCP ports from binding (`netsh int ipv4 add excludedportrange`); KB 3039044 documents a WSAEACCES (10013) failure on such a port ([Microsoft Learn: Error 10013 when you bind excluded port again](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/error-10013-wsaeacces-is-returned)).
- SetConsoleCtrlHandler: "CTRL + C and CTRL + BREAK key combinations are typically treated as signals (CTRL_C_EVENT and CTRL_BREAK_EVENT)". "The system generates CTRL_CLOSE_EVENT, CTRL_LOGOFF_EVENT, and CTRL_SHUTDOWN_EVENT signals when the user closes the console, logs off, or shuts down the system" ([Microsoft Learn: SetConsoleCtrlHandler](https://learn.microsoft.com/en-us/windows/console/setconsolectrlhandler)).

### Inferences
- High: the first target (a developer's local server) needs no service. A console unitd that maps CTRL_C_EVENT and CTRL_BREAK_EVENT to a graceful stop and CTRL_CLOSE_EVENT to a fast stop is enough.
- Medium: default the listener and the control endpoint to 127.0.0.1. A loopback-only listener probably avoids the firewall prompt, but no Microsoft page fetched here says so; test it (see Gaps). A 0.0.0.0 listener started by a non-admin user gets silent block rules, which looks like a broken server.
- High (pending test): any user can bind ports below 1024 on Windows. Port 80 may already be taken by HTTP.sys services; that needs a test.
- Medium: Hyper-V, WSL 2 and Docker Desktop, common on Drupal developer machines, reserve port ranges. Error messages for bind failures should suggest `netsh int ipv4 show excludedportrange protocol=tcp`.
- Medium: for production, implement a native service mode (CreateService, StartServiceCtrlDispatcher) in the style of httpd -k install. Until then, document Servy or WinSW for people who need a service.

### Gaps
- The loopback firewall prompt. Test on a clean Windows 11 VM: listeners on 127.0.0.1, [::1] and 0.0.0.0, as admin and as standard user, and record the prompt and any rules created.
- The time Windows allows a CTRL_CLOSE_EVENT handler before it ends the process. Read the HandlerRoutine docs.
- HTTP.sys port 80 conflicts (`netsh http show servicestate`).
- NSSM's maintenance state (not checked).
- Which Windows features create excluded port ranges. No Microsoft source fetched.

## 4.5 Runtime dependencies (Visual C++ runtime)

### Takeaway
Build with /MD and require the Visual C++ v14 Redistributable, as PHP for Windows and Apache Lounge do. Their winget manifests already declare the dependency. Microsoft recommends central deployment and advises against app-local DLLs. The latest v14 redistributable supports only Windows 10 and 11 and Server 2016 to 2025, which sets the floor. There are separate x64, x86 and ARM64 redistributables.

### Cited Findings
- windows.php.net: "The builds below are built using Visual Studio 2019 (VS16) or Visual Studio 2022 (VS17) compiler. They require the Visual C++ Redistributable for Visual Studio 2015-2022 x64 or x86 installed." It lists PHP 8.5.11 VS17 x64 builds dated 2026-09-22 ([PHP downloads for Windows](https://www.php.net/downloads.php?os=windows); windows.php.net now redirects there).
- "Latest supported v14 (for Visual Studio 2017-2026)". "Any apps built by MSVC Build Tools v14.* available in Visual Studio 2017, 2019, 2022, or 2026 can use the latest Visual C++ v14 Redistributable." It "supports only the following operating systems: Windows 10 and 11, Windows Server 2016, 2019, 2022, and 2025". Permalinks: https://aka.ms/vc14/vc_redist.x64.exe, https://aka.ms/vc14/vc_redist.x86.exe and https://aka.ms/vc14/vc_redist.arm64.exe. "You can't install an ARM64 Redistributable on an x86 system or an x64 Redistributable on an x86 system" ([Microsoft Learn: Latest supported Visual C++ Redistributable downloads](https://learn.microsoft.com/en-us/cpp/windows/latest-supported-vc-redist?view=msvc-170)).
- Redistribution "is limited to licensed Visual Studio users and is subject to Microsoft Software License Terms" and the REDIST list. Microsoft recommends the redistributable packages "because they enable automatic updating". Merge modules "are deprecated". App-local: "It's also possible to directly install the Redistributable DLLs in the application local folder ... For servicing reasons, we don't recommend that you use this installation location." ([Microsoft Learn: Redistribute Visual C++ Files](https://learn.microsoft.com/en-us/cpp/windows/redistributing-visual-cpp-files?view=msvc-170)).
- winget flags unresolved C++ runtime dependencies (Validation-VCRuntime-Dependency) ([Microsoft Learn: Submit your manifest to the repository](https://learn.microsoft.com/en-us/windows/package-manager/package/repository)). ApacheLounge.httpd and PHP.PHP.8.5 declare Microsoft.VCRedist.2015+.x64 and .x86 ([winget-pkgs ApacheLounge.httpd](https://github.com/microsoft/winget-pkgs/blob/master/manifests/a/ApacheLounge/httpd/2.4.68/ApacheLounge.httpd.installer.yaml); [winget-pkgs PHP.PHP.8.5](https://github.com/microsoft/winget-pkgs/blob/master/manifests/p/PHP/PHP/8/5/8.5.8/PHP.PHP.8.5.installer.yaml)). Scoop's php.json suggests extras/vcredist2022 ([Scoop Main php.json](https://github.com/ScoopInstaller/Main/blob/master/bucket/php.json)).
- Apache Lounge now builds with VS18 (file names end in "Win64-VS18") ([winget-pkgs ApacheLounge.httpd](https://github.com/microsoft/winget-pkgs/blob/master/manifests/a/ApacheLounge/httpd/2.4.68/ApacheLounge.httpd.installer.yaml)).

### Inferences
- Medium-high: use /MD. The PHP module loads PHP's own DLLs, which use the shared CRT; one CRT across the process avoids heap and FILE handle mismatches at DLL boundaries. A Drupal developer already needs the redistributable for PHP, so it adds no new prerequisite.
- High: declare Microsoft.VCRedist.2015+.<arch> in the winget manifest and suggest extras/vcredist2022 in the Scoop manifest. For the plain zip, state the prerequisite and the aka.ms link in a README.
- Medium: do not ship vcruntime140.dll app-local. Microsoft advises against it, and the license limits it to the REDIST list.
- Medium: an x64 FreeUnit running under Prism on an ARM64 PC needs the x64 redistributable; a native ARM64 build needs the ARM64 one.

### Gaps
- Settled in 1.4: FreeUnit's PHP module passes a `FILE *` from its own `fopen()` into PHP (src/nxt_php_sapi.c:1290@872bf041), so module and PHP must share one CRT.
- Whether the ARM64 redistributable also installs the x64 runtime for emulated apps. Test on an ARM64 VM.
- Settled in 1.1 and 5.6: windows.php.net lists x64 and x86 builds only, and php-windows-builder builds x64 and x86 only (3.3).

## 4.6 ARM64

### Takeaway
winget and Scoop already handle per-architecture zips; Caddy ships x64 and arm64 through both. Windows 11 24H2 runs x64 apps through the Prism emulator, which Microsoft calls faster but gives no numbers for on its Learn page. A native ARM64 build is useful only once PHP for Windows ships ARM64; until then the x64 build under Prism is the fallback.

### Cited Findings
- "Performance is enhanced with the introduction of the new emulator Prism in Windows 11 24H2. Windows 10 on Arm also supports emulation, but only for x86 apps." Prism "includes significant optimizations that improve the performance and lower CPU usage of apps under emulation" and "is optimized and tuned specifically for Qualcomm Snapdragon processors" ([Microsoft Learn: How emulation works on Arm](https://learn.microsoft.com/en-us/windows/arm/apps-on-arm-x86-emulation)).
- winget Architecture values include arm64 ([winget-pkgs installer schema 1.12.0](https://github.com/microsoft/winget-pkgs/blob/master/doc/manifest/schema/1.12.0/installer.md)); CaddyServer.Caddy lists separate x64 and arm64 zips ([winget-pkgs CaddyServer.Caddy 2.11.7](https://github.com/microsoft/winget-pkgs/blob/master/manifests/c/CaddyServer/Caddy/2.11.7/CaddyServer.Caddy.installer.yaml)).
- Scoop's caddy.json has 64bit and arm64 architecture entries ([Scoop Main caddy.json](https://github.com/ScoopInstaller/Main/blob/master/bucket/caddy.json)).
- An ARM64 VC++ redistributable exists (https://aka.ms/vc14/vc_redist.arm64.exe) ([Microsoft Learn: Latest supported Visual C++ Redistributable downloads](https://learn.microsoft.com/en-us/cpp/windows/latest-supported-vc-redist?view=msvc-170)).

### Inferences
- Medium: ship x64 first and state "runs under emulation on Windows 11 24H2 or later on Arm". Add a native ARM64 zip when PHP has ARM64 builds and CI can test them.
- Medium: Windows 10 on Arm cannot run an x64 FreeUnit at all (x86 emulation only). Do not promise it.

### Gaps
- Prism overhead for a multi-process server (unitd, router, PHP workers). Measure requests per second on an ARM64 device, native against emulated.
- How Chocolatey packages pick per-architecture URLs for ARM64. Read the Chocolatey helper docs.

## 4.7 Windows version support facts

### Takeaway
By a 2027 first release, Windows 10 is past end of support and gets fixes only through ESU: consumers until 2027-10-12 after a June 2026 extension, businesses for up to three years from October 2025. Windows 11 24H2 Home and Pro leave servicing on 2026-10-13. Windows Server 2019 is in extended support until 2029-01-09. Recommend: support Windows 11 versions in servicing, Windows Server 2022 and 2025; treat Windows 10 22H2 and Server 2019 as "expected to work, not tested"; never go below Windows 10 and Server 2016, because the VC++ runtime itself stops there.

### Cited Findings
- Method: Microsoft lifecycle pages store each end date as a UTC timestamp at 06:59:59, which is the end of the previous day in US Pacific time. The dates below are those Pacific dates, which fall on the usual second Tuesday. For example the Windows Server 2025 page stores 11/14/2029 and 11/15/2034, shown here as 2029-11-13 and 2034-11-14 ([Microsoft Learn lifecycle: Windows Server 2025](https://learn.microsoft.com/en-us/lifecycle/products/windows-server-2025)).
- Windows 10 support ended on October 14, 2025. Commercial ESU costs "$61 USD per device for Year One"; "The price doubles every consecutive year, for a maximum of three years." ESU is free for Windows 10 VMs in Windows 365, Azure Virtual Desktop and Azure VMs ([Microsoft Learn: Windows 10 Extended Security Updates](https://learn.microsoft.com/en-us/windows/whats-new/extended-security-updates)).
- Consumer ESU: enroll by syncing settings with Windows Backup "at no additional cost", by redeeming "1,000 Microsoft Rewards points", or by paying "$30 USD (local pricing may vary)". An editor's note dated 2026-06-25 extends coverage for personal devices to Oct. 12, 2027. Terms differ in the EEA ([Windows Experience Blog, 2025-06-24 post with editor's notes](https://blogs.windows.com/windowsexperience/2025/06/24/stay-secure-with-windows-11-copilot-pcs-and-windows-365-before-support-ends-for-windows-10/)). The original consumer end date was 2026-10-13 ([Help Net Security, 2026-06-26](https://www.helpnetsecurity.com/2026/06/26/microsoft-windows-10-free-security-updates-esu-program/)).
- Windows 11 Home and Pro end of servicing: 23H2 ended 2025-11-11; 24H2 ends 2026-10-13; 25H2 (released 2025-09-30) ends 2027-10-12; 26H1 (released 2026-02-10) ends 2028-03-14; 26H2 (released 2026-09-29) ends about 2028-10-10 (the page value is 10/10/2028 06:59:59, which breaks the pattern of the other rows) ([Microsoft Learn lifecycle: Windows 11 Home and Pro](https://learn.microsoft.com/en-us/lifecycle/products/windows-11-home-and-pro)).
- Windows 11 Enterprise and Education end of servicing: 23H2 ends 2026-11-10; 24H2 ends 2027-10-12; 25H2 ends 2028-10-10; 26H1 ends 2029-03-13; 26H2 ends 2029-10-09 ([Microsoft Learn lifecycle: Windows 11 Enterprise and Education](https://learn.microsoft.com/en-us/lifecycle/products/windows-11-enterprise-and-education)).
- Windows Server 2019: mainstream ended 2024-01-09; extended ends 2029-01-09 ([Microsoft Learn lifecycle: Windows Server 2019](https://learn.microsoft.com/en-us/lifecycle/products/windows-server-2019)).
- Windows Server 2022: mainstream ends 2026-10-13; extended ends 2031-10-14 ([Microsoft Learn lifecycle: Windows Server 2022](https://learn.microsoft.com/en-us/lifecycle/products/windows-server-2022)).
- Windows Server 2025: released 2024-11-01; mainstream ends 2029-11-13; extended ends 2034-11-14 ([Microsoft Learn lifecycle: Windows Server 2025](https://learn.microsoft.com/en-us/lifecycle/products/windows-server-2025)).
- Windows 10 Enterprise LTSC 2021 ends 2027-01-12 ([Microsoft Learn lifecycle: Windows 10 Enterprise LTSC 2021](https://learn.microsoft.com/en-us/lifecycle/products/windows-10-enterprise-ltsc-2021)). Windows 11 Enterprise LTSC 2024 (released 2024-10-01) has mainstream support until 2029-10-09 ([Microsoft Learn lifecycle: Windows 11 Enterprise LTSC 2024](https://learn.microsoft.com/en-us/lifecycle/products/windows-11-enterprise-ltsc-2024)).
- The latest VC++ v14 redistributable supports only Windows 10 and 11 and Windows Server 2016, 2019, 2022 and 2025 ([Microsoft Learn: Latest supported Visual C++ Redistributable downloads](https://learn.microsoft.com/en-us/cpp/windows/latest-supported-vc-redist?view=msvc-170)).

### Inferences
- Medium: build for a Windows 10 API floor (_WIN32_WINNT 0x0A00) so Windows 10 22H2 ESU machines and Server 2019 run it, but test and promise only Windows 11 (versions in servicing) and Server 2022 and 2025 on x64.
- Medium: for the developer-server target, Windows 11 covers nearly all users in 2027. Windows 10 ESU ends 2027-10-12 for consumers, likely within months of a first release, so it does not justify test effort.
- High: do not claim support for anything older than Windows 10 or Server 2016; the runtime FreeUnit depends on does not support it.

### Gaps
- The Windows 10 Home and Pro and Windows Server 2016 lifecycle pages were not fetched. Fetch them for the support statement.
- The exact end date of commercial Windows 10 ESU year three (expected October 2028).
- Whether Windows 11 26H1 ships only on new Arm devices. Read the Windows 11 release information page.

## 4.8 Packaging and distribution plan with costs

### Takeaway
Ship in three steps. First preview: an x64 portable zip on GitHub Releases with SHA-256 sums, a winget manifest and an own Scoop bucket, signed through SignPath Foundation if approved in time. Second: sign every release (SignPath for free, or Artifact Signing at $119.88 a year if a legal entity in an eligible country exists), add ARM64 when PHP supports it. Third, with production and service mode: an MSI built with WiX v7 that installs the service and a firewall rule, plus a Chocolatey package. The recurring cost is $0 on the free path.

### Cited Findings
- Artifact Signing Basic is $9.99 per month, which is $119.88 per year ([Microsoft Learn: Change the account SKU](https://learn.microsoft.com/en-us/azure/artifact-signing/how-to-change-sku)).
- Certum open-source certificates start at 25.00 EUR (code), 49.00 EUR (cloud) and 69.00 EUR (set), with at most 459 days of validity ([Certum shop: Code Signing](https://shop.certum.eu/code-signing.html)).
- SignPath Foundation signing is free for qualifying projects, with SignPath Foundation as publisher ([SignPath Foundation terms](https://signpath.org/terms)).
- WiX requires payment only from organizations with more than $10,000 annual revenue ([FireGiant docs: Open Source Maintenance Fee](https://docs.firegiant.com/wix/osmf/)).
- winget needs no signature, and Scoop Main needs 500 stars and 150 forks ([Microsoft Learn: Windows Package Manager repository policies](https://learn.microsoft.com/en-us/windows/package-manager/package/windows-package-manager-policies); [Scoop wiki: Criteria for the main bucket](https://github.com/ScoopInstaller/Scoop/wiki/Criteria-for-including-apps-in-the-main-bucket)).

### Inferences
- Plan, with confidence:

| Step | What ships | Signing | Channels | Recurring cost per year | Confidence |
|---|---|---|---|---|---|
| 1. Preview (first 2027 release) | freeunit-<ver>-windows-x64.zip: unitd.exe, unitctl.exe, PHP module, OpenSSL DLLs, README with the VC++ prerequisite; SHA256SUMS | SignPath Foundation if approved, else unsigned with a documented SmartScreen prompt | GitHub Releases; winget (zip, nested portable, VCRedist dependency); own Scoop bucket | $0 | high |
| 2. Stable developer release | Same zip plus an ARM64 zip once PHP ARM64 builds exist | Every exe, dll and zip content signed and timestamped; one identity kept across releases | Add Scoop Extras or Main when eligible | $0 (SignPath) or $119.88 (Artifact Signing Basic, needs an eligible legal entity) or 25 to 69 EUR (Certum OSS, yearly) | medium |
| 3. Production (later) | MSI from WiX v7: native service, firewall allow rule, upgrade codes | Same identity signs the MSI | winget (wix installer type), Chocolatey | $0 for WiX while the project has no revenue | medium |

- High: Defender and SmartScreen plan. (a) Sign with one stable identity from the first public build, because reputation accrues to the publisher only when the identity stays the same. (b) Before each announcement, submit new binaries to https://www.microsoft.com/en-us/wdsi/filesubmission and check VirusTotal. (c) Document the expected first-week SmartScreen prompt and how to verify the SHA-256 sum. (d) Watch winget validation labels as a free Defender and antivirus signal on every release.
- Medium: do not plan an MSIX or Microsoft Store build. The Store is the only route without SmartScreen prompts, but a packaged service needs admin rights and a restricted capability, which conflicts with a no-admin developer tool.
- Medium: the one decision that changes cost is legal identity. Without a legal entity, the realistic choices are SignPath Foundation (free, foreign publisher name) or Certum open source (cheap, a person's name). With an entity in an eligible country, Artifact Signing at about $120 a year is the simplest CI integration.

### Gaps
- SignPath approval time and scope (third-party DLLs, MSI). Apply early; it gates step 1 signing.
- Whether FreeUnit has, or can get through a fiscal host, a legal entity in an Artifact Signing country.
- Smart App Control and SmartScreen behaviour of the actual artifacts. Run the step 1 zip on clean Windows 11 VMs with Smart App Control on and off, installed by browser download, winget and Scoop.

## 5.1 Developer OS share (Stack Overflow 2024 and 2025)

### Takeaway
Windows is the largest OS group among all developers. Stack Overflow 2025: 49.5% for professional use and 56.7% for personal use (2024: 47.6% and 59.2%). WSL is reported by 16.8% for professional use in both years. No PHP-only cross-tab is published, so it was computed from the official public dataset: respondents who have worked with PHP use Windows at work as often as everyone else (54.0% against 54.0% on the same basis in 2025), and about 35% of them use Windows without WSL.

### Cited Findings
- [ALL DEVS] 2024 question wording: "What is the primary operating system in which you work?" Results are split into "Personal use" and "Pro. use". Responses: 58,600 (89.6% of respondents). Source: [Stack Overflow Developer Survey 2024, Technology](https://survey.stackoverflow.co/2024/technology#most-popular-technologies-op-sys)
- [ALL DEVS] 2024 professional use: Windows 47.6%, MacOS 31.8%, Ubuntu 27.7%, WSL 16.8%, Debian 9.1%, Android 8.4%, Other Linux-based 8%, iOS 7.3%, Red Hat 4.9%, Arch 4.3%, Fedora 3.3%. Personal use: Windows 59.2%, MacOS 31.8%, Ubuntu 27.7%, Android 17.9%, WSL 17.1%, iOS 11.5%, Debian 9.8%, Other Linux-based 8.4%, Arch 8%. Source: [Stack Overflow Developer Survey 2024, Technology](https://survey.stackoverflow.co/2024/technology#most-popular-technologies-op-sys)
- [ALL DEVS] 2025 uses the same wording. Responses: 31,569 (64.4% of respondents). A new option "Linux (non-WSL)" appears. Source: [Stack Overflow Developer Survey 2025, Technology](https://survey.stackoverflow.co/2025/technology#most-popular-technologies-op-sys)
- [ALL DEVS] 2025 professional use: Windows 49.5%, MacOS 32.9%, Ubuntu 27.7%, WSL 16.8%, Linux (non-WSL) 16.7%, Android 11.9%, iOS 10.5%, Debian 10.4%, Red Hat 5.7%, Arch 4.6%, Fedora 3.7%. Personal use: Windows 56.7%, MacOS 32.7%, Android 29.1%, Ubuntu 27.8%, iOS 18.9%, Linux (non-WSL) 17.6%, WSL 15.9%, Debian 11.4%, Arch 9.7%, Fedora 5.8%. Source: [Stack Overflow Developer Survey 2025, Technology](https://survey.stackoverflow.co/2025/technology#most-popular-technologies-op-sys)
- [ALL DEVS] The shares in both years add up to far more than 100%. Despite the word "primary", respondents could pick several systems. Source: [Stack Overflow Developer Survey 2025, Technology](https://survey.stackoverflow.co/2025/technology#most-popular-technologies-op-sys)
- [ALL DEVS] PHP use: 18.2% of respondents in 2024 (language question, 60,171 responses) and 18.9% in 2025 (31,771 responses). Sources: [2024 Technology](https://survey.stackoverflow.co/2024/technology#most-popular-technologies-language), [2025 Technology](https://survey.stackoverflow.co/2025/technology#most-popular-technologies-language)
- [PHP DEVS, computed] From the official public datasets, counting only respondents who answered both the OS question and the language question: in 2025, 5,298 professional respondents have worked with PHP. Of them 54.0% use Windows professionally (all respondents on the same basis, n=28,289: 54.0%), Ubuntu 39.3%, macOS 37.2%, Linux (non-WSL) 22.0% and WSL 21.0% (all: WSL 18.5%). For personal use, 59.1% of 5,800 PHP respondents use Windows (all: 56.8%). In 2024: Windows 54.5% of 9,693 PHP respondents for professional use (all: 52.7% of 52,646), WSL 21.4% (all: 18.6%). The same-basis all-respondent share (54.0%) is higher than Stack Overflow's published 49.5% because the published figure keeps respondents who skipped the language question. Method: rows whose `LanguageHaveWorkedWith` contains `PHP`, multi-select `OpSysProfessional use`; this is my tabulation, not a published figure. Sources: [2025 results.csv](https://github.com/StackExchange/Survey/raw/refs/heads/main/packages/archive/2025/results.csv), [2024 results.csv](https://github.com/StackExchange/Survey/raw/refs/heads/main/packages/archive/2024/results.csv) (ODbL, linked from [survey.stackoverflow.co](https://survey.stackoverflow.co/))
- [PHP DEVS, computed] Professional PHP respondents who chose Windows but not WSL: 35.3% in 2025 and 36.0% in 2024 (all respondents: 37.5% and 36.8%). Windows as the only OS selected: 12.7% and 14.9% (all: 19.0% and 19.7%). Windows together with WSL: 18.7% and 18.4%. Same method. Sources: [2025 results.csv](https://github.com/StackExchange/Survey/raw/refs/heads/main/packages/archive/2025/results.csv), [2024 results.csv](https://github.com/StackExchange/Survey/raw/refs/heads/main/packages/archive/2024/results.csv)

### Inferences
- If every WSL user also selected Windows, about one in three professional Windows respondents also uses WSL (16.8% of 49.5% in 2025). The overlap is not published. Confidence: low.
- The 2025 response count for this question is about half of 2024 (31,569 vs 58,600), and the option list changed. Year-over-year moves of 1 to 2 points are within that noise. Confidence: medium.
- A native Windows server targets the largest single OS group among all developers. Confidence: high for all developers, unknown for PHP developers.

### Gaps
- The cross-tab above counts anyone who has worked with PHP, not people whose main language is PHP; the 2025 `LanguageChoice` column was not used. A main-language cut would sharpen it.
- "Windows without WSL" counts people who did not tick WSL; it does not prove they cannot use WSL.

## 5.2 PHP developers specifically (JetBrains, Perforce/Zend, Laravel)

### Takeaway
No current public source gives the workstation OS of PHP developers. JetBrains State of PHP 2025 and the Developer Ecosystem 2024 landing page publish no OS or local-environment data. The Perforce (Zend) 2025 PHP Landscape Report asks only where PHP is deployed: Windows 13.15%, IIS 4.50%, from 561 responses. The often-quoted "42% of PHP developers develop on Windows" comes from a Zend survey reported in February 2010.

### Cited Findings
- [PHP DEVS] State of PHP 2025 (October 2025): 1,720 respondents who named PHP as their main language; largest groups in Japan, the United States, Russia, China and France. PHP 8.x 89%, PHP 7.x 33%. Laravel 64%, WordPress 25%, Symfony 23%. PhpStorm 68%, VS Code 23%. The post has no OS, local-environment or web-server data. Source: [The State of PHP 2025, JetBrains PhpStorm blog](https://blog.jetbrains.com/phpstorm/2025/10/state-of-php-2025/)
- [ALL DEVS] Developer Ecosystem 2024 landing page: 23,262 respondents; no OS breakdown and no PHP local-environment section on that page. Source: [State of Developer Ecosystem 2024](https://www.jetbrains.com/lp/devecosystem-2024/)
- [PHP DEVS] Perforce 2025 PHP Landscape Report: anonymous survey of self-identified PHP users, October to December 2024, 561 valid responses. Source: [2025 PHP Landscape Report (PDF)](https://www.zend.com/system/files/2025-06/report-zend-2025-php-landscape-report_0.pdf)
- [PHP DEVS] Same report, deployment OS (multi-select): Ubuntu 55.60%, Debian 38.15%, CentOS 18.75%, Alpine 17.67%, RHEL 13.58%, Windows 13.15%, Amazon Linux 11.85%. The Windows value is the sixth bar of the chart, read from the PDF text layer. Web servers: Apache 70.02%, nginx 66.60%, IIS 4.50%. Source: [2025 PHP Landscape Report (PDF)](https://www.zend.com/system/files/2025-06/report-zend-2025-php-landscape-report_0.pdf)
- [PHP DEVS, 2010] Zend survey of 2,000 PHP developers, reported 2010-02-18: Windows 42% as primary development OS, Linux 38.5%, Mac OS X 19.1%. Production: Linux 85%, Windows 11%, Mac OS X 2%. Source: [Windows is the Choice of Enterprise Developers, InternetNews](https://internetnews.com/software/windows-is-the-choice-of-enterprise-developers)
- Conflict: a web search summary attributed the 42% / 38.5% / 19.1% split to the 2025 Zend report. It is from the 2010 article above. The 2025 report has no development-OS question. Source: [2025 PHP Landscape Report (PDF)](https://www.zend.com/system/files/2025-06/report-zend-2025-php-landscape-report_0.pdf)

### Inferences
- Windows as a PHP production target is small but real: 13.15% of Perforce respondents deploy at least one PHP application on Windows, and 4.50% use IIS. The sample is small and self-selected. Confidence: medium.
- The 2010 split (Windows 42% for development, 11% for production) shows the old pattern: write on Windows, deploy on Linux. Whether it holds in 2026 is unknown. Confidence: low.

### Gaps
- The JetBrains Developer Ecosystem Data Playground (https://www.jetbrains.com/lp/devecosystem-data-playground/) may allow an OS by main-language cross-tab. Not checked.
- The State of PHP 2024 post was not read.
- No Laravel survey and no Herd install or user count was found.
- No 2024 to 2026 source read here gives local-environment shares (Docker, Herd, Valet, Sail, XAMPP, WSL, DDEV, Lando) for PHP developers.

## 5.3 Drupal (DDEV telemetry, drupal.org)

### Takeaway
drupal.org recommends DDEV. DDEV's live opt-in telemetry shows 25.2% of platform rows on Windows hosts, and 85% of those already run inside WSL2. Drupal 10 and 11 together are 32.0% of DDEV project types.

### Cited Findings
- [DRUPAL] "The Drupal community recommends using DDEV, a free, cross-platform local development solution." The page names no decision owner and no date. Source: [Local Development Guide, drupal.org](https://www.drupal.org/docs/official_docs/local-development-guide)
- [DDEV USERS] Data as of 2026-10-06, from opt-in usage reporting pulled from Amplitude: 30,682 unique users in September 2026. Most sections cover about the last 7 days. Source: [DDEV Live Usage Statistics](https://ddev.com/usage-stats/)
- [DDEV USERS] Platform: macOS 53.2% (13,072), Linux 21.5% (5,292), WSL2 19.2% (4,717), Windows 3.7% (915), WSL2-mirrored 2.2% (542), Codespaces 0.1% (17), WSL2-virtioproxy 0.0% (7). Source: [DDEV Live Usage Statistics](https://ddev.com/usage-stats/)
- [DDEV USERS] Docker provider on WSL2: docker-ce inside WSL2 64.4% (3,102), Docker Desktop 35.1% (1,690). On macOS: Docker Desktop 55.6% (7,297), OrbStack 32.7% (4,293), Colima 8.3% (1,095), Rancher Desktop 2.1% (276). macOS arm64 94.7%. Source: [DDEV Live Usage Statistics](https://ddev.com/usage-stats/)
- [DDEV USERS] Project types: drupal11 18.6% (5,764), php 16.1% (4,989), drupal10 13.4% (4,134), wordpress 12.0% (3,699), laravel 10.5% (3,237), typo3 7.5% (2,325), craftcms 4.5% (1,387). Source: [DDEV Live Usage Statistics](https://ddev.com/usage-stats/)
- The November 2024 statistics post URL now serves the same live page (title "DDEV Live Usage Statistics"), so the OS split lives on the live page, not in a dated post. Source: [ddev.com/blog/stats-on-ddev-usage-nov-2024](https://ddev.com/blog/stats-on-ddev-usage-nov-2024/)

### Inferences
- Windows-host share of the platform table: (4,717 + 915 + 542 + 7) / 24,562 = 25.2%. Within Windows hosts, WSL2 variants are 85.2% and traditional Windows 14.8%. The table sums to 24,562, not to the 30,682 monthly users, and its window is not stated per table. Confidence: medium.
- Most Drupal developers on Windows who use the recommended tool have already solved local development with WSL2 plus docker-ce. A native FreeUnit will not replace DDEV for them. Its audience is Windows Drupal developers outside DDEV: those blocked from WSL2 or Docker, or who want no VM. Confidence: medium.
- Telemetry only counts people who already run Docker. Developers whose machines block WSL2 or virtualization are invisible in it, so it understates the blocked segment. Confidence: high.
- About two in three WSL2 users run docker-ce rather than Docker Desktop. This fits avoiding the Docker Desktop licence and overhead, but the cause is not measured. Confidence: low.

### Gaps
- Date and owner of the DDEV recommendation. Secondary reports say June 2024; The Drop Times page (https://www.thedroptimes.com/node/35228) returned HTTP 403. Read the drupal.org issue that adopted DDEV.
- No Drupal Business Survey or DrupalCon survey with OS data was found (not searched in depth).
- The Docker-provider split for traditional Windows DDEV users was not extracted.

## 5.4 Windows-native PHP tools and their reception

### Takeaway
Windows-native PHP stacks have very large download volume. XAMPP on SourceForge: 19.26 million downloads in 2025, 85.2% from Windows. WampServer: 1.34 million, 91.3% from Windows. Laragon: 1.81 million GitHub asset downloads of its paid v7 and v8 lines since 2024-12-16. Herd for Windows shipped on 2024-03-26; no Herd numbers are published.

### Cited Findings
- Herd for Windows release, 2024-03-26, by Eric L. Barnes: "Herd uses native binaries for PHP, nginx, and other services"; no containers or VMs; Herd Pro adds databases, caches, queues, storage. Source: [Laravel Herd for Windows is now released, Laravel News](https://laravel-news.com/laravel-herd-for-windows-is-now-released)
- Laragon pricing: unlicensed use is permitted for non-commercial purposes, with a licence reminder popup. Commercial: Annual USD 49 (1 device) or 69 (2 devices); Lifetime USD 149 or 199; Team Annual USD 149 (5 devices) or 399 (15 devices). Non-commercial paid: Standard USD 10 or 20, Extended USD 30 or 40. Source: [Laragon pricing](https://laragon.org/pricing)
- Laragon maintainer, January 2025: "Laragon v6 was always free and continues to be free. Laragon v7/v8 has never been free." Source: [leokhoa/laragon discussion 985](https://github.com/leokhoa/laragon/discussions/985)
- [DOWNLOADS] Laragon GitHub release assets, read 2026-10-07: 7.0.6 published 2024-12-16, 141,036 downloads; twelve 8.x releases from 8.0.0 (2025-03-13) to 8.7.0 (2026-08-14), 1,666,438 downloads; 6.0.0 (2022-09-16), 2,015,575 downloads. Source: [leokhoa/laragon releases](https://github.com/leokhoa/laragon/releases) (counts from https://api.github.com/repos/leokhoa/laragon/releases)
- [DOWNLOADS] XAMPP on SourceForge, calendar 2025: 19,258,068 downloads. Windows 16,398,979; Unknown 1,247,636; Macintosh 872,795; Linux 620,799; Android 117,858. Top countries: India 3,246,160; Indonesia 1,892,785; Pakistan 1,061,561; Brazil 928,612; Philippines 775,528. Source: [SourceForge XAMPP stats 2025](https://sourceforge.net/projects/xampp/files/stats/json?start_date=2025-01-01&end_date=2025-12-31)
- [DOWNLOADS] WampServer on SourceForge, calendar 2025: 1,338,894 downloads. Windows 1,221,845. Top countries: India 202,060; France 108,774; Brazil 73,012; United States 52,162; Pakistan 50,177. Source: [SourceForge WampServer stats 2025](https://sourceforge.net/projects/wampserver/files/stats/json?start_date=2025-01-01&end_date=2025-12-31)
- Conflict with the briefing: Laragon did not simply become paid in 2024. The paid line started with v7.0.6 on 2024-12-16, v6 stays free, and non-commercial use of v7 and v8 is allowed without a licence. Commercial use needs a licence. Source: [Laragon pricing](https://laragon.org/pricing)

### Inferences
- Download counts are not users. SourceForge counts every file fetch, including repeats and bots. GitHub counts asset downloads, including the updater. Confidence: high.
- XAMPP's top countries (India, Indonesia, Pakistan, Brazil, Philippines) point to students and beginners. That is a large audience for adoption and a small one for paid support. Confidence: medium.
- Laragon 8.x drew about 1.67 million downloads in 19 months after commercial use became paid, while WSL2 is free. This is direct evidence that a no-VM Windows PHP stack has demand. Confidence: medium.

### Gaps
- Herd download or active-user counts: none published that I found.
- XAMPP and WampServer for 2024 and 2026: curl hit a Cloudflare challenge; WebFetch worked for 2025 only. Rerun the JSON endpoint for other years to get a trend.
- Laragon's SourceForge mirror was not pulled.

## 5.5 What blocks the alternatives (Docker Desktop terms, WSL policy, VDI)

### Takeaway
Docker Desktop needs a paid seat for professional use at organizations with 250 or more employees or USD 10 million or more in revenue. Business costs USD 24 per user per month. Docker supports Docker Desktop on virtual desktops only for Business customers, only with nested virtualization, and never on non-persistent VDI. Intune can switch WSL off per machine. I found no survey that measures how often enterprises block WSL or Docker.

### Cited Findings
- Docker Desktop is free for "Small businesses (fewer than 250 employees AND less than $10 million in annual revenue)", personal use, education and non-commercial open source. A paid subscription is required for "Professional use in larger organizations" and for government use. Source: [Docker Desktop license agreement, Docker Docs](https://docs.docker.com/subscription/desktop-license/)
- Pricing page, read 2026-10-07: Personal USD 0; Pro USD 9 per user/month billed annually (USD 11 billed monthly); Team USD 15 (USD 16); Business USD 24 (USD 24). Source: [Docker pricing](https://www.docker.com/pricing/)
- Price change announced 2024-09-12, effective 2024-12-10: Pro from USD 5 to 9 per month; Team from USD 9 to 15 per user per month (annual discounts); Business unchanged at USD 24. Docker Hub pull and storage limits from 2025-03-01. Source: [Announcing Upgraded Docker Plans, Docker blog](https://www.docker.com/blog/november-2024-updated-plans-announcement/)
- "Support for running Docker Desktop on a virtual desktop is available to Docker Business customers only." It needs nested virtualization. VMware ESXi and Azure VM: "Supported. Tested by Docker." Nutanix: supported if WSL 2 or Windows container mode works. "Any other hypervisor: Not supported." Non-persistent VDI: "not supported". "Docker Desktop requires nested virtualization, which is not supported by Citrix Hypervisor/XenServer." Without nested virtualization Docker recommends its cloud product, Docker Offload. Source: [Run Docker Desktop for Windows in a VM or VDI environment](https://docs.docker.com/desktop/setup/vm-vdi/)
- Docker Desktop on Windows needs WSL 2.1.5 or later, a 64-bit CPU with SLAT, 8 GB RAM, and hardware virtualization enabled in BIOS/UEFI. Source: [Install Docker Desktop on Windows](https://docs.docker.com/desktop/setup/install/windows-install/)
- Intune setting "Allow the Windows Subsystem For Linux": "When set to disabled, this policy disables access to the Windows Subsystem For Linux for all users on the machine." Other settings disable the inbox WSL, WSL1, the debug shell, custom kernels, and .wslconfig control of nested virtualization, networking mode and firewall. Source: [Intune settings for WSL, Microsoft Learn](https://learn.microsoft.com/en-us/windows/wsl/intune)

### Inferences
- The free tier needs both conditions. A 30-person agency with USD 10 million or more in revenue pays for every developer who uses Docker Desktop at work. Confidence: high (text of the terms).
- Docker Desktop is avoidable: 64% of DDEV WSL2 users run free docker-ce inside WSL2. So the licence pushes users toward WSL2 plus docker-ce, which still needs WSL2 and hardware virtualization. A native FreeUnit needs neither. Confidence: medium.
- Locked-down estates (Citrix Hypervisor VDI, non-persistent desktops, WSL disabled by policy, virtualization off in firmware) cannot run any VM-based option. A native Windows binary is one of the few routes there. The size of this group is unknown. Confidence: high for the mechanism, low for the size.

### Gaps
- No survey or credible report quantifies how often enterprises block WSL2, Docker Desktop or virtualization. One search found only vendor hardening guides. A short poll in Drupal and DDEV community channels would settle it.
- The Microsoft Learn "WSL enterprise" page (https://learn.microsoft.com/en-us/windows/wsl/enterprise) was downloaded but not mined.

## 5.6 ARM Windows laptops

### Takeaway
Windows on Arm is a small share of PCs. Canalys data, as reported by TechRadar, put Snapdragon X at about 720,000 units, under 0.8% of all PCs shipped in Q3 2024. Docker's docs still label Docker Desktop for Windows on Arm "Early Access" in October 2026. windows.php.net publishes no ARM64 PHP builds.

### Cited Findings
- "Only about 720,000 Qualcomm Snapdragon X laptops sold since launch", "under 0.8% of the total number of PCs shipped in Q3". The fetched excerpt did not show the research firm; other outlets credit Canalys. Source: [TechRadar](https://www.techradar.com/pro/Only-about-720000-Qualcomm-Snapdragon--laptops-sold-since-launch)
- Docker lists "Docker Desktop for Windows - Arm (Early Access)" and "WSL 2 backend, Arm (Early Access)". Source: [Install Docker Desktop on Windows](https://docs.docker.com/desktop/setup/install/windows-install/)
- The windows.php.net releases directory lists only x64 and x86 zips (vs16 and vs17, thread-safe and non-thread-safe), with no arm64 file names. The download page shows PHP 8.5 (8.5.11) as current. Sources: [windows.php.net releases](https://windows.php.net/downloads/releases/), [PHP downloads for Windows](https://www.php.net/downloads.php?os=windows)

### Inferences
- ARM laptops are not a demand driver for a first Windows FreeUnit. Ship x64 first. Confidence: medium.
- An ARM64 FreeUnit would also need ARM64 PHP builds, which the official Windows PHP site does not ship today. Confidence: high for the listing on 2026-10-07.

### Gaps
- No primary analyst figure for the 2025 Windows-on-Arm share was read. Secondary only: ABI Research forecast Arm PCs, Macs included, at up to 13% of 2025 shipments (https://www.tomshardware.com/pc-components/cpus/arm-pc-market-share-wont-rise-above-13-percent-in-2025-says-abi-research, not fetched). A search summary credited Mercury Research with Arm at 14.4% of PC CPUs in Q1 2026, Apple included (not verified). Read the IDC or Canalys releases.
- Snapdragon X launch date and Copilot+ PC share were not verified here.
- WSL2 on ARM64 status and the cost of x86-64 container emulation were not checked.

## 5.7 What Windows PHP developers run today as a server

### Takeaway
php-fpm does not exist on Windows. Windows stacks use mod_php in Apache (thread-safe builds), php-cgi behind IIS FastCGI or nginx, or the built-in server. php-cgi on Windows does have a fixed process pool: PHP_FCGI_CHILDREN starts up to 64 children and respawns them. The built-in server is one single-threaded process, and PHP_CLI_SERVER_WORKERS is not supported on Windows.

### Cited Findings
- php-src master: sapi/fpm has config.m4 but no config.w32. sapi/cgi has config.w32. Windows builds read config.w32, so FPM is not built on Windows. Sources: [php-src sapi/fpm](https://github.com/php/php-src/tree/master/sapi/fpm), [php-src sapi/cgi](https://github.com/php/php-src/tree/master/sapi/cgi)
- User notes on the FPM install page: "php-fpm is not avaliable on Windows", and FPM "is built around fork()" (bug 62447). These are user notes, not normative text. Source: [FastCGI Process Manager installation, php.net](https://www.php.net/manual/en/install.fpm.php)
- php-cgi on Windows: WIN32_MAX_SPAWN_CHILDREN is 64. With PHP_FCGI_CHILDREN set, the parent starts min(children, 64) child processes with CreateProcessW, puts them in a Job Object, gives each child the listening socket as stdin, and restarts exited children in a WaitForMultipleObjects loop. PHP_FCGI_MAX_REQUESTS is read on all platforms; default 500. Source: [php-src sapi/cgi/cgi_main.c at 20f4c4d6, lines 227-228 and 2092-2210](https://github.com/php/php-src/blob/20f4c4d605203b95cf01e76e306c25bdff6678b2/sapi/cgi/cgi_main.c#L2092-L2210)
- Built-in server: "The web server runs only one single-threaded process, so PHP applications will stall if a request is blocked." PHP_CLI_SERVER_WORKERS exists since PHP 7.4.0, and "Multiple workers are not supported on Windows." Source: [Built-in web server, php.net](https://www.php.net/manual/en/features.commandline.webserver.php)
- In php-src the worker code is under `#ifdef HAVE_FORK`. Source: [php-src sapi/cli/php_cli_server.c](https://github.com/php/php-src/blob/master/sapi/cli/php_cli_server.c)
- "If you want to use PHP as FastCGI with IIS, use the Non-Thread Safe (NTS) builds of PHP, or if you want to use the Apache HTTP Server, use the Thread Safe (TS) builds of PHP." Source: [PHP downloads for Windows](https://www.php.net/downloads.php?os=windows)
- Symfony CLI local server: it runs php-fpm, php-cgi or php-cli. php-cgi is started as `php-cgi -b <port>`, with a Windows-specific error-log path. The fallback is `php -S 127.0.0.1:<port>`. Source: [symfony-cli local/php/php_server.go](https://github.com/symfony-cli/symfony-cli/blob/main/local/php/php_server.go)
- Symfony docs: "PHP-FPM must be installed locally for the Symfony server to utilize." Source: [Symfony CLI docs](https://symfony.com/doc/current/setup/symfony_cli.html)
- Herd for Windows runs native PHP and nginx binaries. Source: [Laravel News](https://laravel-news.com/laravel-herd-for-windows-is-now-released)
- Conflict with the briefing: it suggested PHP_FCGI_CHILDREN might not work on Windows. In php-src master it works as a static pool of up to 64 processes. Source: [php-src sapi/cgi/cgi_main.c](https://github.com/php/php-src/blob/master/sapi/cgi/cgi_main.c)

### Inferences
- Herd, and any nginx-based Windows stack, must use php-cgi or its own FastCGI host, because FPM is absent. Not verified in Herd's binaries. Confidence: medium-high.
- On Windows, the Symfony CLI uses php-cgi when present (no FPM exists), otherwise `php -S`. Read from the code, not run. Confidence: medium-high.
- What a native FreeUnit would add on Windows: a dynamic process pool, per-application isolation, a configuration API, and one supervisor instead of IIS FastCGI settings, a stack launcher or a static php-cgi pool. Confidence: medium.
- A static php-cgi pool of a few children is exhausted by a few slow requests (Drupal batch jobs, cron, a paused Xdebug session). Read from the code, not measured. Confidence: medium.

### Gaps
- Microsoft Learn IIS FastCGI pages (maxInstances, instanceMaxRequests defaults) were not read.
- Herd, Laragon and XAMPP internals (php-cgi or mod_php, pool sizes) were not inspected.
- Whether spawn-fcgi has a Windows build was not checked.
- When Windows PHP_FCGI_CHILDREN support entered php-src was not traced; it needs the full git history.

## 5.8 Demand analysis and strength of evidence

### Takeaway
Demand is real but mostly unquantified for PHP and Drupal developers. All developers: about half work on Windows. Windows-native PHP stacks draw millions of downloads a year. Among DDEV users, one in four platform rows is a Windows host, and most of those already use WSL2. Nothing public measures how many PHP developers are blocked from WSL2 or Docker, which is the segment where a native FreeUnit has no real competitor besides XAMPP, Laragon and Herd.

### Cited Findings
- [ALL DEVS] Windows 49.5% for professional use, WSL 16.8% (2025, 31,569 responses). Source: [Stack Overflow 2025](https://survey.stackoverflow.co/2025/technology#most-popular-technologies-op-sys)
- [ALL DEVS] PHP used by 18.9% of 2025 respondents. Source: [Stack Overflow 2025](https://survey.stackoverflow.co/2025/technology#most-popular-technologies-language)
- [PHP DEVS] Development OS: no published split. Computed from the Stack Overflow 2025 dataset: 54.0% of 5,298 professional respondents who have worked with PHP use Windows at work, 35.3% without WSL (section 5.1). The last published figure is Windows 42% in a 2010 Zend survey of 2,000 developers. Source: [InternetNews](https://internetnews.com/software/windows-is-the-choice-of-enterprise-developers)
- [PHP DEVS] Deployment: 13.15% deploy PHP on Windows; IIS 4.50% (561 responses, late 2024). Source: [Perforce 2025 PHP Landscape Report](https://www.zend.com/system/files/2025-06/report-zend-2025-php-landscape-report_0.pdf)
- [DDEV USERS] 25.2% Windows hosts (computed from the platform table), 85% of them via WSL2; Drupal 10 and 11 are 32.0% of project types. Source: [DDEV Live Usage Statistics](https://ddev.com/usage-stats/)
- [DOWNLOADS] 2025: XAMPP 16,398,979 Windows downloads; WampServer 1,221,845. Laragon v7 and v8: 1,807,474 asset downloads since 2024-12-16. Sources: [XAMPP stats](https://sourceforge.net/projects/xampp/files/stats/json?start_date=2025-01-01&end_date=2025-12-31), [WampServer stats](https://sourceforge.net/projects/wampserver/files/stats/json?start_date=2025-01-01&end_date=2025-12-31), [Laragon releases](https://github.com/leokhoa/laragon/releases)

### Inferences
- Evidence strength. Strong: the all-developer OS share (two large surveys agree). Strong but biased: DDEV telemetry (opt-in, Docker users only). Weak: PHP-specific OS share (no current public data). Weak: download counts (not users). Absent: how often enterprises block WSL2 or Docker. Confidence: high.
- Likely audience segments, largest first: (1) learners and small shops on XAMPP, WampServer and Laragon; (2) Windows developers who avoid WSL2 and Docker for speed or simplicity, which is Herd's pitch; (3) enterprise developers blocked from virtualization or Docker Desktop licences; (4) Windows production shops on IIS, which are few. The order is a judgement, not measured. Confidence: low-medium.
- For Drupal, the recommended tool already works on Windows through WSL2 for most DDEV users. A native FreeUnit wins only where WSL2 is blocked or unwanted, or where it is simply faster to start. Confidence: medium.
- PHP developers follow the all-developer split: about half work on Windows, and about a third use Windows without WSL (computed, section 5.1). DDEV's 53.2% macOS share against that 54% Windows share suggests Docker-based PHP tooling skews to Mac, or that many Windows PHP developers use non-Docker stacks. Confidence: medium.

### Gaps
- Most useful next step: repeat the Stack Overflow cross-tab for respondents whose main language is PHP, and the same cut in the JetBrains Data Playground.
- Second step: a three-question poll in DDEV and Drupal community channels: Windows or not; WSL2 or Docker blocked at work; would a native Windows server help.
- Third step: ask the Laravel team for Herd Windows install numbers, or watch for a public figure.

## 6.1 Precedents: what comparable projects learned from their Windows ports

### Takeaway
Ports that add Windows to a fork()-based POSIX design remain limited for decades (nginx "beta" since 2009, PostgreSQL still fixing shared-memory reattach bugs in 2026), and ports carried by one sponsor die when the sponsor leaves (Redis by Microsoft Open Tech 2011 to 2016, Envoy 2020 to 2023). The one precedent closest to FreeUnit, FrankenPHP, shipped native Windows in March 2026 by linking the official MSVC-built PHP (TS build plus php8embed.lib) with Visual Studio's clang and lld-link, after a MinGW attempt failed on C runtime mismatches.

### Cited Findings

NGINX Unit upstream (the project FreeUnit forks)
- In nginx/unit issue 604 "Native Windows build" (2021-11-26) the maintainer answered: "Unit doesn't support native Windows interfaces and never will. It has architecture based on POSIX principles and highly tuned for *nix kernels. There's just no way to run it on Windows without WSL2." and "Cygwin wouldn't help. It's just limited and slow emulation of POSIX". ([nginx/unit#604](https://github.com/nginx/unit/issues/604))
- In nginx/unit issue 1008 (2023-11-23) a user could not build a Go application for Unit on Windows; the maintainer replied "Windows is not a supported platform to run Unit" and recommended "a dev container as WSL is not an option for you". ([nginx/unit#1008](https://github.com/nginx/unit/issues/1008))
- An offline full-text search of the archived nginx/unit tracker (1,087 issues and 467 pull requests, 2017-09-06 to 2025-10-08) finds 6 items with Windows, WSL, MinGW or Cygwin in the title; only issue 604 asks for a native build. Method: SQLite FTS5 query `title:windows OR title:win32 OR title:wsl OR title:mingw OR title:cygwin` over the archive dump. ([nginx/unit issues](https://github.com/nginx/unit/issues))

nginx
- nginx 0.7.52 (20 Apr 2009): "Feature: the first native Windows binary release." ([nginx CHANGES-0.7](https://nginx.org/en/CHANGES-0.7))
- The current Windows page says the Windows version "is considered to be a beta version"; "Only the select() and poll() (1.15.9) connection processing methods are currently used"; known issues: "Although several workers can be started, only one of them actually does any work." and "The UDP (and, inherently, QUIC) functionality is not supported." Listed as possible future enhancements: "Running as a service", "Using the I/O completion ports as a connection processing method", "Using multiple worker threads inside a single worker process." ([nginx for Windows](https://nginx.org/en/docs/windows.html))
- Maxim Dounin on the IOCP event module, 2013-11-28: "It's incomplete and doesn't work." ([nginx mailing list](https://mailman.nginx.org/pipermail/nginx/2013-November/041262.html)). On Windows limits, 2011-07-18: "it's how nginx under Windows works, not Windows bugs." ([nginx mailing list](https://mailman.nginx.org/pipermail/nginx/2011-July/028040.html))
- W3Techs, data dated 2026-10-07: of websites using nginx, Unix 98.8% and Windows 1.6% (a site may use more than one OS). ([W3Techs nginx by OS](https://w3techs.com/technologies/segmentation/ws-nginx/operating_system))
- freenginx still ships Windows zips for both branches: freenginx/Windows-1.31.4 (mainline) and freenginx/Windows-1.30.1 (stable). ([freenginx download](https://freenginx.org/en/download.html))

Apache httpd
- "The Apache HTTP Server Project itself does not provide binary releases of software, only source code." The docs point to Apache Lounge and WampServer. On Windows "there are usually only two httpd processes running: a parent process, and a child which handles the requests. Within the child process each request is handled by a separate thread." It installs as a service with `httpd.exe -k install`. ([Using Apache HTTP Server on Microsoft Windows](https://httpd.apache.org/docs/2.4/platform/windows.html))
- Apache Lounge offers httpd 2.4.69 binaries dated 2 Oct 2026, built with "Visual Studio C++ 2026 aka VS18", Win64 and Win32 (no ARM64 listed), requiring the "14.51.36247 Visual C++ Redistributable Visual Studio 2017-2026". ([Apache Lounge download](https://www.apachelounge.com/download/))

PostgreSQL
- November 2002: PeerDirect's native port "passes all postgres regression tests. It's been in BETA since early August with about 27 beta customers"; the project chose to wait for it and planned "a full 4-6 months of development, followed by 2 months of beta". ([pgsql-hackers, 2002-11-08](https://www.postgresql.org/message-id/200211080454.gA84sJh00510%40candle.pha.pa.us))
- July 2003: a contributor asked whether the Win32 port would make 7.4 at all. ([pgsql-hackers, 2003-07-14](https://www.postgresql.org/message-id/A02DEC4D1073D611BAE8525405FCCE2B027DFE%40harris.memetrics.local))
- PostgreSQL 8.0.0 (2005-01-19): "This is the first PostgreSQL release to run natively on Microsoft Windows as a server. It can run as a Windows service." and "the Windows port does not have the benefit of years of use in production environments ... it should be treated with the same level of caution as you would a new product." Earlier releases needed Cygwin. ([PostgreSQL 8.0.0 release notes](https://www.postgresql.org/docs/release/8.0.0/))
- EXEC_BACKEND: "the child process is launched by fork() + exec() (or CreateProcess() on Windows). It does not inherit the state from the postmaster, so it needs to re-attach to the shared memory, re-initialize global variables, reload the config file etc." ([launch_backend.c](https://doxygen.postgresql.org/launch__backend_8c_source.html))
- 2019-04-09, Noah Misch: "We have long had reports of intermittent 'could not reattach to shared memory' errors on Windows"; the fix reserves a second region for allocations that collided with the shared memory address, back-patched to 9.4. ([commit thread](https://www.postgrespro.com/list/thread-id/2436262))
- 2026-09-25, bug 19719 against PostgreSQL 18.6: with huge_pages=on on Windows, backends reattach shared memory without FILE_MAP_LARGE_PAGES, so only the postmaster gets large pages. ([BUG #19719](https://www.postgresql.org/message-id/19719-9eea21ac8841c776%40postgresql.org))
- The community wiki lists Windows-only problems: antivirus software interferes "because PostgreSQL requires file access commands in Windows to behave exactly as documented by Microsoft", and services may fail above about 125 connections due to desktop heap limits. ([Running and Installing PostgreSQL On Native Windows](https://wiki.postgresql.org/wiki/Running_%26_Installing_PostgreSQL_On_Native_Windows))

Redis
- 2011-12-09, antirez declined to merge Microsoft's win32 patch: "handling a win32 port directly in the main project means to delay everything else for the little gain"; he proposed "a win32 port as a separated project, with a different set of developers, and not officially supported by the main project". ([Redis for win32 and the Microsoft patch](https://oldblog.antirez.com/print.php?postid=244))
- The Microsoft port's last releases are win-3.2.100 and win-3.0.504, both 2016-07-01; the repository is archived and says "This project is no longer being actively maintained" and points to Memurai. ([microsoftarchive/redis releases](https://github.com/microsoftarchive/redis/releases); [microsoftarchive/redis](https://github.com/microsoftarchive/redis))
- Memurai Developer Edition "Requires a restart after 10 days" and its use "in a production environment is prohibited"; editions track Redis 8.2.0. ([Memurai](https://www.memurai.com/get-memurai))
- Garnet (Microsoft Research, announced 2024-03-18) is a new .NET cache server that speaks RESP and "achieves state-of-the-art performance on both Linux and Windows". ([Introducing Garnet](https://www.microsoft.com/en-us/research/blog/introducing-garnet-an-open-source-next-generation-faster-cache-store-for-accelerating-applications-and-services/))

Envoy
- Alpha on Windows in October 2020; production use from 1.18.3; led by a Microsoft engineer with the Envoy-Windows-Development group over about a year; Windows Server 2019 lacked edge-triggered notifications, so Envoy added "synthetic edge events". ([InfoQ, 2021-08-03](https://www.infoq.com/news/2021/08/envoy-proxy-on-windows))
- "On August 31, 2023 the Envoy project ended official Windows support due to a lack of resources." Patches are still accepted, but Windows is "excluded from Envoy CI, as well as the Envoy release and security processes." Building needs the Windows 10 SDK 1803 because of afunix.h. ([Envoy Windows requirements](https://www.envoyproxy.io/docs/envoy/latest/faq/windows/win_requirements))
- Not supported on Windows: watchdog, tracers, original source filter, hot restart, SXG filter, VCL socket interface. ([Envoy unsupported features](https://envoyproxy.io/docs/envoy/latest/faq/windows/win_not_supported_features))

Node.js (Windows support designed into the core from the start)
- Node 0.6.0 (2011-11-04): "Native Windows support using I/O Completion Ports for sockets." and "In the last version of Node, v0.4, we could only run Node on Windows with Cygwin." ([Node v0.6.0](https://nodejs.org/en/blog/release/v0.6.0))
- Today Windows x64 is Tier 1 and Windows arm64 Tier 2; the build needs Visual Studio 2022 or 2026, and "ClangCL is required to compile on Windows as of Node.js 24.0.0". ([Node.js BUILDING.md](https://github.com/nodejs/node/blob/main/BUILDING.md))

FrankenPHP (closest precedent: Go server embedding PHP through the embed SAPI)
- Issue 83 "Windows compatibility" opened 2022-11-01 and closed 2026-02-26 (31 reactions). ([php/frankenphp#83](https://github.com/php/frankenphp/issues/83))
- Pull request 2119 "feat: Windows support": opened 2026-01-09, merged 2026-02-26, 72 commits, 42 files, +532/-149 lines, building on an earlier community branch from 2024-12-22 (issue 1286). It "Supports linking to the official PHP release (TS version)" and links `-lphp8ts -lphp8embed` with library paths to both the unpacked release zip php-8.5.1-Win32-vs17-x64 (which holds php8embed.lib at its root) and the lib directory of php-devel-pack-8.5.1-Win32-vs17-x64 (headers and php8ts.lib), compiling cgo with the clang bundled in Visual Studio 2022 (component Microsoft.VisualStudio.Component.VC.Llvm.Clang), `-fuse-ld=lld`, Go 1.26, and vcpkg x64-windows for pthreads and brotli. ([php/frankenphp#2119](https://github.com/php/frankenphp/pull/2119); [php/frankenphp#1286](https://github.com/php/frankenphp/issues/1286))
- Release blog (2026-03-06, v1.12.0): the earlier MinGW build failed on a "Standard Library mismatch" between msvcrt.dll and the MSVC UCRT, with crashes in allocation and file descriptor passing; the fix was Visual Studio's clang frontend, "accepts GCC-like flags (which CGO likes) but uses the Microsoft STL and runtime libraries", plus a Google contribution in Go 1.26 for lld-link. It claims "3.6x" over nginx plus PHP-FPM on Windows Server 2022, and says WSL still gives the highest raw throughput. Sponsors: Intelligence X and Les-Tilleuls.coop. ([Windows support for FrankenPHP: it's finally alive](https://dunglas.dev/2026/03/windows-support-for-frankenphp-its-finally-alive/))
- The Windows release zip contains the FrankenPHP executable and "the official PHP binary for Windows"; services use WinSW; "Windows services cannot be reloaded". ([FrankenPHP production docs](https://github.com/php/frankenphp/blob/main/docs/production.md); [README](https://github.com/php/frankenphp/blob/main/README.md))
- The Windows CI job runs on windows-latest, downloads php-X.Y.Z-Win32-vs17-x64.zip and the matching devel pack, caches vcpkg archives with actions/cache, copies brotli DLLs and pthreadVC3.dll into the zip, and attests build provenance; there is no Authenticode signing step. ([windows.yaml](https://github.com/php/frankenphp/blob/main/.github/workflows/windows.yaml))
- Fifteen successful Windows workflow runs on pull requests (2026-09-19 to 2026-09-21) took 10.3 to 19.4 minutes, median 12.2 minutes (GitHub Actions API, run_started_at to updated_at). ([windows.yaml runs](https://github.com/php/frankenphp/actions/workflows/windows.yaml))
- Follow-up issues in the seven months after launch: antivirus and Defender flag the Windows zip (2026-05-24, closed as false positive; one user saw 21 of 64 VirusTotal engines); max_execution_time is always 0 on Windows (open); service integration (open); static builds and embedded apps are unavailable on Windows (open); caching vcpkg packages in CI (closed 2026-10-02). ([issue 2446](https://github.com/php/frankenphp/issues/2446); [issue 2474](https://github.com/php/frankenphp/issues/2474); [issue 2442](https://github.com/php/frankenphp/issues/2442); [issue 2501](https://github.com/php/frankenphp/issues/2501); [issue 2683](https://github.com/php/frankenphp/issues/2683))
- Performance work moved into php-src: a contributor found that the official dependencies are built with /GL so clang cannot use LTCG, the PGO profile is MSVC-only, and `/GUARD:CF` costs much of the gain; a php-src change was reported as "Merged in 8.6". ([php/frankenphp#2314](https://github.com/php/frankenphp/issues/2314); [php/php-src#21563](https://github.com/php/php-src/pull/21563))

RoadRunner, Caddy, Laravel Herd, Symfony CLI (no embedding, or Go)
- RoadRunner ships roadrunner-2025.1.15-windows-amd64.zip (release 2026-06-17). Workers are separate PHP CLI processes; the relay is "pipes, TCP ... or socket", and pipes are the default and "the fastest communication transport". ([RoadRunner releases](https://github.com/roadrunner-server/roadrunner/releases); [RoadRunner server plugin](https://docs.roadrunner.dev/docs/plugins/server))
- RoadRunner's early Windows bugs (2019-01: "Connections hangs under Windows", "Server crash under Windows after a number of pool restarts") and Windows CI added in 2021. ([issue 83](https://github.com/roadrunner-server/roadrunner/issues/83); [issue 84](https://github.com/roadrunner-server/roadrunner/issues/84); [pull request 706](https://github.com/roadrunner-server/roadrunner/pull/706))
- Caddy implements the Windows service API natively (service_windows.go imports golang.org/x/sys/windows/svc), and its docs show both `sc.exe` and WinSW; "Windows services cannot be reloaded", so `caddy reload` is used. ([service_windows.go](https://github.com/caddyserver/caddy/blob/master/service_windows.go); [Caddy: Keep Caddy running](https://caddyserver.com/docs/running))
- Laravel Herd for Windows was released 2024-03-26 and bundles PHP, nginx and Node.js as native binaries. ([Laravel News](https://laravel-news.com/laravel-herd-for-windows-is-now-released)). It needs Windows 10 or later and admin rights to install the HerdHelper service that edits the hosts file for .test domains; the docs suggest excluding `%USERPROFILE%\.config\herd` from Defender scans for speed. ([Herd installation](https://herd.laravel.com/docs/windows/getting-started/installation))
- Herd runs PHP as php-cgi.exe processes (a bug report shows `spawn C:\Users\...\.config\herd\bin\php83\php-cgi.exe ENOENT`), and on 2026-09-28 a user reported that Windows 11 Smart App Control blocks the PHP 8.4 and 8.5 extension DLLs ("An Application Control policy has blocked this file", Code Integrity event 3077). ([herd-community#1168](https://github.com/beyondcode/herd-community/issues/1168); [herd-community#1757](https://github.com/beyondcode/herd-community/issues/1757))
- Symfony CLI's local server runs php-fpm if present, else `php-cgi -b <port>` (with a Windows-specific error log path), else the built-in `php -S` server. ([local/php/php_server.go](https://github.com/symfony-cli/symfony-cli/blob/main/local/php/php_server.go))

lighttpd, HAProxy, h2o
- lighttpd 1.4.70 (2023-05-10) added a "native Windows build (experimental) (not packaged; no installer)". ([lighttpd 1.4.70](https://www.lighttpd.net/2023/5/10/1.4.70/)). It builds with MinGW or Visual Studio via autotools or CMake; on Windows there is no daemonize (it can run as a service), no multiple worker processes, no syslog, and mod_webdav is not ported. ([DevelWin32](https://redmine.lighttpd.net/projects/lighttpd/wiki/DevelWin32))
- HAProxy has a `TARGET=cygwin` build only; no native Windows target. ([HAProxy Makefile](https://github.com/haproxy/haproxy/blob/master/Makefile); [HAProxy INSTALL](https://github.com/haproxy/haproxy/blob/master/INSTALL))
- h2o issue 2 "Windows porting" has been open since 2014-09-05. ([h2o/h2o#2](https://github.com/h2o/h2o/issues/2))

Cygwin and midipix
- Cygwin's fork is "a non-copy-on-write implementation"; "fork will almost certainly always be inefficient under Win32"; ASLR "interferes with a proper fork"; "Current Windows implementations make it impossible to implement a perfectly reliable fork, and occasional fork failures are inevitable." ([Cygwin user guide, highlights](https://cygwin.com/cygwin-ug-net/highlights.html))
- The Cygwin DLL is LGPLv3 or later, with an exception for linking libcygwin.a with independent modules. ([Cygwin licensing](https://cygwin.com/licensing.html))
- midipix calls itself "/pre/alpha"; its latest news item is dated 2024-08-16. ([midipix](https://midipix.org/))

WSL2 (the status quo)
- Microsoft: "For the fastest performance speed, store your files in the WSL file system if you are working in a Linux command line"; keep the project in the Linux home directory, not under `/mnt/c/...`. ([Working across file systems](https://learn.microsoft.com/en-us/windows/wsl/filesystems))
- WSL uses NAT by default; mirrored mode needs Windows 11 22H2 or later and adds IPv6, localhost in both directions, "Improved networking compatibility for VPNs" and LAN access. ([Accessing network applications with WSL](https://learn.microsoft.com/en-us/windows/wsl/networking))
- Memory: the WSL 2 VM gets "50% of total memory on Windows" by default, plus swap of 25% of that memory; the `autoMemoryReclaim` setting is still listed under `[experimental]`, with the default `dropCache` (cached memory "reclaimed immediately"). ([Advanced settings configuration in WSL](https://learn.microsoft.com/en-us/windows/wsl/wsl-config))
- Intune and Group Policy settings include "Allow the Windows Subsystem For Linux" ("When set to disabled, this policy disables access to the Windows Subsystem For Linux for all users on the machine"). ([WSL Intune settings](https://learn.microsoft.com/en-us/windows/wsl/intune))
- Microsoft open-sourced WSL at Build on 2025-05-19/20, except lxcore.sys (WSL 1) and the 9P file system redirector (p9rdr.sys, p9np.dll). ([BleepingComputer](https://bleepingcomputer.com/news/microsoft/microsoft-open-sources-windows-subsystem-for-linux-at-build-2025))

### Precedent lessons table

| Project | Year | Approach | Effort | Outcome | Lesson for FreeUnit |
|---|---|---|---|---|---|
| nginx | 2009 to now | Native Win32 build of the Unix design; select/poll; MSVC under MSYS | Core team, low priority for 17 years | Still "beta"; one working worker; no IOCP; no service mode; 1.6% of nginx sites | A port without an IOCP event engine remains suitable only for development. Say so in the first release notes. |
| Apache httpd | 1990s to now | Dedicated Windows MPM (one child, many threads); ASF ships source only | Third parties (Apache Lounge) carry builds | Mature, widely used for local PHP stacks | A Windows-specific process model works; the binaries can live outside the project. |
| PostgreSQL | 2002 to 2005, ongoing | EXEC_BACKEND (CreateProcess plus shared-memory reattach), signal emulation | Company port (PeerDirect) plus about 2.5 years of community merge work | Shipped in 8.0 (2005); about 2.6% of master commits per year still mention Windows; ASLR reattach bug fixed only in 2019 | Fork emulation works but leaves a long tail of memory-layout bugs. |
| Redis (MS Open Tech) | 2011 to 2016 | Separate Microsoft fork; upstream refused the patch | Microsoft team | Abandoned after 3.2; archived; users moved to Memurai or Garnet | A sponsor-carried fork dies with the sponsor. |
| Envoy | 2020 to 2023 | Native port inside upstream, driven by Microsoft | About one year to production | Official support ended 2023-08-31 "due to a lack of resources" | Upstream support needs maintainers who stay; plan the exit. |
| Node.js | 2011 to now | Event layer designed for IOCP from the start (libuv) | Core design work | Windows x64 Tier 1, arm64 Tier 2 | Design the I/O abstraction for completion ports, not readiness. |
| FrankenPHP | 2022 request, 2026 ship | Link official MSVC TS PHP plus php8embed.lib with VS clang and lld-link | One 72-commit PR in 7 weeks, on top of a 2024 community branch, plus php-src changes | Shipped v1.12.0 (2026-03-06); open gaps: services, timeouts, static builds; AV false positives | The PHP-embedding path is proven only with the MSVC ABI. Budget for AV and signing. |
| RoadRunner | 2018 to now | PHP CLI workers over pipes; Go server | Moderate | Windows zips in every release | Separate PHP processes avoid the ABI problem but give up embedding. |
| Caddy | 2015 to now | Go; native service API | Small | First-class Windows binaries | A Windows service is a few hundred lines when designed in. |
| Laravel Herd | 2024 | Bundles nginx plus php-cgi.exe, helper service for hosts | Commercial team | Popular; hit Smart App Control blocking unsigned PHP DLLs (2026-09) | The local-dev market exists; unsigned DLLs are now a blocker on new Windows 11 installs. |
| Symfony CLI | 2019 to now | php-cgi on Windows (no php-fpm) | Small | Works for development | php-cgi is the default answer today; a native embedder must beat it. |
| lighttpd | 2023 | Native build, experimental, no installer | Single maintainer | Experimental after 3 years | Experimental status can persist; say what is missing. |
| HAProxy, h2o | n/a | Cygwin only, or nothing | n/a | No native Windows | High-performance servers often refuse Windows. |
| Cygwin, midipix | 1995 to now | POSIX layer with emulated fork | n/a | Fork is slow and "occasional fork failures are inevitable"; midipix pre-alpha | Do not ship FreeUnit on a fork emulation layer. |
| WSL2 | 2019 to now | Real Linux kernel in a VM | Microsoft | Default answer for PHP on Windows; slow on /mnt/c; can be disabled by policy | FreeUnit already runs there; native Windows is for users WSL does not serve. |

### Inferences
- High confidence: FrankenPHP shows that a server embedding PHP through the embed SAPI can ship on Windows by linking the official TS build and php8embed.lib with an MSVC-ABI compiler. The briefing's claim that FrankenPHP has no native Windows build is out of date.
- High confidence: every port that kept a fork()-based design either emulated fork with process re-creation (PostgreSQL) or ended up with one working worker process (nginx, lighttpd). Unit's router, controller and per-application processes communicate over socketpairs with descriptor passing and shared memory (configure probes `socketpair(AF_UNIX, SOCK_SEQPACKET)`, `msg_control`, `shm_open()` and `memfd_create()`, section 2.7), which is the hardest part to emulate; the core architecture study (another chapter) decides whether a Windows build is a port or a redesign.
- Medium confidence: a Windows build of FreeUnit should be scoped like Herd or Symfony CLI (local development server) and say so in its name or docs, as nginx does with "beta", rather than promising production parity.
- Medium confidence: upstream NGINX Unit refused native Windows in 2021 on architectural grounds; FreeUnit reversing that needs a named maintainer group, or it repeats the Envoy and Redis outcome.

### Gaps
- Apache httpd's Bugzilla requires a login for CSV lists, so the Windows share of httpd bugs was not counted. Settle by an authenticated Bugzilla query on op_sys for product "Apache httpd-2".
- No source gives the person-years PostgreSQL spent on the Windows port. Settle by counting Win32-tagged commits from 2003 to 2005 in a full clone.
- Laravel Herd's Windows internals (how many php-cgi processes, how they are supervised) are only visible through bug reports. Settle by installing Herd in a Windows VM and listing processes.
- How many people use nginx for Windows for local development (not production) is unknown; W3Techs counts public sites only.

## 7.1 Maintenance cost model and support tiers

### Takeaway
Countable evidence puts the ongoing Windows share at about 3% of tickets or commits for mature projects (nginx 3.2% of trac tickets 2011 to 2024, PostgreSQL about 2.6% of master commits in 2024 and 2025), plus a 10 to 20 minute CI leg per pull request. The larger risk is not the steady state but the maintainer count: Envoy and the Microsoft Redis port ended when their Windows maintainers left, so a FreeUnit Windows build should start as Tier 3 with a feature matrix and an explicit exit rule.

### Cited Findings
- nginx trac (2,643 tickets, 2011-09 to 2024-09): 85 tickets (3.2%) mention Windows, Win32, Win64, MinGW, MSYS or Cygwin in the summary or in the "uname -a" field, excluding Linux kernels (WSL). By year the count peaked at 16 (2014) and 15 (2020). Method: CSV export of all tickets and a regular expression; a heuristic, not a triage. ([nginx trac query](https://trac.nginx.org/nginx/query))
- nginx on GitHub (issues since the 2024 move): 29 of 642 issues and 49 of 932 pull requests match the word "windows" anywhere in the text, an upper bound. ([nginx/nginx issues](https://github.com/nginx/nginx/issues?q=windows))
- PostgreSQL master commits whose message mentions "windows": 75 of 2,208 in 2023 (3.4%), 71 of 2,720 in 2024 (2.6%), 74 of 2,819 in 2025 (2.6%). GitHub commit search covers only the default branch. Examples from 2025: "Fix O_CLOEXEC flag handling in Windows port.", "Drop support for MSVCRT's float formatting quirk.", "Fix "inconsistent DLL linkage" warning on Windows MSVC". ([GitHub commit search](https://github.com/search?q=repo%3Apostgres%2Fpostgres+windows&type=commits); [Searching commits](https://docs.github.com/en/search-github/searching-on-github/searching-commits))
- FrankenPHP's Windows job: median 12.2 minutes per pull request run (n=15, September 2026), on windows-latest with a vcpkg cache. ([windows.yaml runs](https://github.com/php/frankenphp/actions/workflows/windows.yaml))
- FreeUnit's current macOS job is a model for a first Windows job: it runs on macos-15, only on pull requests and pushes that touch configure, auto/** or src/**, and builds "The C test programs ... No modules, no pytest, no sudo". (.github/workflows/build-test-macos.yml:3-31@872bf041). No FreeBSD job exists at that revision (no FreeBSD runner in .github/ at 872bf041).
- Rust tiers: Tier 1 "will always build and pass tests" and needs at least 3 target maintainers; Tier 2 "will always build, but they may or may not pass tests" and needs at least 2; Tier 3 has "no guarantees". ([Rust target tier policy](https://doc.rust-lang.org/rustc/target-tier-policy.html))
- CPython PEP 11: Tier 1 "CI failures block releases"; Tier 2 needs a reliable buildbot and two core developers, breakage fixed or reverted within 24 hours; Tier 3 needs one core developer and "Failures on these platforms do not block a release." x86_64 and i686 Windows MSVC are Tier 1; aarch64 Windows MSVC is Tier 2; MinGW is not listed. ([PEP 11](https://peps.python.org/pep-0011/))
- Go: a first-class port needs "At least two developers" named to maintain it plus a builder maintainer; a broken port can be removed from the next release. ([Go porting policy](https://go.dev/wiki/PortingPolicy))
- Node.js: Tier 1 and Tier 2 test failures block releases; "Experimental" platforms "May not compile or test suite may not pass" and do not block releases. ([Node.js BUILDING.md](https://github.com/nodejs/node/blob/main/BUILDING.md))

### Inferences
- Medium confidence: steady-state cost estimate for FreeUnit after the initial port: about 3% of maintainer time on Windows-specific fixes (nginx and PostgreSQL ratios), one Windows CI leg of 10 to 20 minutes per relevant pull request (free on public repositories, see the CI chapter), plus release work for signing and package manifests. The initial port cost is not estimable from these precedents because it depends on the core architecture decision.
- Proposed support tiers for FreeUnit (medium confidence; modelled on Rust, PEP 11 and Node.js):
  - Tier 1: Linux x86_64 and aarch64 (glibc, musl). Full CI including modules and pytest; failures block merges and releases; packages shipped.
  - Tier 2: macOS (arm64) and FreeBSD. CI builds and runs the C tests (macOS already does; FreeBSD needs a job); failures block releases but not every merge; two named maintainers.
  - Tier 3: Windows x64 "development use". CI builds the core and the PHP module and runs the C tests and a pytest subset on pull requests that touch configure, auto/** or src/**; failures do not block releases; at least one named maintainer; a published feature matrix (event engine, TLS, PHP 8.4 and 8.5 x64 in TS and NTS, no isolation, no user switching, no unix-socket listeners until proven); and an exit rule like Go's: if the Windows job stays red for one release cycle with no owner, the build is dropped from the next release and the docs say so.
  - Promotion from Tier 3 to Tier 2 only with two named maintainers, an IOCP or equivalent event engine, a green full pytest run, and signed release artefacts.

### Gaps
- No public data gives FreeUnit-like projects' Windows CI cost in minutes per month; the CI chapter has the runner billing facts.
- The share of Windows-specific security fixes is not countable from the sources above. Settle by tagging past nginx and PostgreSQL security advisories by platform.

## 7.2 Open questions and the experiment that settles each

### Takeaway
Most open questions are cheap to settle with one Windows CI job, a clean Windows 11 VM and a community poll. Three gate the plan: whether configure and the module link work in MSVC mode, how the core's process and IPC model maps to Windows (architecture chapter), and how many PHP developers are actually blocked from WSL2 and Docker.

### Cited Findings
- No FreeUnit code has been built, linked or run on Windows at 872bf041: no workflow uses a Windows runner (.github/workflows at 872bf041), and `NXT_WINDOWS` is set but never read (auto/os/test:87@872bf041).
- Every chapter above lists its own gaps; this table merges and ranks them.

### Inferences
- Rank by how much each answer changes the plan: architecture and toolchain first, demand second, packaging details last. Confidence: medium.

### Gaps

| Rank | Open question | Why it matters | Experiment or reading that settles it | Section |
|---|---|---|---|---|
| 1 | How do fork, socketpair descriptor passing, shared memory and the event engine map to Windows? | Decides port versus redesign, and TS versus NTS PHP | Architecture chapter; prototype one router-to-worker channel on Windows | 0, 1.0, 3.5 |
| 2 | Does configure under the MSYS2 shell with `CC=clang-cl` get through auto/? How many probes and link lines fail? | Decides "extend auto/" versus "move to Meson" | One `windows-2025` job that keeps the configure log as an artifact | 2.7, 2.8 |
| 3 | Does a clang-cl PHP module link against php8ts.lib, load and serve one request? Does gcc fail on `@@N` names as predicted? | Confirms the compiler choice | 20-line link test plus a minimal SAPI module on `windows-2022` | 1.2, 1.7 |
| 4 | What does `fopen(name, "re")` do on the UCRT under PHP's invalid parameter handler? | A crash or a 404 on every request | Small test program on Windows; read main/main.c for the handler PHP installs | 1.4 |
| 5 | Which local control channel ships on Windows: AF_UNIX, named pipe or TCP loopback? | Security of the control API; test client support (CPython has no Windows AF_UNIX) | Decision in the architecture chapter; bind and connect tests on `windows-2025`, including abstract addresses | 3.5, 2.4 |
| 6 | How many PHP or Drupal developers have WSL2, Hyper-V or Docker blocked at work? | Size of the segment with no good alternative | Three-question poll in DDEV and Drupal channels; Stack Overflow main-language cross-tab; JetBrains Data Playground | 5.1, 5.5, 5.8 |
| 7 | Does a listener on 127.0.0.1 avoid the Windows Firewall prompt, for admins and standard users? | First-run experience; silent block rules look like a broken server | Clean Windows 11 VM: listen on 127.0.0.1, [::1] and 0.0.0.0, record prompts and rules | 4.4 |
| 8 | Does Smart App Control block unsigned PHP DLLs loaded by a signed unitd.exe? | The official PHP binaries carry no Authenticode signature; Herd users hit blocks | Windows 11 VM with Smart App Control on; install by browser, winget and Scoop | 4.3, 1.8, 6.1 |
| 9 | Will SignPath Foundation sign FreeUnit, including redistributed third-party DLLs, and how fast? | Free signing path for releases | Apply to SignPath Foundation early | 4.2 |
| 10 | Can FreeUnit get a legal entity or fiscal host in an Artifact Signing country? | Simplest paid signing path (USD 119.88 a year) | Governance decision; ask Azure about organization validation | 4.2 |
| 11 | Real job durations and cold vcpkg build time on `windows-2025` | CI budget and required-check timing | One scratch run of the win-core and win-php jobs from 3.7 | 3.4, 3.7 |
| 12 | Do `cargo check --target x86_64-pc-windows-msvc` for src/otel and tools/unitctl pass? | Rust parts in the first milestone or not | One Windows job | 2.4 |
| 13 | Prism overhead for an x64 FreeUnit on Windows on Arm, and when official ARM64 PHP builds appear | ARM64 timing | Benchmark on an ARM64 device; watch php-windows-builder and the PHP Foundation | 4.6, 1.1, 5.6 |
| 14 | Microsoft Build Tools licence terms for Linux CI cross builds | Whether a Linux MSVC-ABI build job is allowed | Read https://go.microsoft.com/fwlink/?LinkId=2086102 | 1.5 |
| 15 | WiX v7 EULA acceptance in unattended CI, and the OSMF FAQ | MSI step in 4.8 | Read the WiX v7 docs and https://opensourcemaintenancefee.org/ | 4.1 |
| 16 | Windows share of Apache httpd bugs; person-years of the PostgreSQL port | Sharper maintenance-cost estimate | Authenticated Bugzilla query; Win32 commit count 2003 to 2005 in a full clone | 6.1, 7.1 |
| 17 | How Laravel Herd supervises php-cgi on Windows | Benchmark target for the "better than php-cgi" claim | Install Herd in a Windows VM and list processes and pool sizes | 6.1, 5.7 |
