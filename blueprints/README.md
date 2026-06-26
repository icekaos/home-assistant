# Blueprints

This folder contains Home Assistant automation blueprints used by this
configuration.

## Terrace Irrigation System

File: `irrigation_blueprint.yaml`

The `Terrace Irrigation System` blueprint controls a terrace irrigation valve
using a precomputed terrace moisture sensor and a shared threshold helper.

The moisture calculation is intentionally kept outside the blueprint. This keeps
the dashboard and the irrigation automation aligned: both read the same terrace
moisture value and the same threshold.

## Architecture

The recommended setup is:

```text
raw small-pot moisture sensor
raw large-pot moisture sensor
        |
        v
template terrace moisture sensor
        |
        +--> dashboard thirsty status
        |
        +--> irrigation blueprint
```

In this repository, the shared terrace moisture sensors are defined in:

```text
packages/notify_plant.yaml
```

Examples:

```text
sensor.plant_moisture_ts
sensor.plant_moisture_tcm
```

The matching threshold helpers are also defined in the package:

```text
input_number.plants_threshold_ts
input_number.plants_threshold_tcm
```

## Why The Moisture Calculation Is Outside The Blueprint

Home Assistant blueprints are best suited for reusable automations and scripts.
Template sensors are regular Home Assistant entities, so they belong in normal
configuration, such as a package.

Keeping the weighted moisture sensor outside the blueprint has three practical
benefits:

- the dashboard can show the same value used by irrigation;
- the blueprint stays simpler and only decides whether to irrigate;
- thresholds are configured once and reused consistently.

## Terrace Moisture Calculation

The terrace moisture sensor is calculated from the small-pot and large-pot
moisture sensors:

```text
terrace_moisture = small_pot_moisture * small_pot_weight
                 + large_pot_moisture * (1 - small_pot_weight)
```

For example, with `small_pot_weight = 0.7`:

```text
terrace_moisture = small_pots * 0.7 + large_pots * 0.3
```

This gives more importance to small pots, which usually dry faster.

## Required Inputs

Each automation created from this blueprint needs:

- a terrace moisture sensor, for example `sensor.plant_moisture_ts`;
- an irrigation threshold helper, for example `input_number.plants_threshold_ts`;
- a switch entity controlling the irrigation valve;
- an `input_boolean` used as manual irrigation pause;
- a mobile app notification device;
- irrigation duration, watering window, cooldown, and retry settings.

Vacation mode is intentionally not part of this blueprint.

## When Irrigation Starts

The automation starts when all of these are true:

- the terrace moisture sensor is below the selected threshold helper;
- the current time is inside the configured watering window;
- the manual pause helper is `off`;
- the irrigation valve is currently `off`.

The core condition is:

```text
states(moisture_sensor) < states(irrigation_threshold)
```

There is no long `for:` delay on the template trigger. The automation can start
as soon as Home Assistant evaluates the template as true, typically after a
referenced entity changes state or after a time-based template refresh.

## Watering Window

Irrigation is allowed only inside a configured time window.

The start and end of the window can each be configured as:

- a fixed time;
- sunrise, with an optional offset;
- sunset, with an optional offset.

Examples:

```text
06:00 -> 09:00
sunrise + 30 min -> 09:00
sunset - 60 min -> sunset + 30 min
```

The template also supports windows that cross midnight:

```text
22:00 -> 06:00
```

In that case the end time is treated as belonging to the next day. The blueprint
also checks the equivalent window from yesterday, so that `02:00` is correctly
recognized as part of the window that started the previous night.

## Irrigation Cycle

When triggered, the automation:

- opens the irrigation valve;
- waits for the configured retry delay;
- retries once if the valve is still `off`;
- sends a warning and stops if the valve does not respond after the retry;
- optionally sends a start notification;
- keeps the valve open for the configured duration;
- closes the valve;
- optionally sends a stop notification;
- waits for the cooldown period;
- checks whether moisture is still below threshold and sends a warning if so.

The post-irrigation warning is informational only. It does not start another
watering cycle.

## Why It Does Not Loop Continuously

The automation uses:

```yaml
mode: single
```

While one irrigation cycle is running, new triggers are ignored.

The action sequence includes the irrigation duration and the cooldown delay, so
the automation remains busy during both phases. After it finishes, it will not
start again unless the template trigger changes from `false` to `true` again.

If the moisture remains below the threshold continuously, the template remains
true and does not produce a new trigger. A new cycle can happen later if the
condition first becomes false and then becomes true again, or when a new
watering window opens.

## Notifications

The blueprint supports optional notifications for:

- irrigation start;
- irrigation stop.

It also sends warning notifications for:

- valve failure after two attempts;
- moisture still below threshold after irrigation and cooldown.

## Installation

Copy the blueprint file to your Home Assistant blueprint folder:

```text
config/blueprints/automation/irrigation_blueprint.yaml
```

Then reload blueprints or restart Home Assistant.

Create an automation from:

```text
Settings -> Automations & Scenes -> Create Automation -> From Blueprint
```

Select:

```text
Terrace Irrigation System
```

For each terrace, select the matching shared entities.

Example for Terrazzo Salotto:

```text
Terrace moisture sensor: sensor.plant_moisture_ts
Irrigation threshold: input_number.plants_threshold_ts
```

Example for Terrazzo Camera Matrimoniale:

```text
Terrace moisture sensor: sensor.plant_moisture_tcm
Irrigation threshold: input_number.plants_threshold_tcm
```

## Suggested Defaults

Typical starting values:

```text
small_pot_weight: 0.7
irrigation_threshold: 35-40%
irrigation_duration: 5 minutes
cooldown_minutes: 30 minutes
retry_wait_seconds: 10 seconds
```

Adjust these values after observing how quickly the measured terrace moisture
changes after watering.
