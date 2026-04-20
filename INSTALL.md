# INSTALL.md — Guided First-Time Initialization

**This is the agent-facing initialization flow. `install.sh` seeds the package; this guide runs the first live passes with gated one-line questions. An LLM or operator follows these steps top-to-bottom, pausing at each `❓` prompt for a yes/no or short answer.**

---

## 0. Prerequisites

- OpenClaw installed, `openclaw` CLI on `PATH`.
- `python3` available (no third-party packages required).
- `jq` recommended for inspection.
- Upstream consolidators active: [`reflections-hybrid`](https://github.com/catx0rr/reflections-hybrid) and OpenClaw native dreaming.

> ❓ **Do you want me to install memory-purifier now, or read what it is for first and give insights?**
>
> - **Install now** → continue to §1.
> - **Explain first** → summarize [`README.md`](README.md) (post-consolidation, non-destructive canonicalization; two-pass LLM; reads `MEMORY.md`/`RTMEMORY.md`/`PROCEDURES.md`/`episodes/`; emits `LTMEMORY.md`/`PLAYBOOKS.md`/`EPISODES.md`), then re-ask this question.

---

## 1. Gated install questions

Ask the operator these four short questions, one at a time, then pass the answers to `install.sh`.

> ❓ **Which agent profile do you want: `business` or `personal`?**  _(default: `personal`)_
>
> ❓ **Which IANA timezone should I use for cron configuration?**  _(default: `Asia/Manila`; examples: `America/Los_Angeles`, `Europe/Berlin`, `Etc/UTC`)_
>
> ❓ **Do you want cron announce enabled (reports appear in chat) or no-deliver (silent)?**  _(default: `no-deliver` = `false`)_
>
> ❓ **Do you want to use the default timeout of 1200 seconds, or specify a custom timeout?**  _(default: `1200`)_

Map the answers:

| Answer | Installer flag |
|---|---|
| profile | `--agent-profile <business\|personal>` |
| timezone | `--cron-tz <IANA>` |
| announce = enabled | `--cron-announce true` |
| announce = no-deliver | `--cron-announce false` |
| custom timeout | `--timeout-seconds <int>` |

---

## 2. Run the installer

```bash
# specify your skill root defaults to workspace/skills/ directory
SKILL_ROOT="$HOME/.openclaw/workspace/skills/memory-purifier"

# Remote install (uses the answers from §1; replace values accordingly)
curl -fsSL https://raw.githubusercontent.com/catx0rr/memory-purifier-hybrid/main/install.sh | \
  bash -s -- \
    --agent-profile <business|personal> \
    --cron-tz <IANA> \
    --cron-announce <true|false> \
    --timeout-seconds <int>

# Or from a local checkout
cd <path-to-package>
bash install.sh --agent-profile <business|personal> --cron-tz <IANA> --cron-announce <true|false> --timeout-seconds <int>
```

### CLI flag reference

| Flag | Default | Effect |
|---|---|---|
| `--agent-profile business\|personal` | `personal` | Seeded profile + cadence. |
| `--local` | off | Install from the directory containing `install.sh` (offline; no git). |
| `--cron-tz <IANA>` | `Asia/Manila` | Timezone for cron registration. Shape-validated before use. |
| `--cron-announce true\|false` | `false` | `true` = cron without `--no-deliver` + `reporting.enabled=true`; `false` = the inverse. |
| `--timeout-seconds <int>` | `1200` | Positive-int per-run timeout. |
| `--skip-cron` | off | Skip cron registration. |
| `--force-config` | off | Overwrite `memory-purifier.json`; also reseeds `reporting.enabled`. |

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
4. Seeds `memory-purifier.json` with profile cadence + `cron` block (`tz`, `timeout_seconds`, `announce`).
5. Seeds control-plane JSONs: `purifier-metadata.json`, `purified-manifest.json`, `purifier-last-run-summary.json`.
6. Merges `memoryPurifier` namespace into `$WORKSPACE/runtime/memory-state.json` — seeding `reporting.enabled` from `--cron-announce`.
7. Registers cron jobs with a short **launcher message** pointing at the correct prompt file. Reconciliation owns Wed + Sun via day-of-week exclusion.
8. Verifies all seeded files parse; prints next-step instructions pointing back here.

The installer does **not** create live artifacts — those come from the first live run (§5).

### Cron launcher message

Registration passes a short launcher — not the prompt body:

```
Run memory purifier.

Read `<abs-path>/prompts/incremental-purifier-prompt.md` and follow every step strictly.
```

### Cron delivery ↔ reporting.enabled sync

`--cron-announce` seeds both the `--no-deliver` flag and `reporting.enabled`. Drift later is reconciled by [`scripts/sync_cron_delivery.py`](scripts/sync_cron_delivery.py), called by the cron supervisor prompts each fire. That helper is the single deterministic actor mutating cron delivery after install.

---

## 3. Verify the seeded runtime

```bash
# specify your skill root defaults to workspace/skills/ directory
SKILL_ROOT="$HOME/.openclaw/workspace/skills/memory-purifier"
CONFIG_ROOT="${CONFIG_ROOT:-$HOME/.openclaw}"
WORKSPACE="${WORKSPACE:-$HOME/.openclaw/workspace}"
RUNTIME_DIR="$WORKSPACE/runtime"

ls -la "$RUNTIME_DIR/" "$RUNTIME_DIR/locks/"
jq '.profile, .cadence, .cron' "$CONFIG_ROOT/memory-purifier/memory-purifier.json"
python3 -c "import json; print('ok' if 'memoryPurifier' in json.load(open('$WORKSPACE/runtime/memory-state.json')) else 'missing')"
```

Expected: seeded JSONs present, config parses, `memoryPurifier` namespace merged. Live artifacts (`purified-claims.jsonl`, etc.) are **not yet expected**.

---

## 4. Reflections-aware test gate

Detect whether `$WORKSPACE/RTMEMORY.md` and `$WORKSPACE/PROCEDURES.md` both exist.

```bash
if [ -f "$WORKSPACE/RTMEMORY.md" ] && [ -f "$WORKSPACE/PROCEDURES.md" ]; then
    echo "both present"
else
    echo "one or both missing"
fi
```

### If both are present

> ❓ **I detected RTMEMORY.md and PROCEDURES.md. Do you want me to test memory-purifier in a disposable workspace first and show the results?**
>
> - **Yes** → run the disposable-workspace test below, report the summary JSON, then re-ask whether to continue with real initialization.
> - **No** → skip to §5.

```bash
# Disposable test (does not touch the live workspace)
export TEST_WS="/tmp/mp-first-run"
mkdir -p "$TEST_WS/episodes"
echo "Operator prefers terse responses." > "$TEST_WS/MEMORY.md"
echo "# RTMEMORY" > "$TEST_WS/RTMEMORY.md"
echo "# PROCEDURES" > "$TEST_WS/PROCEDURES.md"

python3 "$SKILL_ROOT/scripts/run_purifier.py" \
    --mode incremental --workspace "$TEST_WS" --profile business --dry-run | jq '.status, .steps'
```

Expected: `"status": "ok"` with each step `status: "ok"` or `"skipped"`. If anything reports `error` or `partial_failure`, stop and inspect.

### If one or both are missing

> **I did not detect RTMEMORY.md and PROCEDURES.md. Install [`reflections-hybrid`](https://github.com/catx0rr/reflections-hybrid) first if you want memory-purifier to complement it.**

Skip to §5 without the disposable-workspace test.

---

## 5. First live incremental initialization

```bash
python3 "$SKILL_ROOT/scripts/run_purifier.py" --mode incremental
```

Reads live consolidated substrate, runs both LLM passes, writes artifacts, renders markdown views, writes manifest, validates, signals downstream. On success the orchestrator prints a summary JSON with `"status": "ok"`.

---

## 6. Verify live artifacts

```bash
ls -la "$RUNTIME_DIR/"
# Expect:
#   purified-claims.jsonl, purified-contradictions.jsonl, purified-entities.json,
#   purified-routes.json, purified-manifest.json, purifier-last-run-summary.json
#   (deferred-candidates.jsonl, rejected-candidates.jsonl if any)

jq '.status, .runId, .lastSuccessfulCursor, .downstreamWikiIngestSuggested' "$RUNTIME_DIR/purified-manifest.json"
head -n 5 "$WORKSPACE/LTMEMORY.md" "$WORKSPACE/PLAYBOOKS.md" "$WORKSPACE/EPISODES.md"
# Personal profile also: "$WORKSPACE/HISTORY.md" "$WORKSPACE/WISHES.md"
```

Expected: `status: "ok"`, populated `lastSuccessfulCursor`, `downstreamWikiIngestSuggested: true`, each markdown view beginning with the header + regeneration line.

---

## 7. Confirm cron is registered

```bash
openclaw cron list --json | jq '.[] | select(.name | startswith("memory-purifier"))'
```

Expected — business profile: `memory-purifier-incremental`, `memory-purifier-reconciliation`. Personal profile: plus `memory-purifier-incremental-evening`. Each has `cron`, the chosen `tz`, `session: "isolated"`, the chosen `deliver` flag, `timeout_seconds`, and a short launcher message referencing the prompt file.

Manual registration (only needed if `--skip-cron` was used): see [`references/cadence-profiles.md §5`](references/cadence-profiles.md).

---

## 8. Optional first reconciliation

After §5 succeeds, the first reconciliation round can be invoked on demand rather than waiting for the Wed/Sun slot.

> ❓ **Do you want me to run the reconciliation once now?**
>
> - **Yes** → run the command below, then render the first-init report template from the final JSON.
> - **No** → stop here; installation is complete. The next reconciliation will still fire automatically on its cron schedule (business / personal both: `15 13 * * 3,0` or `15 5 * * 3,0` respectively). Notify the operator with a line like:
>
>   ```
>   🔄 Skipping the one-off reconciliation. The next scheduled {incremental|reconciliation} run will fire at {next cron execution schedule}.
>   ```
>
>   Fill the values deterministically from `openclaw cron list --json` (earliest upcoming fire across the `memory-purifier-*` jobs + its mode label).

```bash
python3 "$SKILL_ROOT/scripts/run_purifier.py" --mode reconciliation | tee /tmp/mp-recon-first-init.json | jq '.status'
```

Expected: `"status": "ok"` with full-horizon re-read and any supersession/contradiction state updated.

### First reconciliation initialization report (static template)

Emit this exactly once, after the first successful reconciliation. Fill values **only from the final JSON** (or `$WORKSPACE/runtime/purifier-last-run-summary.json` as deterministic fallback). Omit the entire `🪙 Token Usage` block when `tokenUsage.source == "unavailable"`.

```
⚗️ First Memory purifier reconciliation initialized!
• Claims: {claimsNew} new · {claimsTotal} total
• Supersessions: {supersessionCount}
• Contradictions: {contradictionCount}
• Wiki ingest suggested: {downstreamWikiIngestSuggested}
🪙 Token Usage
• Prompt: {tokenUsage.prompt_tokens}
• Completion: {tokenUsage.completion_tokens}
• Total: {tokenUsage.total_tokens} ({tokenUsage.source})

🔄 Next memory purifier schedule:
• {next cron execution schedule}
• {type of cron that will execute next}
```

Resolve the next schedule deterministically from `openclaw cron list --json`, picking the earliest upcoming fire across the `memory-purifier-*` jobs and labeling its mode (`incremental` vs `reconciliation`). Do not improvise alternative wording.

---

## Troubleshooting

| Symptom | Check |
|---|---|
| `install.sh` fails at cron step | `openclaw` on PATH? Try `--skip-cron` then register manually per [`references/cadence-profiles.md §5`](references/cadence-profiles.md). |
| Dry-run returns `status: skipped` at scope | `$WORKSPACE/MEMORY.md` or `RTMEMORY.md` present with content? |
| Personal views not emitting on personal profile | `personal_surfaces.enabled: true` in config? `CHRONICLES.md` / `DREAMS.md` present? |
| Duplicate claims on reruns | `lastSuccessfulCursor` populated in manifest? Claim ids are content-hashed — reruns with unchanged inputs must not produce duplicates. |
| Cron registration refuses | Existing entry with same name? `openclaw cron delete --name memory-purifier-incremental` then retry. |
| Chat delivery toggled but nothing posts | Cron `deliver` flag still `--no-deliver`? The next cron fire runs `sync_cron_delivery.py` and reconciles — the fire after that should deliver. |

Deeper issues: inspect

- `$RUNTIME_DIR/purified-manifest.json` — `warnings[]`, `partialFailures[]` for the latest run
- `$RUNTIME_DIR/locks/failed-*.json` — raw LLM responses on Pass 1 / Pass 2 validation failure
- `$HOME/.openclaw/telemetry/memory-log-YYYY-MM-DD.jsonl` — shared memory-log; filter with `jq 'select(.component == "memory-purifier.purifier")'`
- `$TELEMETRY_ROOT/last-run.md` — deterministic human-readable snapshot of the last run

**Telemetry shape:** every event carries `domain: "memory"`, `component: "memory-purifier.purifier"`, `event ∈ {run_started, run_completed, run_skipped, run_failed}`, plus a `token_usage` block. Token usage counts only Pass 1 + Pass 2 LLM calls — `exact` when the provider returns usage metadata, `approximate` when computed from char counts, `unavailable` when no real model was invoked (fixture runs, `run_started` events).

**Chat reporting** lives in `<workspace>/runtime/memory-state.json` under `memoryPurifier.reporting`:

- `enabled: false` (seeded from `--cron-announce false`) — silent in chat; telemetry + latest-report still written
- `enabled: true` (seeded from `--cron-announce true`) + `mode: "silent" | "summary" | "full"`

---

## Upgrading

```bash
cd "$SKILL_ROOT"
git pull --ff-only
bash install.sh  # idempotent; preserves existing config unless --force-config is passed
```

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
