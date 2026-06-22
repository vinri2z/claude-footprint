# claude-footprint

[![GitHub stars](https://img.shields.io/github/stars/vinri2z/claude-footprint)](https://github.com/vinri2z/claude-footprint/stargazers)
[![License: MIT](https://img.shields.io/badge/license-MIT-green)](LICENSE)
[![GitHub release](https://img.shields.io/github/v/release/vinri2z/claude-footprint)](https://github.com/vinri2z/claude-footprint/releases)

Track the carbon **and water** footprint of your Claude Code sessions.

**1. Install (or update):**

```bash
curl -fsSL https://raw.githubusercontent.com/vinri2z/claude-footprint/main/install.sh | bash
```

Same command to install and to update to the latest version.

**2. Restart Claude Code.** Your CO2 and water appear in the status line:

```
claude-footprint ⌥ main | 🟢 Opus 4.7 ▓▓▓░░░░░░░ 35% | $0.50 · 65g CO₂ · 💧 0.8L | Use 24% ↻13:00
```

Segments, left to right: project + git branch, model + context window %, session cost + CO2 + water, 5h block usage % + reset time. A 🔥 prefix appears when the sustained burn rate would overshoot 100% of the limit by the end of the 5h block (after a 15 min grace window, only once usage reaches 15%).

**5h quota source.** The percentage comes directly from Anthropic's `/api/oauth/usage` endpoint (the same data Claude Code displays in `/usage`). No heuristic, no token-limit file to seed. Two sources in order:

1. **stdin** (preferred): if Claude Code injects `rate_limits.five_hour.used_percentage` in the statusline JSON, that value is used straight away.
2. **OAuth API fallback**: `GET https://api.anthropic.com/api/oauth/usage` with the bearer token from macOS Keychain, `CLAUDE_CODE_OAUTH_TOKEN`, or `~/.claude/.credentials.json`. Cached 60s in `~/.claude/claude-carbon/oauth-usage.json`.

Accurate on every plan, including Max 20x.

**3. Use the slash commands:**

- `/footprint-report` - text report with totals, equivalences, top sessions
- `/footprint-card` - generate shareable PNG report cards (requires `playwright-core`, see [Dependencies](#dependencies))

## What it does

- Adds a live CO2 and water estimate to the Claude Code status line, next to the session cost
- Persists each session to a local SQLite database
- Backfills historical data from existing `~/.claude` transcripts
- Two slash commands: `/footprint-report` (text) and `/footprint-card` (PNG)

## Example report

<p align="center">
  <img src="docs/example-report-v2.png" alt="Claude Carbon Report" width="540">
</p>

Generate yours with `/footprint-card` in Claude Code. Exports summary and detailed PNGs to `exports/`.

<details>
<summary>Advanced options (CLI)</summary>

```bash
# Since a specific date
bash ~/code/claude-footprint/scripts/generate-report.sh --since 2026-03-01

# All time
bash ~/code/claude-footprint/scripts/generate-report.sh --all
```

</details>

<details>
<summary>Custom install directory</summary>

```bash
CLAUDE_FOOTPRINT_DIR=~/my-path/claude-footprint curl -fsSL https://raw.githubusercontent.com/vinri2z/claude-footprint/main/install.sh | bash
```

</details>

<details>
<summary>Manual install</summary>

```bash
git clone https://github.com/vinri2z/claude-footprint.git ~/code/claude-footprint
bash ~/code/claude-footprint/scripts/setup.sh
```

Then add to `~/.claude/settings.json`:

```json
{
  "statusLine": {
    "type": "command",
    "command": "~/code/claude-footprint/scripts/statusline.sh"
  },
  "hooks": {
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "~/code/claude-footprint/scripts/persist-session.sh"
          }
        ]
      }
    ]
  }
}
```

Restart Claude Code.

</details>

## How it works

![Data flow](docs/data-flow.png)

**Three data paths, two levels of accuracy:**

| Script               | Trigger                 | Data source           | Subagents    | Cache reads         | Accuracy      |
| -------------------- | ----------------------- | --------------------- | ------------ | ------------------- | ------------- |
| `backfill.sh`        | Manual / setup          | JSONL files           | Included     | Counted (8% energy) | Best estimate |
| `persist-session.sh` | Stop hook (session end) | JSONL files           | Included     | Counted (8% energy) | Best estimate |
| `statusline.sh`      | Every turn (live)       | `context_window` JSON | Not included | Included (approx)   | Approximate   |

**backfill** and **persist-session** parse the raw JSONL transcripts (main session + subagent files), applying per-model emission factors. They deduplicate assistant messages by `(message.id, requestId)`, so resumed and compacted sessions are not double-counted (this matches `ccusage`; without it the token sum inflates roughly 3x). Each session stores its raw token breakdown (input, cache write, cache read, output), which feeds the SQLite database used by reports.

**Cost** is the theoretical API list value (pay-as-you-go), not your subscription price: input, output, cache write (1.25x input), and cache read (0.1x input) at current Anthropic rates, set in `data/prices.json`. On deduplicated data it matches `ccusage`.

**statusline** reads `context_window.total_input_tokens` from Claude Code at each turn. This value represents the current context size (not a cumulative total), includes cache reads, and does not account for subagent tokens. It's an indicative live display, not a data source for reports.

### Surviving the 30-day transcript purge

Claude Code deletes JSONL transcripts after about 30 days, so the SQLite database is the durable record. The `Stop` hook captures each session before its transcript ages out, and a once-a-day background re-scan (`SessionStart` hook, `safety-rescan.sh`) catches any session the `Stop` hook missed while its transcript still exists. Because each row stores raw token counts, `recompute.sh` regenerates cost and CO2 from `data/factors.json` + `data/prices.json` at any time, with no transcript needed. When Anthropic changes a price or a factor is revised, edit the config and run:

```bash
bash scripts/recompute.sh
```

## Commands

| Command          | What it does                                        |
| ---------------- | --------------------------------------------------- |
| `/footprint-report` | Text report with CO2 + water totals, equivalences, top sessions |
| `/footprint-card`   | Generate shareable PNG report cards (CO2 + water)   |

<details>
<summary>Scripts (run automatically, rarely needed manually)</summary>

| Script               | What it does                                                                              |
| -------------------- | ----------------------------------------------------------------------------------------- |
| `setup.sh`           | Init database, backfill historical sessions, show total                                   |
| `statusline.sh`      | Status line script (called automatically by Claude Code)                                  |
| `persist-session.sh` | Stop hook (saves session data on exit)                                                    |
| `safety-rescan.sh`   | SessionStart hook (throttled background re-scan, catches missed sessions)                 |
| `backfill.sh`        | Re-parse all historical JSONL transcripts (incl. subagents)                               |
| `recompute.sh`       | Re-derive cost/CO2/water from stored tokens after a price/factor change (no transcripts needed) |
| `generate-report.sh` | Export PNG report cards (CLI, with `--since` / `--all`)                                   |

Note: backfill now derives project names from the transcript's `cwd` (matching the live hook). Sessions backfilled before this change keep their old, possibly truncated names; delete those rows and re-run `backfill.sh` to normalize them.

</details>

## Emission factors

Factors from [Jegham et al. 2025](https://arxiv.org/abs/2505.09598), a peer-reviewed study measuring energy consumption of LLM inference on AWS infrastructure.

| Model  | Input (gCO2e/Mtok) | Output (gCO2e/Mtok) | Basis                      |
| ------ | ------------------ | ------------------- | -------------------------- |
| Fable  | 1000               | 6000                | Extrapolated (2x Opus)     |
| Opus   | 500                | 3000                | Extrapolated (3x Sonnet)   |
| Sonnet | 190                | 1140                | Measured                   |
| Haiku  | 95                 | 570                 | Extrapolated (0.5x Sonnet) |

**Important: these are order-of-magnitude estimates, not precise measurements.**

- Sonnet factors are derived from Jegham et al. direct measurements. Fable, Opus and Haiku are extrapolated (no public data from Anthropic on per-model energy consumption).
- Sessions run on non-Anthropic models (e.g. local models behind `ANTHROPIC_BASE_URL`) are stored with their raw tokens but zero cost/CO2 and excluded from reports - a datacenter factor doesn't apply to them. Add patterns to `exclude_models` in `data/factors.json` to exclude more models by name.
- Cache read tokens are counted at a reduced factor (default 0.08 of an input token, set in `data/factors.json`). A cached token skips prefill compute but still incurs decode-phase memory reads, so it is cheap but not free. This is an engineering estimate derived from the literature, not Anthropic's 0.1x billing ratio. See [METHODOLOGY.md](METHODOLOGY.md).
- Carbon intensity uses AWS grid-average (0.287 kgCO2e/kWh), not real-time grid data.
- Anthropic does not publish Scope 1, 2, or 3 emissions. These estimates are independent and based on academic research, not provider data.

Factors are editable in `data/factors.json`. See [METHODOLOGY.md](METHODOLOGY.md) for the full scientific basis, formula, and equivalences.

### Water factors

Water is derived from the same inference energy as CO2, using a water-intensity factor (`WIF = onsite WUE 0.18 + offsite EWIF 3.14 = 3.32 L/kWh`) in place of the carbon intensity (`CIF = 0.287 kgCO2e/kWh`). Per-model water factor = `co2_factor × 3.32 / 287 ≈ co2_factor × 0.0115679 L/gCO2e`.

| Model  | Input (L/Mtok) | Output (L/Mtok) |
| ------ | -------------- | --------------- |
| Fable  | 11.568         | 69.408          |
| Opus   | 5.784          | 34.704          |
| Sonnet | 2.198          | 13.187          |
| Haiku  | 1.099          | 6.594           |

This is a **deliberately conservative (over-estimated)** figure: the offsite term uses the US-grid average water intensity, not the more efficient AWS-region mix. Measured in liters, reported next to CO2 everywhere. Water factors are editable in `data/factors.json`. See [METHODOLOGY.md](METHODOLOGY.md) for sources (AWS 2024 WUE; Li et al. 2023, arXiv:2304.03271; Reig/WRI EWIF; EESI).

### Golden vectors

The methodology is pinned by golden test vectors in [`tests/methodology-vectors.json`](tests/methodology-vectors.json): hand-computed expected CO2/cost/water values for known token breakdowns, replayed by `bash tests/run-vectors.sh` in CI on every push. Downstream consumers (such as TokenClimate) keep a copy of this file and verify weekly that their implementation produces the same numbers. If you edit `data/factors.json` or `data/prices.json`, update the vectors in the same commit, otherwise CI fails.

## Dependencies

- `jq` - JSON parsing
- `sqlite3` - local database
- `git` - branch detection in status line (optional)
- `curl` - 5h quota usage via Anthropic's `/api/oauth/usage` endpoint (optional, 60s cache)
- `playwright-core` + Chromium - PNG export for `/footprint-card` (optional)

`jq` and `sqlite3` are pre-installed on macOS. On Linux: `apt install jq sqlite3`.

To use `/footprint-card`, install Playwright and its Chromium browser:

```bash
npm install -g playwright-core
npx playwright install chromium
```

## Reduce your footprint

Measuring is step one. Here are concrete levers to reduce your AI carbon footprint, ranked by impact.

### Use the right model for the task

Output tokens cost 5x more energy than input tokens. Opus consumes ~3x more than Sonnet per token.

```json
{
  "env": {
    "CLAUDE_CODE_SUBAGENT_MODEL": "claude-haiku-4-5"
  }
}
```

Use Opus for architecture and planning. Sonnet for daily work. Haiku for subagents (exploration, file reading, reviews). This alone can cut your emissions by 60%.

### Install RTK (Rust Token Killer)

[RTK](https://github.com/rtk-ai/rtk) is a CLI proxy that filters noise from shell outputs (progress bars, verbose logs, passing tests) before they hit the context window. 60-90% token reduction on CLI commands, zero quality loss.

```bash
brew install rtk-ai/tap/rtk
rtk init -g
```

### Reduce thinking tokens

Claude's extended thinking can use up to 32k hidden tokens per message. Capping it reduces consumption without degrading quality on routine tasks.

```json
{
  "env": {
    "MAX_THINKING_TOKENS": "10000"
  }
}
```

### Compact earlier

By default, Claude Code compacts context at 95% usage. Compacting earlier keeps context cleaner and avoids bloated sessions.

```json
{
  "env": {
    "CLAUDE_AUTOCOMPACT_PCT_OVERRIDE": "50"
  }
}
```

### Write concise instructions

Add to your project's CLAUDE.md:

```
Be concise. No preamble, no summaries unless asked.
```

Output tokens are the most expensive in both cost and energy.

### Combined impact

| Lever                | Estimated reduction     |
| -------------------- | ----------------------- |
| Right model per task | -60% vs all-Opus        |
| RTK                  | -70% on CLI tokens      |
| Thinking cap at 10k  | -70% on thinking tokens |
| Haiku subagents      | -80% on exploration     |
| **All combined**     | **-50 to 70% total**    |

### Further reading

- [IEA - Energy and AI (2025)](https://www.iea.org/reports/energy-and-ai/) - data center projections
- [Jegham et al. - How Hungry is AI?](https://arxiv.org/abs/2505.09598) - per-model energy measurements
- [UCL/UNESCO - 90% AI energy reduction](https://www.ucl.ac.uk/news/2025/jul/practical-changes-could-reduce-ai-energy-demand-90) - frugal AI approaches
- [GreenIT.fr - AI impacts 2025-2030](https://www.greenit.fr/impacts-ia-monde-2025-2030-rapport/) - French data

## Why

Every Claude Code session uses real compute, real energy, real emissions. The number is small per query, but it adds up. Making it visible is the first step to owning it.

## Open source

claude-footprint is free and open source under the [MIT license](LICENSE). Contributions welcome.

Built by [Vincent Rizzo](https://github.com/vinri2z).
