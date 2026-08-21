# Rockchip PWM fan control — working notes

Scratch notes for the `rock-5b-fan-pwm-frequency` branch. Not for upstream
submission as-is. Delete before sending patches.

## Status

| Area | State |
|---|---|
| PWM carrier frequency (coil whine) | **Done**, committed — 5B, 5B+, 5T |
| Nine-level cooling ladder on 5B | Committed, but see "Open question 1" |
| Fan not actuated at all | **Fix written, uncommitted** — DT + 3 driver files |
| Other affected boards | **Not started** — see "Remaining boards" |

Everything below the first two rows is **uncommitted working-tree state**.

## What was verified, and how

All of this is static analysis of this repo. **None of it has been confirmed on
hardware or in a real Armbian build.** The kernel config on an actual Armbian
image may differ from `arch/arm64/configs/rockchip_linux_defconfig`, and the
central conclusion depends on that config — see "Open question 2".

### The fan is never driven by the thermal core

Chain, each link checked in-tree:

1. `rockchip_linux_defconfig:283` — `CONFIG_THERMAL_DEFAULT_GOV_POWER_ALLOCATOR=y`.
2. `drivers/thermal/thermal_of.c` never parses a governor property; `tzp` is
   `kzalloc`'d, so `__find_governor("")` returns `def_governor`
   (`thermal_core.c:56`, `:1289`). `soc_thermal` therefore runs
   **power_allocator**, not step_wise.
3. The board's trip nodes are named `trip-point@N`; the base dtsi's are
   `trip-point-N`. Different names, so they **merge** rather than override.
   Decompiled DTB had 11 trips in DT order:
   `[0]` 75 C passive, `[1]` 85 C passive, `[2]` 115 C critical,
   `[3..10]` the eight board active trips. `thermal_of.c` does not sort trips.
4. `get_governor_trips()` (`gov_power_allocator.c`) walked trips in index order
   and `break`'d on the first non-passive/non-active type — the critical trip at
   index 2. Trips 3-10 were never seen.
5. `power_allocator_throttle()` opens with
   `if (trip != params->trip_max_desired_temperature) return 0;` — returns
   immediately for every fan trip.
6. Even past that, `allocate_power()` only touches instances where
   `cdev_is_power_actor(cdev)`. `pwm_fan_cooling_ops` has no
   `get_requested_power` / `state2power` / `power2state`, so the fan is skipped.

`check_power_actors()` would reject the fan, but it only runs at
`power_allocator_bind()` time, when the instance list is still empty — cooling
devices bind later. So there is no fallback to step_wise.

Consequence: `pwm_fan_probe()` does `set_pwm(ctx, MAX_PWM)` and nothing ever
changes it. Matches the reported "Rock 5B fan runs flat out" behaviour.

### Hysteresis was dead code

`gov_step_wise.c` computed `throttle` as a bare `tz->temperature >= trip_temp`
and never called `get_trip_hyst`. The `hysteresis` DT property fed only
`__thermal_zone_set_trips()` (an IRQ window that `rockchip_thermal.c:2202`
discards — it programs `high` only, ignoring `low`) and netlink notifications.

Note: 5 C hysteresis on 5 C trip spacing is *fine* once hysteresis works. Trip N
disengages exactly where trip N-1 engages — a clean monotone ladder, no chatter,
no gap. The spacing was never the bug.

## How to confirm on hardware

```sh
cat /sys/class/thermal/thermal_zone0/policy          # predicted: power_allocator
cat /sys/class/thermal/thermal_zone0/type            # soc-thermal
grep . /sys/class/thermal/cooling_device*/type
grep . /sys/class/thermal/cooling_device*/cur_state  # fan pinned at max?
cat /sys/class/hwmon/hwmon*/pwm1                     # expect 255 pre-fix
```
Then load the SoC and watch whether `pwm1` moves.

## Uncommitted changes

### DT — moved fan off cooling-maps onto `rockchip,temp-trips`

- `arch/arm64/boot/dts/rockchip/rk3588-rock-5b.dts`
  - Added `rockchip,temp-trips` (45-80 C -> states 1-8).
  - **Deleted the whole `&soc_thermal` override**: eight active trips and eight
    cooling-maps, all dead. It also set `polling-delay-passive = <2000>`,
    overriding the SoC default of 20 ms and slowing CPU/GPU passive throttling
    for no reason; removing it restores the default.
  - `cooling-levels` deliberately kept — `pwm_fan_of_get_cooling_data()` still
    parses it into the state -> duty table that temp-trips indexes into.
- `rk3588-rock-5b-plus.dts`, `rk3588-rock-5t.dts`
  - Same conversion. Their maps bound the fan to `&target` / `&threshold`;
    `&target` *is* the control trip, but the fan still got skipped for not being
    a power actor, so equally dead.
  - Dropped map4/map5, kept `sustainable-power = <5000>`.
  - **Left `&threshold { temperature = <60000>; }` alone on purpose** — that is
    the power_allocator switch-on point for CPU/GPU, not a fan setting.

