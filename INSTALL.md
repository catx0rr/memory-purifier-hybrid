# INSTALL.md — First-Time Initialization

**`install.sh` installs the package and seeds control-plane JSONs. It does not run a purifier pass. Follow this guide to complete first live initialization.**

---

## 0. Prerequisites

- OpenClaw installed, with `openclaw` CLI on `PATH`.
- `python3` available (no third-party packages required).
- `jq` recommended for inspection.
- Upstream consolidators active: `reflections-hybrid` and OpenClaw native dreaming.
- Timezone set to **Asia/Manila** (or accept deviation if overriding `timezone` in config).

---

## 1. Run the installer

```bash
# Clone-and-install (recommended; omitted --agent-profile defaults to personal)
curl -fsSL https://raw.githubusercontent.com/catx0rr/memory-purifier/main/install.sh | bash

# Or specify explicitly
curl -fsSL https://raw.githubusercontent.com/catx0rr/memory-purifier/main/install.sh | bash -s -- --agent-profile business

# Or from a local checkout
cd <path-to-package>
bash install.sh                          # uses default profile: personal
bash install.sh --agent-profile business
```

### CLI flags

| Flag | Effect |
|---|---|
| `--agent-profile business\|personal` | Profile for seeded config and cron. **Default when omitted: `personal`.** Invalid values fail with usage help. |
| `--local` | Install from the directory containing `install.sh`. Skips `git clone/pull`; use for offline installs and the test harness. |
| `--cron-tz <IANA>` | IANA timezone for cron registration. **Default: `Asia/Manila`.** Minimum-shape validation rejects obvious typos before hitting `openclaw cron add`. |
| `--cron-announce true\|false` | `true` registers cron **without** `--no-deliver` and seeds `memoryPurifier.reporting.enabled = true`; `false` registers **with** `--no-deliver` and seeds `enabled = false`. **Default: `false`.** This is the install-time source of truth for chat-delivery eligibility. |
| `--timeout-seconds <int>` | Positive-integer per-run timeout for cron registration. **Default: `1200`.** |
| `--skip-cron` | Do not register cron jobs |
| `--force-config` | Overwrite existing `memory-purifier.json`. Also reseeds `memoryPurifier.reporting.enabled` from `--cron-announce` when the namespace is already present. |
| `--help` | Print usage |

### Path overrides (env vars)

| Env var | Default |
|---|---|
| `CONFIG_ROOT` | `$HOME/.openclaw` |
| `WORKSPACE` | `$HOME/.openclaw/workspace` |
| `SKILLS_PATH` | `$HOME/.openclaw/workspace/skills` |
| `TELEMETRY_ROOT` | `$HOME/.openclaw/telemetry/memory-purifier` |

### What `install.sh` does

1. Clones/updates `$SKILL_ROOT` (or uses an in-place checkout when `--local`).
2. Creates `$WORKSPACE/runtime/` and `locks/`.
3. Creates `$CONFIG_ROOT/memory-purifier/` and `$TELEMETRY_ROOT/`.
4. Seeds `memory-purifier.json` with profile-appropriate cadence, plus a `cron` block carrying `tz`, `timeout_seconds`, and `announce` for the delivery-sync helper.
5. Seeds control-plane JSONs: `purifier-metadata.json`, `purified-manifest.json`, `purifier-last-run-summary.json`.
6. Idempotently merges `memoryPurifier` namespace into `$WORKSPACE/runtime/memory-state.json` — seeding `reporting.enabled` from `--cron-announce`.
7. Registers cron jobs (business: 2 jobs; personal: 3 jobs) with a short **launcher message** (not the prompt body) pointing at the correct prompt file. The prompt runs `scripts/run_purifier.py`. Reconciliation owns Wed + Sun via day-of-week exclusion on the incremental expressions. Each job is registered with `--tz "$CRON_TZ"`, `--timeout-seconds "$TIMEOUT_SECONDS"`, and either `--no-deliver` or the announce form depending on `--cron-announce`.
8. Verifies all seeded files parse; prints next-steps pointing to this document.

It does **not** create `purified-claims.jsonl`, contradictions, entities, routes, or any markdown views. Those are produced by the first successful live run (step 5 below).

### Cron launcher message

Registration passes a short launcher — not the prompt prose — so cron metadata stays small and prompts can evolve without re-registering:

```
Run memory purifier.

Read `<abs-path>/prompts/incremental-purifier-prompt.md` and follow every step strictly.
```

(The reconciliation job points at `reconciliation-purifier-prompt.md`.) All operational detail lives in the prompt file itself.

### Cron delivery ↔ reporting.enabled sync

`--cron-announce` seeds both settings at install time. They can drift later if the operator flips `memoryPurifier.reporting.enabled` in `memory-state.json` without updating cron. The cron supervisor prompts call `scripts/sync_cron_delivery.py` before chat delivery to reconcile drift — the helper re-registers mismatched jobs with the correct flag so the **next** fire obeys `reporting.enabled`. This is the single deterministic actor that mutates cron delivery after install. The current run's delivery is whatever the cron scheduler already decided when it fired.

