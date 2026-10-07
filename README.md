# FreeUnit for Windows: research and plan

This repository holds the research and the plan for a native Windows build of
FreeUnit (https://github.com/freeunitorg/freeunit), the community fork of
NGINX Unit. It does not hold the port. Code changes land in the FreeUnit
repository as pull requests. This repository keeps the plan, the Phase 0
experiments and their results, the CI recipes and the feature matrix.

Status: research and plan, 2026-10-07. No Windows code exists yet.

The short answer: the research concludes that FreeUnit can be made to run
natively on Windows 11 and Windows Server 2022 or later, once its process and
IPC layer is rebuilt. The plan
recommends a conditional go: ten weeks of small experiments and a demand poll
first, then a Tier 3 "development use" build whose first demo serves Drupal
through the PHP module on Windows 11 x64.

## Layout

| Path | Content |
|---|---|
| docs/research-report.md | What the research found, and how the conflicts between the notes were settled. |
| docs/plan.md | Decisions, architecture, Phase 0 experiments, milestones, code rules, risks and open decisions. |
| research/history-upstream-and-fork.md | Upstream's position on Windows, demand in the trackers, nginx's Windows history. |
| research/portability-audit.md | Every Unix dependency of the core and libunit at revision 872bf041, with grades and a size estimate. |
| research/process-and-ipc-design.md | A Windows design for processes, ports, shared memory, signals and modules. |
| research/windows-io-model.md | Windows I/O interfaces, the event engine options and the platform floor. |
| research/toolchain-packaging-demand.md | Compiler, PHP builds, build system, CI, packaging, signing, demand and precedents. |
| LICENSE | Apache License 2.0. |

Later pull requests add `experiments/`, `ci/`, `matrix/`, `decisions/` and
`.github/workflows/`, as described in docs/plan.md section 8.

## How to read it

1. Read the first paragraph of docs/research-report.md. It answers the main
   questions.
2. Read section 1 of docs/plan.md for the recommendation and the gate that
   decides it.
3. Read section 5 of the report for the four places where the research notes
   disagreed, and the evidence that settled each.
4. Use docs/plan.md sections 3 to 6 as the working plan: decisions D1 to D20,
   the architecture, experiments E01 to E24 and milestones M1 to M8.
5. Go to research/ for the cited evidence. Code references read
   `path:line@872bf041` and point into
   https://github.com/freeunitorg/freeunit at that revision.

## Contributing

- One experiment per pull request: the program, the build command, and the
  raw output with the Windows edition, the build number and the compiler
  version. Do not edit results after the fact.
- A result that contradicts the plan changes the affected decision in
  docs/plan.md in the same pull request, with the reason.
- Code for the port goes to https://github.com/freeunitorg/freeunit and
  follows the code rules in docs/plan.md section 7.
- Write plain English: short sentences, exact names and numbers, full URLs.
  Mark every claim nobody has tested as unverified.
- Publish only public material: no private host names, no local paths, no
  credentials.

## License

Apache License 2.0. See LICENSE.
