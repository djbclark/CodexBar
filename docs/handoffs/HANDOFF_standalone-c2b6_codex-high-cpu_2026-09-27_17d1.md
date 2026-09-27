---
schema_version: 1
handoff_id: 17d1
parent_handoff_ids: []
lineage: none
chain: [standalone-c2b6]
repo: CodexBar
workspace: CodexBar
branch: fix-claude-artifact-redecode
head_sha: fb7547a4c98002fffdb76606bfc2733878b417f9
created_at: 2026-09-27T16:06:00-04:00
writer: Codex
---
# Handoff — CodexBar high CPU (all-history priority-day enumeration)

## The Goal

Stop CodexBar from pegging CPU on a ~5-minute cadence; identify whether an
external poller (launchd/jobber/hermes/~/ops) was involved; land fixes on the
fork PRs and update GitHub issues/PRs with evidence.

## Where We Are

**Fixed locally; upstream merge pending.**

1. Root cause confirmed: CodexBar's own all-history cost refresh (not external
   polling). `CostReportingPeriod.allTime` + calendar-day enumeration in priority
   metadata merge produced ~739k day keys and a ~15.7 MB dense priority payload
   on 0.67.0.
2. Fix is on fork branches and pushed:
   - [#4045](https://github.com/steipete/CodexBar/pull/4045) `fix-infinite-loop-daykeys` @ `899f04617`
   - [#4053](https://github.com/steipete/CodexBar/pull/4053) `fix-claude-artifact-redecode` @ `fb7547a4c` (includes #4045 merge)
3. Local packaged build running since ~13:58 EDT:
   `/Users/djbclark/src/CodexBar/CodexBar.app` (0.68.1 build 160), PID stable,
   ~1.5% CPU steady. Cask at `/Applications/CodexBar.app` not running.
4. Live cache after ~2h on fixed build: `turnIDsByDay` 739,526 → **0**,
   priority payload 15.7 MB → **1.1 KB**, `usage_rows` **10,085** restored.
5. ClawSweeper re-reviewed after follow-up comments (~15:53 EDT): #4045 at
   diamond lobster (5/6), #4053 at gold shrimp (3/6); both "ready for maintainer
   look". Both PRs had **merge conflicts** with `main` at last check.

## What We Tried

1. **Suspected external poller** — audited launchd/jobber/hermes/~/ops. Only
   unrelated hourly `com.djbclark.aiuse` from `~/ops/site-djbclark`. Not the cause.
2. **Local `dayKeys` clamp to Jan 2020** — written then **discarded**; maintainer
   had already pushed sparse recorded-day reconciliation (`c5c253abd`) to PR branches.
3. **`compile_and_run.sh --test`** — full suite **129/130 groups pass** (~48 min);
   failed on pre-existing `MenuBarLayoutRendererTests` 50ms perf gate, so packaging
   did not run until a second `./Scripts/compile_and_run.sh` without `--test`.
4. **Scratch CLI upgrade trace** — dense 0.67.0 DB copy restamped to 0.68.0 hash;
   `CodexBarCLI cost --provider codex --period all --refresh` in 3.05s, 10,085 rows
   preserved. Did not compact empty days until turns reconciled (expected).

## Key Decisions

| Decision | Rejected alternative | Why |
|---|---|---|
| Sparse recorded-day reconciliation (remote `c5c253abd`) | Re-implement local clamp/loop fix | Maintainer already landed the correct approach on PR heads |
| Add `98de5f52231e524e` (0.68.0) as compatible predecessor | Rebuild all caches on upgrade | ClawSweeper P1; avoids wiping rows/checkpoints for 0.68.0 users |
| Merge #4045 adoption work into #4053 branch | Keep PRs strictly independent | #4053 ClawSweeper blocked on saved-cache upgrade proof |
| Leave `aiuse` launch agent alone | Disable hourly agent | Unrelated to CodexBar CPU; managed by ~/ops intentionally |
| Do not add 0.67.0 hash (`86c0a89614a11da9`) as predecessor | Zero-downtime 0.67→fix upgrade | Out of scope; user's DB rebuilt/rescanned (expected) |

## Evidence & Data

| Signal | Before (0.67.0 cask) | After (local 0.68.1) |
|---|---|---|
| `sample` on `cost-usage-scan` | 942/942 in day-key loop | No saturated loop |
| CPU | ~100% sustained | ~1.5% steady |
| `turnIDsByDay` | 739,526 empty entries | 0 |
| Priority payload | 15,777,708 bytes b64 | 1,132 bytes b64 |
| `usage_rows` | 10,085 | 10,085 (after rescan) |
| `parser_hash` | `86c0a89614a11da9` | `33d3202ea786d9ce` |

**Tests:** `make check` pass; focused cost suites pass; full suite 129/130
(sole failure: `MenuBarLayoutRendererTests` perf gate, pre-existing).

**Key files touched (committed on branch):**
- `Sources/CodexBarCore/Vendored/CostUsage/CostUsageStore.swift`
- `Tests/CodexBarTests/CostUsageScannerCodexPriorityAllHistoryTests.swift` (new)
- `Tests/CodexBarTests/CostUsageCoverageCompatibilityTests.swift`
- `Tests/CodexBarTests/CostUsageStoreTests.swift`

## Operator Feedback

- "After we finish all the fixes and test, remember to update issues and PRs
  and comments" — done (#4045 body, #4053/#3882 comments, ClawSweeper re-review).
- "Do all of that" — full test + relaunch completed.
- "That all the loose ends?" / "How long until 2 lands" — clarified: our work
  done; merge timing is maintainer-dependent; merge conflicts present.

## Where We're Going

1. **Resolve PR merge conflicts with `main`** on `fix-infinite-loop-daykeys` and
   `fix-claude-artifact-redecode`, then push and ping maintainer for merge.
2. Optionally add 0.67.0 predecessor hash if zero-downtime upgrade from cask matters.
3. Optional: `#4053` narrower-window/pricing retention test if ClawSweeper still nits.
4. After upstream merge + release, upgrade `/Applications/CodexBar.app` via cask.

## Quick Start

```bash
cd /Users/djbclark/src/CodexBar
git fetch djbclark origin
git checkout fix-claude-artifact-redecode   # or fix-infinite-loop-daykeys for #4045 only
git log --oneline -3

# Verify fixed app still running
pgrep -fl 'CodexBar.app/Contents/MacOS/CodexBar'
ps -p $(pgrep -f 'CodexBar.app/Contents/MacOS/CodexBar' | head -1) -o etime,pcpu

# Check live cache compaction
python3 - <<'PY'
import sqlite3, os, json, base64
db=os.path.expanduser('~/Library/Caches/CodexBar/cost-usage/cost-usage.sqlite')
c=sqlite3.connect(f'file:{db}?mode=ro', uri=True)
meta=dict(c.execute('select key,value from meta'))
rows=c.execute('select count(*) from usage_rows').fetchone()[0]
sm=c.execute('select payload from scan_metadata where id=1').fetchone()
obj=json.loads(sm[0]) if sm and sm[0] else {}
p=obj.get('priorityTurnStatePayload','')
print('parser_hash', meta.get('parser_hash'), 'rows', rows, 'payload_b64', len(p))
c.close()
PY

# Rebase/merge main to clear PR conflicts (fetch steipete first)
git fetch origin main
git merge origin/main   # resolve conflicts, then push djbclark

# Re-run tests if needed
make check
swift test --filter 'CostUsageScannerCodexPriorityAllHistory|CostUsageStoreTests'

# Rebuild/relaunch local fix
./Scripts/compile_and_run.sh
```
