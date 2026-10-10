# Windows developer setup for Phase 0

This page is for a contributor who starts the Phase 0 experiments (docs/plan.md section 5) on a
Windows machine with Claude Code. Follow it top to bottom. Each step ends with a check.

## 1. Machine

- Windows 11 x64, version 24H2 or later (`winver` shows build 26100 or higher). The plan's floor is
  build 20348; E10, E16 and E19 need a second local standard user, so create one now.
- 40 GB free disk, 16 GB RAM. Developer Mode on (Settings > System > For developers) so `git` can
  create symlinks.
- Optional later: a Windows 10 22H2 VM and a Windows Server 2019 VM for E05 and E13.

## 2. Toolchain

Run in an elevated PowerShell:

```powershell
winget install Git.Git
winget install Microsoft.VisualStudio.2022.BuildTools --override "--quiet --wait --add Microsoft.VisualStudio.Workload.VCTools --add Microsoft.VisualStudio.Component.VC.Llvm.Clang --add Microsoft.VisualStudio.Component.VC.Llvm.ClangToolset --add Microsoft.VisualStudio.Component.Windows11SDK.26100"
winget install MSYS2.MSYS2
winget install GLab.GLab
winget install BurntSushi.ripgrep.MSVC
winget install Python.Python.3.13
```

Then:

- MSYS2: open "MSYS2 MSYS", run `pacman -Syu`, reopen, run `pacman -S make diffutils`.
- PHP: download the 8.5 **non-thread-safe x64 development package** (`php-devel-pack-8.5.x-nts-Win32-vs17-x64.zip`)
  and the matching binaries zip from https://windows.php.net/download/ into `C:\php85`.
- Check, from "x64 Native Tools Command Prompt for VS 2022":

```
clang-cl --version
lld-link --version
dumpbin /exports C:\php85\dev\php8.lib | find /c "@@"
```

The last line must print about 340 (plan E01). Record the exact numbers in the first results file.

## 3. Claude Code

The account must be Pro or higher; the free plan has no Claude Code access.

```powershell
irm https://claude.ai/install.ps1 | iex
claude --version
claude doctor
```

- Run `claude` once in the repository and log in through the browser with the claude.ai account.
- Keep Git for Windows installed: Claude Code uses Git Bash for its Bash tool and falls back to PowerShell without it.
  If `claude doctor` cannot find bash, add to `%USERPROFILE%\.claude\settings.json`:
  `{"env": {"CLAUDE_CODE_GIT_BASH_PATH": "C:\\Program Files\\Git\\bin\\bash.exe"}}`.
- Use Windows Terminal with PowerShell. Do not run Claude Code under WSL for this work: the experiments
  exercise native Win32 APIs, and WSL hides them.
- Optional: the Claude Code extension for VS Code (VS Code 1.94 or later).

Docs: https://code.claude.com/docs/en/setup

## 4. Repositories

The repositories live on GitHub (the source of truth) and are mirrored to GitLab at
https://gitlab.com/freeunit, synced from GitHub every hour. Clone from GitLab; a GitLab account is enough
to push branches and open merge requests there. Cloning over HTTPS needs no account at all.

```powershell
mkdir C:\src; cd C:\src
git clone https://gitlab.com/freeunit/freeunit-windows.git   # the plan, Phase 0 experiments, results
git clone https://gitlab.com/freeunit/freeunit.git           # the server; every plan citation is at 872bf041
```

In WSL, under `/home/<user>/src`:

```sh
git clone https://gitlab.com/freeunit/freeunit.git           # Linux build for comparisons
git clone https://gitlab.com/freeunit/freeunit-harness.git   # build, test, review and perf tooling for agents
```

GitHub originals, for issues, pull requests and the archived upstream: https://github.com/freeunitorg/freeunit-windows,
https://github.com/freeunitorg/freeunit, https://github.com/freeunitorg/freeunit-harness, and
https://github.com/nginx/unit. Background reading: https://github.com/freeunitorg/docs and
https://github.com/freeunitorg/freeunit4drupal. If `git clone` is blocked, every GitLab project has a zip under
`https://gitlab.com/freeunit/<name>/-/archive/main/<name>-main.zip` (`master` for freeunit).

Workflow: ask a maintainer to add you to the GitLab group as Developer. Push branches to the GitLab project and
open a merge request there; `master` and `main` are protected and follow GitHub. A maintainer ports accepted
changes to GitHub, and the next sync brings them back. Without any account: `git format-patch origin/main` in the
experiment branch and send the files.

## 5. First week