---

## 2. Verify the seeded runtime

```bash
CONFIG_ROOT="${CONFIG_ROOT:-$HOME/.openclaw}"
WORKSPACE="${WORKSPACE:-$HOME/.openclaw/workspace}"
SKILL_ROOT="$WORKSPACE/skills/memory-purifier"
RUNTIME_DIR="$WORKSPACE/runtime"

ls -la "$RUNTIME_DIR/"
ls -la "$RUNTIME_DIR/locks/"
cat "$CONFIG_ROOT/memory-purifier/memory-purifier.json" | jq '.profile, .cadence'
python3 -c "import json; print('ok' if 'memoryPurifier' in json.load(open('$WORKSPACE/runtime/memory-state.json')) else 'missing')"
```

Expected: seeded JSONs present, config parses, `memoryPurifier` namespace merged into shared state. Live artifacts (`purified-claims.jsonl`, etc.) are **not yet expected** to exist.

---

## 3. Dry-run the pipeline

A dry-run exercises every step without writing any artifacts, manifest, telemetry, or cursor. It's the safest way to confirm the pipeline chains correctly before first live execution.

```bash
python3 "$SKILL_ROOT/scripts/run_purifier.py" --mode incremental --dry-run | jq '.status, .steps'
```

Expected: `"status": "ok"` with each step (`discover`, `scope`, `extract`, `pass1`, `cluster`, `pass2`, `assemble`, `render`) reporting `status: ok` or `skipped`. If any step reports `error` or `partial_failure`, stop and inspect — do not proceed.

---

## 4. (Optional) Test with a disposable workspace first

If you'd rather not run against your live workspace on first execution, stage a throwaway:

```bash
export TEST_WS="/tmp/mp-first-run"
mkdir -p "$TEST_WS/episodes"
echo "Operator prefers terse responses." > "$TEST_WS/MEMORY.md"
echo "# RTMEMORY" > "$TEST_WS/RTMEMORY.md"
echo "# PROCEDURES" > "$TEST_WS/PROCEDURES.md"

python3 "$SKILL_ROOT/scripts/run_purifier.py" \
    --mode incremental \
    --workspace "$TEST_WS" \
    --profile business \
    --dry-run | jq '.status'
```

When satisfied, rerun against the real workspace.

---

## 5. First live incremental run

```bash
python3 "$SKILL_ROOT/scripts/run_purifier.py" --mode incremental
```

This reads your live consolidated substrate, runs both LLM passes, writes the artifacts, renders the markdown views, writes the manifest, runs validation, and signals downstream. On success, the orchestrator prints a summary JSON with `"status": "ok"`.

---

## 6. Verify live artifacts

```bash
ls -la "$RUNTIME_DIR/"
# Expect:
#   purified-claims.jsonl
#   purified-contradictions.jsonl (may be empty)
#   purified-entities.json
#   purified-routes.json
#   purified-manifest.json
#   purifier-last-run-summary.json
#   (deferred-candidates.jsonl, rejected-candidates.jsonl if any)

cat "$RUNTIME_DIR/purified-manifest.json" | jq '.status, .runId, .lastSuccessfulCursor, .downstreamWikiIngestSuggested'
```

Expected: `status: "ok"`, a populated `lastSuccessfulCursor`, `downstreamWikiIngestSuggested: true`.

---

## 7. Verify purified markdown views

```bash
head -n 5 "$WORKSPACE/LTMEMORY.md"
head -n 5 "$WORKSPACE/PLAYBOOKS.md"
head -n 5 "$WORKSPACE/EPISODES.md"
# Personal profile only:
#   head -n 5 "$WORKSPACE/HISTORY.md"
#   head -n 5 "$WORKSPACE/WISHES.md"
```

Expected: each view begins with the header + regeneration line. Claim blocks appear underneath if any active claims were routed to that home.

---

## 8. Confirm cron is registered (or register manually)

```bash
openclaw cron list --json | jq '.[] | select(.name | startswith("memory-purifier"))'
```

Expected entries — business profile: `memory-purifier-incremental`, `memory-purifier-reconciliation`. Personal profile: adds `memory-purifier-incremental-evening`. Each has `cron`, `tz: "Asia/Manila"`, `session: "isolated"`, `deliver: false`, and a `message` containing the absolute path to the appropriate prompt file.

If `install.sh` was run with `--skip-cron`, register manually. **Pass a short launcher message** that points at the prompt file — never inline the prompt body:

