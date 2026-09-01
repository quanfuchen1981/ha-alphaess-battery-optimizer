# ha-alphaess-battery-optimizer

Self-learning Home Assistant automations for an **AlphaESS** home battery + solar
system in a variable/time-of-use electricity market. The goal is simple: charge
the battery enough overnight (on cheap off-peak power) to get through the next
day without running flat before solar kicks in again, but not a single percent
more than necessary — because every extra percent charged overnight is money
that didn't need to be spent.

This is not a static rule set. The system blends two independent solar
forecast providers, buckets tomorrow's expected generation into tiers, and then
**learns** from what actually happened overnight (did the battery run
dangerously low, or stay wastefully high?) to nudge its own targets over time.

## Why this exists

Most "charge to 80% overnight" automations are a single fixed number that gets
tuned once and never revisited. That either wastes money (charging to 80% on a
day when 50% would have been plenty) or risks running the battery flat before
sunrise on a day the forecast undershot. This project tries to close that loop
automatically, with safety bounds that can't be learned away.

## How it works

### 1. Blended solar forecast

Two independent forecast sources (Solcast P50 and Forecast.Solar P10) are
blended, weighted by a tunable pessimism factor
(`input_number.alphaess_pv_pessimism_weight`), to produce a single
`corrected_solar` estimate for tomorrow's generation in kWh. Blending two
providers with different biases is more robust than trusting either one alone.

### 2. Tiered overnight charge targets

`corrected_solar` is bucketed into five tiers (poor / limited / usable / good
/ strong, split at 6 / 8 / 11 / 14 kWh), each mapped to a **charge target
percentage** that used to be a hardcoded constant (85 / 75 / 65 / 55 / 45%).
Those five constants are now `input_number` helpers
(`input_number.battery_target_tier_*`), so they can be tuned live from the
dashboard — or adjusted automatically (see below) — without editing YAML.

Weather-conditional floors (e.g. a rainy or heavily overcast day ahead) can
still raise the effective target regardless of tier, and a final clamp keeps
every computed target within a hard 45–85% safety band no matter what the
tiers or floors say.

### 3. Closed-loop self-learning

Every morning, `Energy - Battery Target Self-Learning` looks at what actually
happened the previous night:

- parses last night's forecast tier out of the logged decision string
- reads the actual overnight minimum battery SoC
  (`input_number.overnight_min_battery_soc`)
- classifies the outcome: **LOW** (SoC dropped to ≤13%, too close to running
  out), **HIGH** (SoC stayed ≥30%, i.e. it over-charged and wasted cheap-rate
  budget that didn't need spending), or **OK**
- nudges *only that tier's* target input_number up (LOW) or down (HIGH) by a
  configurable step, clamped to 40–90%
- logs every adjustment (or the reason none was made) to the Logbook for
  transparency
- de-duplicates so the same night's decision is never processed twice

Over days/weeks, each tier's target converges toward the minimum charge level
that's still safe for that kind of forecast day — which is exactly the
quantity you want to minimize your electricity bill without gambling on
running the battery flat overnight.

### 4. Safety design

The self-learning loop can only move numbers that are already bounded on both
ends:

- each tier target is clamped to 40–90% by the self-learning automation itself
- the advisory automation's final output is *separately* clamped to 45–85%
  regardless of what the tiers say
- weather floors force a higher target on rain/heavy-cloud days independent of
  the learned tier value
- the loop only ever adjusts *one* tier by a few percentage points per day, so
  a single bad forecast or a one-off anomalous night can't cause a large,
  sudden swing

## Automations in this repo

| File | Purpose |
|---|---|
| `automations/nightly_battery_charge_advisory.yaml` | Core nightly planner: blends forecasts, buckets into tiers, applies weather floors and the final safety clamp, writes the overnight charge target. |
| `automations/battery_target_self_learning.yaml` | The closed-loop tuner described above — reads last night's outcome and adjusts the relevant tier's target. |
| `automations/nightly_effect_logger.yaml` | Records the effect of each night's decision for later analysis. |
| `automations/log_charge_target_decisions.yaml` | Logs each computed decision (forecast, target, weather) for the self-learning automation to parse the next morning. |
| `automations/track_overnight_min_soc.yaml` | Tracks the lowest battery SoC observed overnight. |
| `automations/reset_overnight_min_soc_tracker.yaml` | Resets the overnight minimum SoC tracker each day so it reflects only the most recent night. |

## Required helpers

Create these via **Settings → Devices & services → Helpers** before importing
the automations (all referenced by entity ID inside the YAML):

**Number helpers (`input_number`)**

| Entity ID | Suggested min/max | Purpose |
|---|---|---|
| `input_number.battery_target_tier_poor` | 40–90 | Target % for the poorest forecast tier (<6 kWh) |
| `input_number.battery_target_tier_limited` | 40–90 | Target % for 6–8 kWh forecast |
| `input_number.battery_target_tier_usable` | 40–90 | Target % for 8–11 kWh forecast |
| `input_number.battery_target_tier_good` | 40–90 | Target % for 11–14 kWh forecast |
| `input_number.battery_target_tier_strong` | 40–90 | Target % for ≥14 kWh forecast |
| `input_number.overnight_min_battery_soc` | 0–100 | Lowest SoC observed overnight (written by the tracker automation) |
| `input_number.alphaess_pv_pessimism_weight` | 0–1 | How much weight the more conservative forecast source gets when blending |

**Text helpers (`input_text`)**

| Entity ID | Purpose |
|---|---|
| `input_text.last_night_charge_decision` | The logged decision string the self-learning automation parses each morning |
| `input_text.battery_learning_last_processed` | Bookkeeping so the same night's decision is never processed twice |

You will also need your own solar forecast integrations (Solcast and
Forecast.Solar, or equivalents) and an AlphaESS integration exposing a battery
SoC sensor and a way to set the charge target/schedule.

## Adjusting the learning rate

The self-learning automation exposes `step_up` and `step_down` as script
variables near the top of its action sequence — increase them for faster
convergence (at the cost of more visible day-to-day swings in the target), or
decrease them for a slower, steadier approach.

## Disclaimer

These automations encode one household's specific tariff structure, battery
hardware, and risk tolerance. Review every threshold, entity ID, and clamp
value against your own setup before relying on it — in particular, do **not**
loosen the safety clamps without understanding why they're there.

## License

MIT — see `LICENSE`.
