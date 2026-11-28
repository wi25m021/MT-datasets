> Data dictionary and collection protocol, copied from the prototype (app) repository.
> Paths such as `app/`, `tools/` and `data/` refer to that repository. "H2 test" below = the
> thermal comparison across days (Sub-RQ1/Sub-RQ2), not hypothesis H2 of the proposal.

# DATA — telemetry schema, collection protocol, device spec, pilot data

This file is the thesis' **data-availability evidence**: what is logged, how it is
collected, on which device, and where the committed pilot data lives. Research design:
`.claude/CLAUDE.md`. Engineering notes: `CLAUDE.md`.

Schema version: **v1** (2026-10-15). Prompt suite: `cuad_pilot_v1`.

## 0. Status at a glance

| | |
|---|---|
| Device | Pixel 10, Tensor G5, Android 16 |
| Backends measured | `aicore` (Gemini Nano, nano-v4-full), `litert` (gemma-4-E2B-it, CPU) |
| Prompt suite | `cuad_pilot_v1`, 15 prompts, 5 size levels x 3 task types |
| Sessions committed | 3 (1 sustained pilot, 2 ramp) |
| Measured runs committed | 541 |
| Of which with valid energy | 344 |
| Thermal range reached | `thermal_status` 0-1, headroom 0.43-0.85 |
| Target for the thesis | about 1000 measured runs over 3 unplugged ramp sessions, plus 30 cloud runs |

The measurement data does not exist publicly and is produced by the prototype. The sessions
listed in section 7 are the evidence that collection works end to end on the target device.


## 1. Files

On the phone (pull with
`adb pull /sdcard/Android/data/com.gladun.aicore/files/telemetry ./data/raw/`):

```
telemetry/
  sessions/<session_id>/          session_id = yyyyMMdd-HHmmss-<preset>
    session.json                  device spec, versions, protocol parameters, conditions, status
    runs.csv                      one row per run (schema below), appended + fsync'd per run
    outputs.csv                   collect mode: run_id, prompt_id, backend, repetition, expected_answer, output_text
    thermal.csv                   1 Hz device-state trace for the whole session
    traces/<block_id>.csv         100 ms energy/memory trace, continuous over the block
  diagnostics/diag-<timestamp>.json   Stage-0 / verify-on-device probe results
```

In the repo: committed pilot sessions live in `data/pilot/<session_id>/` (same layout).

CSV conventions: RFC 4180 quoting, `.` decimal separator (Locale.ROOT), empty field = null,
booleans as `0`/`1`, timestamps ISO-8601 UTC with milliseconds. No prompt text is ever written;
response text only in collect-mode `outputs.csv` (public CUAD prompts).

## 2. `runs.csv` — data dictionary

