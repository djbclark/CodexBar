# Fork-only doctrine — djbclark/CodexBar (never in an upstream PR)

This file is ours, and it exists in two places on purpose:

- **This live copy** (untracked, and listed in `.git/info/exclude`) — what any
  agent working in this clone reads. It is never committed on a PR branch.
- A recovery mirror on the fork-only branch `djb/fork` of `djbclark/CodexBar`,
  at `fork/AGENTS.DJB.md`. Plektra keeps fork-only files (`plk/base`:
  `.forgejo/`, `scripts/djb-*`) the same way.

**Never use `djb/fork` as the base of a PR against `steipete/CodexBar`, and never
include this file in one.** If the mirror and this file ever differ, this one
reflects the newest ruling.

1. **Never build or test CodexBar on this Mac** (djbclark, 2026-10-14). The
   payload is a macOS app (AppKit / SwiftUI / WidgetKit, Xcode 27), and
   **forgejo.mit.edu cannot build it**: SIPB's Hyades runners are Linux only —
   `runs-on: ubuntu` (also `alpine`, `rocky`), no macOS, no Xcode (skill
   `forgejo-ci`, 2026-10-14, item 1.1/1.3). So the verifier is **GitHub-hosted
   macOS CI**: push the branch to our fork and/or open the upstream PR —
   upstream's `.github/workflows/ci.yml` builds and tests on macOS with Xcode 27.
2. Consequently the Plektra-style Hyades plumbing (`scripts/djb-ci-run`,
   `.forgejo/workflows/...`) does **not** apply to this repo and must not be added
   for the app target. If a Hyades job is ever wanted here it can only cover
   `swift build --product CodexBarCLI` and the Linux suite (`TestsLinux`); say so
   plainly instead of implying the app was built.
3. Remotes: `origin` = `steipete/CodexBar` (upstream, fetch; PR target) and
   `djbclark` = our fork (push branches there, then PR fork → upstream).
4. Fork-only files (this file, any fork tooling) never go into an upstream PR.
   Keep a PR branch to the single upstream-relevant change; run
   `git diff --stat origin/main...HEAD` before pushing and check nothing fork-only
   is in it.
5. Upstream wants focused tests plus `make check`; because of rule 1 those run in
   CI, not here. Put the commands that CI runs in the PR body, and say that CI is
   the verification.