```bash
INCR_PROMPT="$SKILL_ROOT/prompts/incremental-purifier-prompt.md"
RECON_PROMPT="$SKILL_ROOT/prompts/reconciliation-purifier-prompt.md"
INCR_LAUNCHER="Run memory purifier.

Read \`$INCR_PROMPT\` and follow every step strictly."
RECON_LAUNCHER="Run memory purifier.

Read \`$RECON_PROMPT\` and follow every step strictly."

# Business profile (announce=false → --no-deliver; omit --no-deliver to announce)
openclaw cron add \
    --name "memory-purifier-incremental" \
    --cron "15 13 * * 1,2,4,5,6" \
    --tz "Asia/Manila" --session isolated --timeout-seconds 1200 --no-deliver \
    --message "$INCR_LAUNCHER"

openclaw cron add \
    --name "memory-purifier-reconciliation" \
    --cron "15 13 * * 3,0" \
    --tz "Asia/Manila" --session isolated --timeout-seconds 1200 --no-deliver \
    --message "$RECON_LAUNCHER"

# Personal profile (three entries; same --tz, --timeout-seconds, --no-deliver rules)
#   memory-purifier-incremental-morning:    15 5 * * 1,2,4,5,6 → $INCR_LAUNCHER
#   memory-purifier-incremental-evening:    15 17 * * *       → $INCR_LAUNCHER
#   memory-purifier-reconciliation:         15 5 * * 3,0      → $RECON_LAUNCHER
```

Incremental expressions exclude `3,0` (Wed + Sun) so reconciliation owns its slot on those days without collision — see [`references/cadence-profiles.md`](references/cadence-profiles.md) §4.

---

## 9. (Optional) Run reconciliation once

Before waiting for the Wed/Sun slot, you can prove reconciliation mode works against live artifacts:

```bash
python3 "$SKILL_ROOT/scripts/run_purifier.py" --mode reconciliation
```

Expected: full-horizon re-read; existing claims re-evaluated against `prior_claims_context`; any supersession / contradiction state updated.

---

## Troubleshooting

| Symptom | Check |
|---|---|
| `install.sh` fails at cron step | `openclaw` on PATH? Try `--skip-cron` then register manually (step 8). |
| Dry-run returns `status: skipped` at scope | `$WORKSPACE/MEMORY.md` or `RTMEMORY.md` present with content? |
| Personal views not emitting on personal profile | `personal_surfaces.enabled: true` in config? `CHRONICLES.md` / `DREAMS.md` present? |
| Duplicate claims on reruns | `lastSuccessfulCursor` populated in manifest? Inspect `assemble_artifacts.py` output. Claim ids are content-hashed — reruns with unchanged inputs must not produce duplicates. |
| Cron registration refuses | Existing entries with same name? `openclaw cron delete --name memory-purifier-incremental` then retry. |

Deeper issues: inspect

- `$RUNTIME_DIR/purified-manifest.json` — `warnings[]`, `partialFailures[]` for the latest run
- `$RUNTIME_DIR/locks/failed-*.json` — raw LLM responses on Pass 1 / Pass 2 validation failure
- `$HOME/.openclaw/telemetry/memory-log-YYYY-MM-DD.jsonl` — **shared memory-log** (all memory plugins); filter purifier events with `jq 'select(.component == "memory-purifier.purifier")'`
- `$TELEMETRY_ROOT/last-run.md` — deterministic human-readable snapshot of the last run

**Telemetry shape (in the shared memory-log):** every event carries `domain: "memory"`, `component: "memory-purifier.purifier"`, `event ∈ {run_started, run_completed, run_skipped, run_failed}`, plus a `token_usage` block (`prompt_tokens`, `completion_tokens`, `total_tokens`, `source`). Token usage counts only Pass 1 + Pass 2 LLM calls — never deterministic script work — with `source: "exact"` when the provider returns usage metadata, `"approximate"` when computed from actual prompt/completion char counts, `"unavailable"` when no real model was invoked (e.g. fixture tests).

**Chat reporting** is configured in `<workspace>/runtime/memory-state.json` under `memoryPurifier.reporting`:

- `enabled: false` (default; seeded from `--cron-announce false`) — silent in chat; telemetry and latest-report are still written
- `enabled: true` (seeded from `--cron-announce true`) + `mode: "silent" | "summary" | "full"` — `summary` is a one-liner, `full` is a bullet-list report with a `🪙 Token Usage` block

Cron delivery mode (`--no-deliver` vs announce) must match `reporting.enabled` for chat output to actually reach the operator. `scripts/sync_cron_delivery.py` is the single deterministic actor that reconciles drift; the cron supervisor prompts call it each fire so the next run is aligned.

---

## Upgrading

```bash
cd "$SKILL_ROOT"
git pull --ff-only
bash install.sh  # re-runs with the previously seeded profile; the default is personal
```

`install.sh` is idempotent. Existing config, manifest, and cron entries are preserved. Forced config overwrite requires `--force-config`.

## Uninstall

```bash
openclaw cron list --json | jq -r '.[] | select(.name | startswith("memory-purifier")) | .name' | \
    xargs -I{} openclaw cron delete --name {}
rm -rf "$WORKSPACE/runtime/" "$TELEMETRY_ROOT" "$CONFIG_ROOT/memory-purifier" "$SKILL_ROOT"
# Optional — prune the memoryPurifier namespace from shared state:
python3 -c "
import json
p = '$WORKSPACE/runtime/memory-state.json'
d = json.load(open(p)); d.pop('memoryPurifier', None)
json.dump(d, open(p, 'w'), indent=2)
"
```

Rendered markdown views (`LTMEMORY.md`, etc.) are left in place as human-readable snapshots. Remove manually if desired.
