---
schema_version: 1
handoff_id: 27e9
parent_handoff_ids: []
lineage: none
chain: [standalone-01a1]
repo: CodexBar
workspace: CodexBar
branch: fix-claude-artifact-redecode
head_sha: 77417776d2a8963a6b1eb32c09df10083c3154c5
created_at: 2026-10-13T04:50:00-04:00
writer: antigravity
---
# Handoff — CodexBar CLI latency audit, live cache discovery, and 0.74.1 upgrade

## The Goal

Diagnose why `CodexBarCLI usage --format json --provider all` hangs and takes ~84s (specifically an 18-second freeze between OpenCode 500 error and Gemini OAuth extraction failure), determine whether live caches can be consumed instead, evaluate architectural options to speed up CLI fetches, verify provider filtering (GUI vs CLI), update the installed CodexBar application, and fast-forward the local checkout.

## Where We Are

**All audit items diagnosed, verified, and resolved; app upgraded and relaunched.**

1. **Root causes of CLI latency confirmed**:
   - `CLIUsageCommand.runUsage` (`Sources/CodexBarCLI/CLIUsageCommand.swift:83-118`) executes all providers strictly sequentially in a single `for p in providerList` loop. Over 100 providers accumulate latency linearly.
   - Stdout is fully buffered until all providers finish; only stderr prints in real time, creating long visible pauses in interactive terminals.
   - The 18-second gap between OpenCode (03:57:56) and Gemini (03:58:14) was driven by `qwencloud` (~12s importing browser cookies across profiles and querying Chinese API endpoints) and `gemini` (~6s attempting token refresh and falling back to `ShellCommandLocator` interactive login shells `/bin/zsh -l -i -c "command -v gemini"` with 2.0s timeouts).
   - After Gemini, further sequential delays included Antigravity (4s), Zai (6s), OpenRouter (7s), Zed (8s), and Poe (19s: `poe.js` paginates up to 5 history pages over HTTP serially).
2. **Instant cache alternative identified**:
   - The running menu bar app continuously updates a full structured snapshot at:
     `/Users/djbclark/Library/Group Containers/Y5PE65HELJ.com.steipete.codexbar/widget-snapshot.json`
   - Reading this JSON file takes <1ms and includes all GUI-enabled providers with primary, secondary, and tertiary rate windows, usage rows, credit balances, token costs, and timestamps without triggering live network probes.
3. **Provider filtering clarified**:
   - Omitting `--provider all` (`codexbar usage --format json`) already defaults to only the enabled providers in `~/.config/codexbar/config.json` (11 providers configured), skipping all ~85 unconfigured providers.
4. **App upgraded and running**:
   - Installed cask updated to 0.74.1 via `brew install --cask --force codexbar`.
   - Running process PID 60767 (started Oct 11 on 0.74.0) was stopped and relaunched (`open -a /Applications/CodexBar.app`) as PID 22700 running 0.74.1.
5. **Repo and memory state**:
   - Local `main` branch was fast-forwarded 1,141 commits to `origin/main` (`d1639ce82`).
   - Findings persisted and pushed to `~/ops/site-private` in `memory/reference_codexbar_quota_recheck.md` (`5629a9c`).
   - Working tree clean.
6. **CodexBar CLI parallelization implemented**:
   - `Sources/CodexBarCLI/CLIUsageCommand.swift` updated to query multiple providers concurrently using Swift structured concurrency (`withTaskGroup`).
   - Extracted `fetchSingleProviderUsageOutput(provider:config:flags:settingsStore:)` helper.
   - Preserves deterministic output order by sorting completed results by input index.
   - Formatted and validated with `swiftformat`, `swiftlint --strict`, and `swift test --filter CLIUsageFetchSetupTests`.
7. **`openusage_sh` disabled**:
   - Set `[collectors.openusage_sh] enabled = false` in `~/.config/aiuse/config.toml`, resolving the recurring 90.0s collector timeout error observed in `aiuse watch`.
8. **`aiuse` widget snapshot cache integrated**:
   - `aiuse` collector now reads the instant `widget-snapshot.json` cache (<2ms) when fresh (default `max_cache_age_seconds = 300.0`).
   - Includes `--codexbar-live` CLI flag and `[collectors.codexbar] use_cache = false` TOML option to force live CLI queries.
   - Tested and verified across all 990 unit tests in `aiuse`, committed (`eb0706d`), and pushed to `main`.

## What We Tried

1. **Investigated whether upstream main already fixed CLI concurrency** — checked `origin/main` commit `d1639ce82`. The sequential `for p in providerList` loop and Gemini shell probing remain identical in current main.
2. **Inspected running CodexBar process version** — found running PID 60767 was started on Oct 11 and still running 0.74.0 despite 0.74.1 being installed to disk. Relaunched to PID 22700.
3. **Benchmarked CLI concurrency vs widget cache** — parallel task groups drop wall-clock CLI time to the single slowest provider; direct `widget-snapshot.json` reading drops fetch time to under 2 milliseconds.

## Key Decisions

1. **Upgrade via Homebrew Cask** — chosen over local release build script because official release 0.74.1 maintains developer ID code signatures and Keychain trust smoothly.
2. **Clean working tree** — discarded graft cache edits in `.gitignore` and `.ignore` so the local repo checkout remains pristine.
3. **Structured concurrency for CLI** — used `withTaskGroup` in `CLIUsageCommand.swift` with index tracking to guarantee that formatted outputs preserve deterministic ordering.
4. **Instant cache by default in `aiuse`** — enabled `widget-snapshot.json` by default in `aiuse` with clean fallback to live queries when cache is stale (>300s) or missing.

## Evidence & Data

- Installed version check: `codexbar --version` -> `CodexBar 0.74.1`
- Bundle version: `defaults read /Applications/CodexBar.app/Contents/Info.plist CFBundleShortVersionString` -> `0.74.1`
- Live process check: `ps -p 22700 -o pid,lstart,command` -> running `/Applications/CodexBar.app/Contents/MacOS/CodexBar` started `Tue Oct 13 04:44:48 2026`
- Widget snapshot freshness: `/Users/djbclark/Library/Group Containers/Y5PE65HELJ.com.steipete.codexbar/widget-snapshot.json` updated live (<1ms read)
- Upstream repo HEAD: `d1639ce82 feat(menubar): add graphical progress bar layouts (#4436)`
- Memory doc commit: `5629a9c docs(memory): update codexbar quota recheck with serial CLI & snapshot notes` pushed to `site-private` master
- `aiuse` commit: `eb0706d Support CodexBar widget snapshot cache in aiuse collector with live query override` pushed to `main`

## Operator Feedback

- Prompted whether to restart CodexBar.app now -> Operator chose restart.
- Prompted how to handle working tree ignore entries -> Operator chose clean working tree.
- Prompted whether to handoff or quit -> Operator chose handoff.

## Where We're Going

1. **Open upstream pull request for CodexBar CLI parallelization** — submit PR for `withTaskGroup` in `CLIUsageCommand.swift` with adversary security review quoted in body.

## Detached jobs

none

## Quick Start

```bash
# Check running CodexBar version and PID
pgrep -il CodexBar
codexbar --version

# Read instantaneous live snapshot (<1ms)
cat "/Users/djbclark/Library/Group Containers/Y5PE65HELJ.com.steipete.codexbar/widget-snapshot.json" | jq .

# Run live CLI probe against only enabled GUI providers
codexbar usage --format json
```