Start with the blocking experiments in this order, one directory `experiments/Exx-<name>/` each, raw output
in `results/<date>-<build>.txt` with Windows edition, build number and compiler version:

1. E03 clang-cl on atomics and GNU extensions (half a day; proves the compiler).
2. E01 PHP exports and a minimal SAPI link.
3. E02 configure under MSYS2 with clang-cl. Record every failing probe.
4. E04 AF_UNIX under WSAPoll and ProcessSocketNotifications.

Then E11, E12, E15 and the rest. Results are never edited after the fact; a wrong result gets a new file.

Working with the agent:

- Give it the experiment's question and expected result from the plan, and ask it to write the program,
  the build command and the README. Review the program before running it.
- Ask it to quote raw tool output in results files, not summaries.
- Hosted runners: `.github/workflows/experiments.yml` runs the same programs on `windows-2022` and
  `windows-2025`; add each experiment to the matrix when it builds locally.
- Commits: plain English, one experiment per commit, `Co-Authored-By: <full model name> <noreply@anthropic.com>`
  for the model that wrote the code. Open a pull request per experiment.

## 6. Linux side: WSL2, not Docker Desktop

Phase 0 itself is native Windows. Only E23 is a Linux reading (`nm` over the module `.so` files), and the
hosted `ubuntu-24.04` runner can do it. Everything else that touches Linux needs a real Linux user space:

- the FreeUnit harness (16 bash tools, 12 Python tools, `unit-build`, `dsh` for DeepSeek reviews, pytest);
- a Linux FreeUnit build to compare behaviour against (fork start time for E12, AF_UNIX ping-pong for E24).

Use WSL2 with Ubuntu 24.04. Docker Desktop on Windows runs on the WSL2 backend anyway, adds a license
prompt and a second layer, and gives no Win32 access. Install Docker Engine inside WSL only if you need the
`freeunit-test` Alpine container.

```powershell
wsl --install -d Ubuntu-24.04
```

Inside WSL:

```sh
sudo apt install -y build-essential clang make pkg-config libssl-dev libpcre2-dev python3 python3-pytest nodejs npm git gh
curl -fsSL https://claude.ai/install.sh | bash      # a second, separate Claude Code install for WSL
git clone https://gitlab.com/freeunit/freeunit.git ~/src/freeunit
git clone https://gitlab.com/freeunit/freeunit-harness.git ~/src/freeunit-harness
```

Rules:

- Keep Linux clones under `/home/<user>` in WSL, never under `/mnt/c`. Searches across `/mnt/c` are slow and
  incomplete, and builds there are several times slower.
- Run Claude Code native for the experiments and in a WSL terminal for the harness. They are two installs
  with separate settings; log in to both.
- WSL2 numbers are functional references only. Its kernel is virtualised and its network is NAT, so do not
  publish WSL timings as the Linux baseline for E12 or E24; take those from a Linux box or the hosted runner.

## 7. Git: two clones, GitHub in the middle

Keep one clone per side and never share a working tree across `/mnt/c`. The native clone is for the
experiments, the WSL clone for the harness, `dsh` and Linux comparisons. Pull requests on GitHub are the
only sync point between them.

Native Windows, once, before the first clone:

```powershell
git config --global core.autocrlf false    # configure and the 54 auto/ scripts are sh; a CR breaks them under MSYS2
git config --global core.symlinks true     # freeunit-harness tracks 3 symlinks (CLAUDE.md, .agents/skills, .claude/skills); needs Developer Mode
git config --global core.longpaths true
git config --global user.name "<name>"; git config --global user.email "<email>"
glab auth login                             # or plain git over HTTPS; the token lands in Git Credential Manager
```

WSL, once:

```sh
git config --global user.name "<same name>"; git config --global user.email "<same email>"
glab auth login                             # a second token for the Linux side, or reuse Windows GCM:
# git config --global credential.helper "/mnt/c/Program\ Files/Git/mingw64/bin/git-credential-manager.exe"
```

Workflow per experiment: branch in the native clone, commit, push to GitLab and open a merge request; review the diff from the WSL
clone with `dsh` (`npm install -g @deepseek-ai/dsh`, sessions land in `~/.dsh/sessions`, which the harness
tools read) and with the harness; push fixes from whichever side made them, after `git pull --rebase` on
the other. Check line endings before the first push: `git ls-files --eol | grep -c crlf` must print 0.

`dsh` is a Node CLI with no platform restriction in its package metadata. Run it in WSL anyway: the
harness scripts that read its sessions are bash and Python with Linux paths.