`rockchip,temp-trips` works because `rockchip-system-monitor` is bound to
`soc-thermal` (`rk3588s.dtsi:2200`) and polls `thermal_zone_get_temp()` every
200 ms (`THERMAL_POLLING_DELAY`), notifying `pwm_fan_thermal_notifier_call()`
independently of any governor.

### Drivers

- `drivers/hwmon/pwm-fan.c` — `pwm_fan_temp_to_state()` had **zero** hysteresis,
  a pure ascending threshold scan. Added `PWM_FAN_TEMP_HYST` (2000 mC), applied
  only to trips at or below the currently occupied state, so the fan speeds up
  immediately but will not hunt on the way down.
  - Interaction to be aware of: `rockchip_system_monitor_thermal_update()`
    already suppresses falling-temperature notifications until the drop exceeds
    2000 mC from `last_temp`. The two compose, giving an effective falling
    deadband of roughly 2-4 C. Tune `PWM_FAN_TEMP_HYST` with that in mind.
- `drivers/thermal/gov_step_wise.c` — made hysteresis real. Added
  `thermal_trip_is_engaged()` as a latch, plus a branch that keeps throttling
  while the temperature sits inside `(trip_temp - hyst, trip_temp)` and the trip
  is already engaged. Rising edge still fires at the exact trip temperature.
  - **Blast radius: every thermal zone on every platform using step_wise.**
    Cooling states now hold ~`hysteresis` longer than before. That is the
    documented intent of the property, but it is a behaviour change well beyond
    these boards. Does not affect the Rock 5 boards after the DT change above,
    since they no longer use the thermal core for the fan at all. Consider
    whether to keep this in the same series or split it out.
- `drivers/thermal/gov_power_allocator.c` — `get_governor_trips()` no longer
  `break`s on a hot/critical trip, so trips described after one are still seen.
  Order-dependence fix; changes nothing for these boards
  (`trip_max_desired` stays index 1).

### Verification done

- All three DTBs compile (`cpp` + `dtc`, exit 0); decompiled and confirmed the
  emitted `rockchip,temp-trips` values and that **no thermal zone references the
  fan phandle any more**.
- All three .c files pass `gcc -fsyntax-only -std=gnu11` with kernel include
  flags, exit 0.
- **No real kernel build**: no aarch64 cross-compiler on that machine, and
  objtool needs libelf which was not installed. Syntax/type check only — not a
  link, not a boot.

## Open questions

1. **Commit `dd3ccc4ac62a` ("list all nine fan cooling levels")** was written on
   the belief that the thermal core consults `cooling-levels` via the maps. It
   does not. The nine-level table is still used — temp-trips indexes into it —
   so the commit is not wrong, but its commit message explains the change in
   terms of `max_state` and cooling-map binding, which is not the real
   mechanism. Reword before submitting.
2. **Armbian's actual kconfig may differ.** If it sets
   `CONFIG_THERMAL_DEFAULT_GOV_STEP_WISE`, the cooling-map route would have
   worked and the DT conversion is unnecessary (though not harmful — temp-trips
   takes priority in `pwm_fan_probe()` and returns before registering the
   cooling device). Check `zcat /proc/config.gz | grep THERMAL_DEFAULT_GOV`
   on a real image before committing to this direction.
3. **The 4-step ladder on 5B+ / 5T is unvalidated** — carried over from their
   old `cooling-levels`. The 5B uses nine levels over the same 45-80 C span.
   Marked `FIXME` in both DTs.
4. Three other governor-selection routes were considered and rejected for now:
   forcing `policy=step_wise` via udev (userspace, not a kernel fix), and
   teaching `thermal_of` to parse a `governor` DT property (correct in general,
   largest blast radius).

## Remaining boards

Same "fan pinned at max" defect, not yet touched. Boards using cooling-maps with
a non-power-actor fan on a power_allocator zone:

- `rk3588-rock-5-itx.dts`, `rk3588s-rock-5c.dts`, `rk3588s-radxa-e52c.dts`,
  `rk3588s-radxa-e54c.dts`, `rk3588s-radxa-nx5-io.dts`, `rk3576-rock-4d.dts`,
  `rk3576-radxa-cm4-io.dts`

Boards already on `rockchip,temp-trips` (so they only needed the pwm-fan
hysteresis fix): `rk3588s-rock-5a.dts`, `rk3588s-radxa-cm5-io.dts`,
`rk3588-orangepi-5-ultra.dts`, `rk3566-orangepi-3b-v2.1.dts`,
`rk3576-recomputer-rk3576-devkit.dts`.

Still-audible PWM carriers found in the earlier sweep, unfixed:

- 16.667 kHz (60000 ns): `rk3576-rock-4d.dts`, `rk3576-radxa-cm4-io.dts`,
  `rk3576-recomputer-rk3576-devkit.dts`
- 4 kHz (250000 ns): `rk3588-blade3-v101-linux.dts`, `rk3588s-lubancat-4.dts`
- **50 Hz** (20000000 ns): `rk3588-orangepi-5-ultra.dts`,
  `rk3566-orangepi-3b-v2.1.dts` — audible flutter, worst in the tree
- ~40 boards sit at 20 kHz (50000 ns). Right at the edge of audibility and the
  de-facto convention in this tree; deliberately left alone.