Role: **F** = router feature (known before inference — the only columns a router may use),
**O** = outcome (measured cost or result), **K** = key/join column, **C** = context/quality
(describes how the row was produced; filter or group on it, don't predict from it).

| # | Column | Unit / values | Source | Role | Notes, null rules |
|---|---|---|---|---|---|
| 1 | `run_id` | UUID | app | K | one per attempt (retries get new ids); matches `run_id` in traces |
| 2 | `session_id` | string | app | K | joins `session.json` |
| 3 | `block_id` | `<session_id>-b<NNN>` | app | K | warmup + N reps of one prompt on one backend; Option A grouping |
| 4 | `timestamp_iso` | ISO-8601 UTC | wall clock at t0 | K | |
| 5 | `t0_elapsed_ms` | ms | `SystemClock.elapsedRealtime()` at t0 | K | joins `traces/` and `thermal.csv` |
| 6 | `mode` | `collect` \| `router` | app | C | |
| 7 | `protocol_phase` | `smoke` \| `ramp` \| `cooldown` \| `sustained` | app | C | `sustained` inside a ramp session = preheat |
| 8 | `backend` | `aicore` \| `litert` \| `cloud` | app | F | |
| 9 | `model_id` | string | AICore `getBaseModelName()`; LiteRT model file name; cloud model name | C | |
| 10 | `accelerator` | `managed` \| `gpu` \| `cpu` \| `npu` \| `remote` | backend | C | AICore = `managed` (black box) |
| 11 | `repetition` | 0..N | app | C | 0 = warmup |
| 12 | `is_warmup` | 0/1 | app | C | 1 iff repetition = 0 |
| 13 | `attempt` | ≥ 1 | app | C | > 1 = retry after BUSY / timeout / pause |
| 14 | `gap_since_last_run_ms` | ms | since the previous run on the same backend ended | F | null for a backend's first run in the session |
| 15 | `prompt_id` | e.g. `L4-extract` | suite | K | |
| 16 | `prompt_category` | `xsmall` \| `small` \| `medium` \| `large` \| `xlarge` | suite | F | |
| 17 | `task_type` | `yesno` \| `extract` \| `summary` | suite | F | predicts output length |
| 18 | `prompt_chars` | characters | `prompt.length` | F | tokenizer-agnostic length feature |
| 19 | `prompt_tokens` | tokens | AICore `countTokens(prompt)`; LiteRT `lastPrefillTokenCount` | C | backend-specific tokenizer; LiteRT count includes chat template; null if unavailable |
| 20 | `thermal_status_start` | 0–6 | `PowerManager.currentThermalStatus` at t0 | F | 0 none, 1 light, 2 moderate, 3 severe, 4 critical, 5 emergency, 6 shutdown |
| 21 | `thermal_status_end` | 0–6 | at run end | O | |
| 22 | `thermal_headroom_start` | float | `PowerManager.getThermalHeadroom(0)`, 1 Hz cache | F | 1.0 ≈ SEVERE throttling threshold; null if NaN |
| 23 | `thermal_headroom_end` | float | same, at run end | O | |
| 24 | `battery_temp_start_c` | °C (0.1 resolution) | battery temperature, `ACTION_BATTERY_CHANGED` | F | lags SoC by minutes |
| 25 | `battery_temp_end_c` | °C | same, at run end | O | |
| 26 | `battery_pct_start` | % | `BATTERY_PROPERTY_CAPACITY` | F | |
| 27 | `battery_pct_end` | % | same | O | |
| 28 | `charge_counter_start_uah` | µAh | `BATTERY_PROPERTY_CHARGE_COUNTER` | C | Option A signal |
| 29 | `charge_counter_end_uah` | µAh | same | O | |
| 30 | `voltage_start_mv` | mV | battery receiver cache (or sysfs `voltage_now`; source in session.json) | C | |
| 31 | `voltage_end_mv` | mV | same | O | |
| 32 | `is_charging` | 0/1 | `BatteryManager.isCharging()` | F | |
| 33 | `plugged` | 0, 1 AC, 2 USB, 4 wireless, 8 dock | `EXTRA_PLUGGED` | F | ≠ 0 → external power, energy invalid |
| 34 | `airplane_mode` | 0/1 | `Settings.Global.AIRPLANE_MODE_ON` | F | |
| 35 | `power_save_mode` | 0/1 | `PowerManager.isPowerSaveMode` | F | |
| 36 | `ttft_ms` | ms | first non-empty chunk − t0 (monotonic) | O | null if no chunk arrived |
| 37 | `last_chunk_ms` | ms | last non-empty chunk − t0 | O | |
| 38 | `total_latency_ms` | ms | stream completion − t0 | O | on error: time until the error surfaced |
| 39 | `tokens_per_sec` | tokens/s | (output_tokens − 1) / ((last_chunk_ms − ttft_ms)/1000) | O | decode rate; null if output_tokens < 2 or last_chunk_ms ≤ ttft_ms |
| 40 | `backend_ttft_ms` | ms | LiteRT `BenchmarkInfo.timeToFirstTokenInSecond` × 1000 | O | null for AICore |
| 41 | `backend_prefill_tps` | tokens/s | LiteRT `BenchmarkInfo` | O | null for AICore |
| 42 | `backend_decode_tps` | tokens/s | LiteRT `BenchmarkInfo` | O | null for AICore |
| 43 | `output_tokens` | tokens | AICore `countTokens(output)` after the run; LiteRT `lastDecodeTokenCount` | O | null on error |
| 44 | `output_chars` | characters | output length | O | |
| 45 | `chunk_count` | count | non-empty stream chunks | O | **not** tokens |
| 46 | `finish_reason` | `stop` \| `max_tokens` \| `other` | backend | O | null if not reported |
| 47 | `output_sha1` | hex | SHA-1 of output text | O | determinism check; content is not stored |
| 48 | `peak_memory_bytes` | bytes | max app-process VmRSS (`/proc/self/status`) in run window | O | app process only — AICore's model lives in the AICore process |
| 49 | `sys_avail_mem_start_bytes` | bytes | `/proc/meminfo` MemAvailable at t0 | F | |
| 50 | `sys_avail_mem_min_bytes` | bytes | min MemAvailable in run window | O | start − min ≈ system-wide memory cost incl. AICore process |
| 51 | `cpu_time_ms` | ms | `Process.getElapsedCpuTime()` delta | O | app process only |
| 52 | `current_samples_n` | count | trace samples in [t0, t_end + W] | C | |
| 53 | `current_distinct_n` | count | distinct `CURRENT_NOW` values in window | C | < 3 → per-run energy is low quality; use block integral |
| 54 | `current_mean_ua` | µA, raw sign | mean `CURRENT_NOW` in window | O | sign as reported by the device |
| 55 | `power_mean_w` | W | mean P in window, P = s·I·V·1e-9 | O | gross (includes idle); discharge positive |
| 56 | `idle_power_baseline_w` | W | median P of the latest idle segment | C | null before the first baseline |
| 57 | `energy_joules_est` | J | trapezoid ∫(P − P_idle) dt over [t0, t_end + W] | O | never clipped; null when `energy_method = none` |
| 58 | `energy_method` | `current_integration` \| `none` | app | C | device never writes `block_delta` (assigned in Python) |
| 59 | `error_code` | int | see below | O | null on success |
| 60 | `error` | text ≤ 300 chars | exception class + message | O | empty = success |

`error_code` values: AICore `GenAiException` codes as-is (`9` BUSY, `27` daily battery quota,
`30` background blocked, `12` request too large, `8` not available, …); app codes:
`9001` stall timeout (60 s without a chunk), `9002` total timeout (180 s), `9003` aborted
(session stopped mid-run), `9004` paused (app left the foreground), `9100` LiteRT native
error, `9999` unknown.

Timing: t0 is taken after the untimed setup (prompt token count, cache clear / fresh
conversation, start snapshot). All durations use the monotonic clock.

## 3. Trace files

`traces/<block_id>.csv` — 100 ms, continuous over the block:

| Column | Unit | Notes |
|---|---|---|
| `t_elapsed_ms` | ms | `SystemClock.elapsedRealtime()` at the sample (actual, not scheduled) |
| `current_ua` | µA | raw `BATTERY_PROPERTY_CURRENT_NOW` |
| `voltage_mv` | mV | battery receiver cache or sysfs |
| `rss_kb` | kB | app VmRSS |
| `mem_avail_kb` | kB | `/proc/meminfo` MemAvailable |
| `phase` | `idle` \| `warmup` \| `run` \| `tail` \| `gap` | `tail` = the W ms after a run's last chunk (gauge lag), still tagged with its `run_id` |
| `run_id` | UUID or empty | run the sample belongs to |

Idle baselines are separate trace files, `traces/<session_id>-idleNN.csv` (all samples
`phase=idle`; first `idle_settle_ms` are settling), listed with their median power in
`session.json` → `idle_segments`. Block traces are `traces/<block_id>.csv`.

`thermal.csv` — 1 Hz, whole session: `t_elapsed_ms, thermal_status, thermal_headroom,
battery_temp_c, gyro_temp_c, pressure_temp_c, battery_pct, plugged, current_ua, voltage_mv,
mem_avail_kb`. `gyro_temp_c` / `pressure_temp_c` are Pixel vendor sensors
(`com.google.sensor.gyro_temperature`, `com.google.sensor.pressure_temp`): IMU and barometer
die temperatures, i.e. device heat.

`session.json` — `session_id`, `schema_version`, `status` (`running` → `completed` |
`stopped` | `aicore_quota` | `battery_low` | `error` | `interrupted`), start/end
timestamps, app version, device (manufacturer, model, SoC, Android release/SDK, build
fingerprint, security patch), AICore package versionName/longVersionCode + base model name +
token limit + ModelConfig (at start and end), LiteRT library version + model file + size +
accelerator + `maxNumTokens`, prompt suite id + SHA-256, generation parameters, protocol
parameters (preset, N, caps, stop level, seeds, intervals), energy parameters (current sign
`s`, gauge lag `W`, voltage source, sample interval), screen brightness override,
airplane mode at start, placement.

## 4. Collection protocol

Before every session:
1. Phone: Pixel 10, bootloader locked, system up to date — then Play Store auto-updates
   **off** for the study period (AICore / Gemini Nano update silently; version is logged per
   session anyway; a version change starts a new data epoch).
2. Diagnostics run on the same day (current sign, gauge lag W, headroom range) — the
   values are copied into `session.json` automatically.
3. Battery ≥ 80 %. **Unplug.** Airplane mode **on**, Wi-Fi off (on-device backends).
4. Close background apps, Do Not Disturb on. Phone face up on a stand, **no case**.
5. Start the session in the Collect tab. The app fixes screen brightness, keeps the screen
   on and shows a static dark screen. Do not touch the phone; the app must stay in the
   foreground (AICore refuses background requests).
6. Afterwards: plug in, `adb pull`, run `python3 tools/validate_telemetry.py <session dir>`,
   copy accepted sessions into `data/pilot/` (pilot) or the main dataset.

Presets:
- **smoke** — 1 cycle, 2 prompts, N = 1. Pipeline check; may run plugged (flagged).
- **ramp** — optional cool gate (headroom ≤ h_cool, max wait), optional preheat (`preheat_min`
  minutes of `preheat_backend` alone, phase `sustained`, so the ramp starts hot; off by default),
  cycles with no pauses,
  prompt order reshuffled per cycle (seeded), backends ABBA per prompt, blocks of warmup +
  N = 2, stop at thermal status ≥ configured level / time cap (default 60 min; 120 min used
  since 2026-11-21) / battery ≤ 30 % / user;
  then cooldown probe pairs every 3 min for up to 30 min.
- **sustained** — one backend, continuous, 20 min.
- Battery window per session: start ≥ 80 % (charged to 93–100 % since 2026-11-21) → 30 % floor.
  Exception (since 2026-11-22): deliberate low-battery sessions starting cool at ~55 %, so battery
  level is not confounded with thermal state (full sessions start full + cool, end low + hot);
  they end at the floor without cooldown; identified by `conditions_start.battery_pct`. To compare
  backends near empty, the floor may be lowered (`protocol.battery_floor_pct` in session.json, e.g. 2 %).
  Replicate ramp sessions on ≥ 3 days (H2 test).
- Plugged condition (since 2026-11-22): ramp with `allow_plugged = true`, phone charging from a
  15 W USB-C source (Mac port ≈ power bank), offline. Models the "power bank, no internet" use
  case and gives the `plugged` feature variation. Latency, TTFT, throughput, memory, thermal valid;
  energy not (`energy_method = none`). Identified by `plugged ≠ 0`; charging heat adds to thermal state.

## 5. Device spec

Filled from Diagnostics (`diag-*.json`) once the phone is authorized for adb.

| Field | Value |
|---|---|
| Model | Pixel 10 (`frankel`), 12 GB RAM (MemTotal 11,817,424 kB), 4 KB pages |
| SoC | Google Tensor G5 |
| Android | 16, SDK 36, build `google/frankel/frankel:16/BD3A.251105.010.E1/14337626:user/release-keys`, security patch 2025-11-05 (checked 2026-10-15) |
| Bootloader | locked (`ro.boot.flash.locked=1`, `verifiedbootstate=green`) |
| AICore | **Since ≤ 2026-11-21 (main dataset epoch):** `26.10.15.11.release.prod_aicore_20261015.00_RC11` (versionCode 545573), `getBaseModelName()` = **nano-v4-full**, model preference FULL, release stage STABLE. Pilot 2026-10-15/20 epoch: `0.release.prod_aicore_20260917.00_RC01` (versionCode 526925), nano-v3. System stub `aicore_20250130.00_RC01`; `getTokenLimit()` = 8192; implicit caching available; system prompt available; thinking mode unavailable; `countTokens` request overhead = 8 tokens (`countTokens("a") − 1`; subtracted, logged as `count_tokens_overhead`) |
| LiteRT-LM | `litertlm-android` 0.18.0, accelerator CPU (GPU/PowerVR decode ~2 tok/s vs CPU ~17–20 tok/s, 2026-10-20), `gemma-4-E2B-it.litertlm` (2,588,147,712 bytes, SHA-256 `181938105e0eefd105961417e8da75903eacda102c4fce9ce90f50b97139a63c`) |
| CURRENT_NOW | `BATTERY_PROPERTY_CURRENT_NOW` in uA, scale 1, sign `-1` while discharging (calibrated 2026-11-22). Sampled every 100 ms; gauge lag W = 500 ms. Voltage from `ACTION_BATTERY_CHANGED`. Values update slowly: a 1 s run yields few distinct readings, so check `current_distinct_n` and fall back to the block integral. |
| Thermal | `getThermalHeadroom(0)` readable, observed 0.43 idle to 0.85 under sustained load. `currentThermalStatus` reached 1 (LIGHT) at most over a 148 min unplugged ramp; 2 (MODERATE) was never reached, so the configured stop level 3 never triggers and sessions end on the battery floor instead. Battery temperature 35-40 C. |

## 6. Prompt suite `cuad_pilot_v1`

- Built by `tools/build_prompt_suite.py` from CUAD v1 (`CUAD_v1.zip`, SHA-256
  `88b694d99007d39777fa44cd72daf8297773d285dc3eab0091ba32078888d18e`; local copy
  `data/cuad/CUAD_v1.json` + `master_clauses.csv`, gitignored) →
  `app/src/main/assets/prompts/cuad_pilot_v1.json`.
- 15 **whole** contracts, one per prompt: 5 size levels (targets ~250, 600, 1300, 3000, 7000
  prompt tokens; the contract closest to each target, all ≤ ~7,600 tokens so prompt +
  256-token answer fit the 8,192-token window) × 3 tasks:
  `yesno` = CUAD *Termination For Convenience*, `extract` = CUAD *Governing Law*,
  `summary` = own question. Expected answers from `master_clauses.csv`
  (`<Category>-Answer`) are stored per prompt (`expected_answer`); not used by the app.
  Categories: L1 xsmall, L2 small, L3 medium, L4 large, L5 xlarge.
- Generation (identical for every backend): greedy `topK = 1`, `topP = 1`, `seed = 42`,
  `maxOutputTokens = 256`, temperature 0 where accepted.

**Attribution.** Contracts are taken from CUAD v1 — *Contract Understanding Atticus
Dataset*, The Atticus Project, licensed under CC BY 4.0
(https://creativecommons.org/licenses/by/4.0/), https://doi.org/10.5281/zenodo.4595826.
Changes: whitespace normalised; each whole contract wrapped in a question template. Citation: Hendrycks, Burns, Chen, Ball. *CUAD: An Expert-Annotated NLP
Dataset for Legal Contract Review.* NeurIPS 2021.

## 7. Pilot data

Each committed session keeps its full directory (`session.json`, `runs.csv`, `thermal.csv`,
`outputs.csv`, `traces/`) under `data/pilot/<session_id>/` and a line here.

| Session | Preset | Backends | Rows total | Measured / warmup | Errors | Energy | Notes |
|---|---|---|---|---|---|---|---|
| `20261015-203459-sustained` | sustained, N = 3 + warmup, 10 min | aicore (nano-v3) | 75 | 48 / 27 | 11 x BUSY | invalid | First pilot. Plugged in over adb, battery 100 %, so `energy_method = none`. Headroom 0.43 to 0.66, battery 27.9 to 33.9 C, `thermal_status` 0 throughout. BUSY from run 58 (~8 min in); 7 ok rows lack token counts because `countTokens` hit the same quota. Energy calibration not yet measured. |
| `20261122-185632-ramp` | ramp, N = 2 + warmup, 120 min cap | aicore (nano-v4-full), litert | 516 | 344 / 172 | 18 x AICore policy | **valid** | Main reference session. Unplugged, battery 40 to 2 %, ended `battery_low` after 148 min. Headroom 0.62 to 0.85, `thermal_status` 0 to 1, battery 35.3 to 39.4 C. 172 measured runs per backend, all 15 prompts, prompt sizes 245 to 5381 tokens. `energy_method = current_integration` on every row, median 56 samples per run. Deliberate low-start-battery design so battery level is not confounded with thermal state. |
| `20261122-213050-ramp` | ramp, N = 2 + warmup, 120 min cap | aicore (nano-v4-full), litert | 224 | 149 / 75 | 6 x AICore policy | invalid | Plugged condition (15 W USB-C, offline), battery 7 to 29 %, `status = interrupted`. `energy_method = none` by design; latency, throughput, memory and thermal columns are usable. Gives `plugged` feature variation for the "power bank, no internet" case. |

Totals committed: 815 rows, 541 measured runs, of which 344 carry valid energy.

Pipeline test only, not pilot data: `data/raw/20261015-203302-smoke` (4 rows, pre-overhead-fix
token counts).

### What is still missing

- Two further unplugged ramp sessions on different days, to reach about 1000 measured runs
  and to confirm that results repeat (section 4, "Replicate ramp sessions on >= 3 days").
- The cloud backend, 15 prompts x 2 repetitions = 30 runs, in one separate online session.

## 8. Known data-quality caveats

- AICore memory and CPU time are not visible to the app (separate process); use
  `sys_avail_mem_*` as a system-level proxy and say so.
- Per-run energy depends on the fuel gauge's update rate and averaging; check
  `current_distinct_n` and prefer block integrals when it is low.
- Battery temperature lags; prefer `thermal_headroom_*`.
- Greedy repetitions are not independent samples; cross-validate grouped by prompt and
  session.
- AICore streams in few, large chunks (1–2 chunks for short answers), so `tokens_per_sec`
  (first → last chunk) is coarse for short outputs; Python can also use
  `output_tokens / (total_latency_ms − ttft_ms)`. Raw timings are all in the row.
- The first run after a model (re)load is slow (smoke 2026-10-15: warmup TTFT 1958 ms vs 632 ms
  for the next rep) — that is why every block starts with a logged warmup (`is_warmup=1`).
- Sessions run while plugged in (`plugged ≠ 0`, e.g. adb-started tests) have no valid energy
  data (`energy_method=none`); latency, tokens and thermal columns remain usable.
- Smoke session `20261015-203302-smoke` predates the `countTokens` overhead fix: its
  `prompt_tokens`/`output_tokens` include ≈ 9 template tokens and its `tokens_per_sec` is
  inflated accordingly.
- The AICore daily battery quota can end sessions early (`status = aicore_quota`); these
  sessions are kept, not discarded.
- `thermal_status` did not exceed 1 (LIGHT) in any session so far, even over 148 minutes of
  continuous load. Thermal variation in the dataset is therefore carried by
  `thermal_headroom_*` and `battery_temp_*`, not by the coarse status level. Report it that way.
- AICore rejects some CUAD prompts with `REQUEST_PROCESSING_ERROR` (ErrorCode 4, content
  policy): 18 of 516 rows (3.5 %) in `20261122-185632-ramp` and 6 of 224 in
  `20261122-213050-ramp`, always AICore, never LiteRT. These rows are logged as errors, kept,
  and the rejection rate is itself a reportable result for a legal-domain workload.
- LiteRT-LM runs on CPU because the GPU delegate was measured at about 2 tok/s against 17 to
  20 tok/s on CPU (2026-10-20). The comparison is therefore between the runtimes as they can
  actually be deployed on this device, not between two runtimes on the same hardware unit.
