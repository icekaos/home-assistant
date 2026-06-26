# Blueprints

This folder contains Home Assistant automation blueprints used by this
configuration.

## Terrace Irrigation System

File: `irrigation_blueprint.yaml`

The `Terrace Irrigation System` blueprint automates terrace irrigation using two
soil moisture sensors, one representative of small pots and one representative
of large pots. It calculates a weighted average moisture value and starts
irrigation only when the terrace is dry enough and the current time is inside
the configured irrigation window.

## What It Does

The automation:

- reads the moisture value from the small-pot sensor;
- reads the moisture value from the large-pot sensor;
- calculates a weighted average moisture value;
- chooses the active threshold based on normal mode or vacation mode;
- checks whether the current time is inside the allowed watering window;
- opens the irrigation valve when watering is needed;
- retries once if the valve does not respond;
- closes the valve after the configured duration;
- waits for a cooldown period to allow water absorption;
- optionally sends start, stop, and warning notifications.

## Required Entities

Each automation created from this blueprint needs:

- one moisture sensor for small pots;
- one moisture sensor for large pots;
- one switch entity controlling the irrigation valve;
- one `input_boolean` used as manual irrigation pause;
- one `input_boolean` used as vacation mode;
- one mobile app notification device.

The moisture sensors should expose `device_class: moisture`.

## Moisture Calculation

The blueprint does not use a single sensor directly. It calculates a weighted
average:

```text
weighted_moisture = small_pot_moisture * small_pot_weight
                  + large_pot_moisture * (1 - small_pot_weight)
```

For example, with `small_pot_weight = 0.7`:

```text
weighted_moisture = small_pots * 0.7 + large_pots * 0.3
```

This lets the small pots influence the irrigation decision more strongly, which
is useful because small pots usually dry faster.

## Thresholds

The blueprint has two irrigation thresholds:

- `normal_threshold`: used during normal operation;
- `vacation_threshold`: used when the vacation mode helper is `on`.

When vacation mode is enabled, the automation uses the vacation threshold and
the vacation irrigation duration. This is useful if you want watering to start
earlier or last longer while nobody is at home.

The trigger condition is:

```text
weighted_moisture < active_threshold
```

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

The template also supports windows that cross midnight. For example:

```text
22:00 -> 06:00
```

In that case the end time is treated as belonging to the next day. The blueprint
also checks the equivalent window from yesterday, so that `02:00` is correctly
recognized as being inside the watering window that started the previous night.

## When The Automation Starts

The automation starts when all of these are true:

- the weighted moisture is below the active threshold;
- the current time is inside the configured watering window;
- the manual pause helper is `off`;
- the irrigation valve is currently `off`.

There is no additional `for:` delay on the template trigger. This means the
automation can start as soon as Home Assistant evaluates the template as true.
In practice, this usually happens when one of the referenced entities changes
state or when Home Assistant refreshes the time-based template evaluation.

## Why It Does Not Loop Continuously

The automation uses:

```yaml
mode: single
```

While one irrigation cycle is running, new triggers are ignored.

The action sequence also includes a cooldown delay after the valve is closed.
During the irrigation duration and the cooldown period, the automation is still
running, so another cycle cannot start.

After the automation finishes, it will not immediately start again unless the
template trigger changes from `false` to `true` again. If the moisture remains
below the threshold the whole time, the template remains true and does not
produce a new trigger. A new cycle can happen later if the condition first
becomes false and then becomes true again, or when a new watering window opens.

## Valve Retry Logic

After sending the first `switch.turn_on` command, the automation waits for the
configured retry delay.

If the valve is still `off`, it sends a second `switch.turn_on` command and
waits again.

If the valve is still `off` after the retry, the automation sends a notification
and stops. This is intended to handle slow or sleeping Zigbee devices.

## Notifications

The blueprint can optionally notify when irrigation starts and stops.

It always has warning notification paths for:

- valve failure after two attempts;
- moisture still below threshold after irrigation and cooldown.

The post-irrigation warning does not start a new watering cycle. It only reports
that the measured moisture is still low after the configured absorption delay.

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

Fill in the sensors, valve, helpers, thresholds, watering window, durations, and
notification settings for the terrace.

## Suggested Defaults

Typical starting values:

```text
small_pot_weight: 0.7
normal_threshold: 40%
vacation_threshold: 50%
normal_duration: 5 minutes
vacation_irrigation_duration: 8 minutes
cooldown_minutes: 30 minutes
retry_wait_seconds: 10 seconds
```

Adjust these values after observing how quickly the soil moisture changes after
watering.

## Operational Notes

Use the dashboard plant thresholds for visibility and diagnostics, and use the
blueprint threshold for the actual terrace irrigation decision. The blueprint
threshold is based on the weighted average of the two terrace sensors, so it may
not always match the threshold of a single plant or pot group.

If the moisture sensors update every few minutes, keeping the trigger without a
long `for:` delay makes the automation respond at the first useful sensor
update inside the watering window.
