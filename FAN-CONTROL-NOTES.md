# Rockchip PWM fan control — working notes

Scratch notes for the `rock-5b-fan-pwm-frequency` branch. Not for upstream
submission as-is. Delete before sending patches.

## Status

| Area | State |
|---|---|
| PWM carrier frequency (coil whine) | **Done**, committed — 5B, 5B+, 5T |
| Nine-level cooling ladder on 5B | **Done**, committed — confirmed correct, see below |
| Hysteresis was dead code | **Done**, committed — `gov_step_wise.c` |
| pwm-fan vendor path had no hysteresis | **Done**, committed — affects other boards |
| power_allocator trip discovery | **Done**, committed — robustness only |
| Other affected boards | **Not started** — see "Remaining boards" |

## IMPORTANT: earlier power_allocator analysis was wrong for this board

An earlier pass concluded that `soc_thermal` runs **power_allocator**, that the
cooling-maps were therefore inert, and that the fan had to be moved onto
`rockchip,temp-trips`. That conversion was written, committed, and then
**reverted** — see the revert commit on this branch.

The premise was false. The real Rock 5B runs **step_wise**, not
power_allocator. The earlier conclusion came from
`arch/arm64/configs/rockchip_linux_defconfig:283`
(`CONFIG_THERMAL_DEFAULT_GOV_POWER_ALLOCATOR=y`), but this repo's defconfig is
not what the running image is built with. Confirm the governor on the target,
never from the defconfig:

```sh
cat /sys/class/thermal/thermal_zone0/policy       # soc-thermal: step_wise here
cat /sys/class/thermal/thermal_zone0/available_policies
```

Under step_wise the cooling-map route works correctly, so the DT stays as it
was and the fix is entirely in the governor. Keep this in mind before acting on
any defconfig-derived reasoning in this tree.

The parts of that analysis that remain true regardless of governor, and are
worth keeping:

- The board's trip nodes are named `trip-point@N`, the base dtsi's are
  `trip-point-N`. Different names, so they **merge** rather than override.
  The compiled DTB has 11 trips in DT order: `[0]` 75 C passive,
  `[1]` 85 C passive, `[2]` 115 C critical, `[3..10]` the eight board active
  trips. `thermal_of.c` does not sort them. Harmless under step_wise, which
  processes every trip index independently.
- `thermal_of.c` never parses a governor property, so the governor cannot be
  selected from DT in this tree — only by kconfig default or by writing
  `/sys/class/thermal/thermal_zone0/policy`.
- `get_governor_trips()` in `gov_power_allocator.c` stopped trip discovery at
  the first hot/critical trip. Real order-dependence bug, fixed on this branch,
  but it does not affect these boards.
- `pwm_fan_cooling_ops` implements none of the power-actor ops, so a fan can
  never be actuated by power_allocator. Relevant to any board that *does*
  default to that governor.

### Why commit "list all nine fan cooling levels" was right

`thermal_zone_bind_cooling_device()` rejects a map with
`upper > cdev->max_state` (returns `-EINVAL`). With the original five
`cooling-levels`, `max_state` was 4, so map4 through map7
(`<&fan0 4 5>` .. `<&fan0 7 8>`) all failed to bind and the fan could not
respond above 65 C. Extending the ladder to nine levels fixed real breakage.
The commit message stands as written.

## What was verified, and how

### Hysteresis was dead code

`gov_step_wise.c` computed `throttle` as a bare `tz->temperature >= trip_temp`
and never called `get_trip_hyst`. The `hysteresis` DT property fed only
`__thermal_zone_set_trips()` (an IRQ window that `rockchip_thermal.c:2202`
discards — it programs `high` only, ignoring `low`) and netlink notifications.

Note: 5 C hysteresis on 5 C trip spacing is *fine* once hysteresis works. Trip N
disengages exactly where trip N-1 engages — a clean monotone ladder, no chatter,
no gap. The spacing was never the bug.

Traced through the live configuration to be sure. `thermal_cdev_update()` takes
the max target across instances, so at 62 C the engaged trips 45/50/55/60 give
targets converging on 4 (the upper of the 60 C map). Falling to 58 C, the 60 C
trip is inside its band and stays engaged, holding state 4. Only below 55 C does
it drop to 3, where the 55 C trip's band holds it. Monotone the whole way down.
Without the fix each boundary chatters instead.

**This is now the primary fix for the Rock 5B**, not a side improvement.

## How to confirm on hardware

```sh
cat /sys/class/thermal/thermal_zone0/policy          # predicted: power_allocator
cat /sys/class/thermal/thermal_zone0/type            # soc-thermal
grep . /sys/class/thermal/cooling_device*/type
grep . /sys/class/thermal/cooling_device*/cur_state  # fan pinned at max?
cat /sys/class/hwmon/hwmon*/pwm1                     # expect 255 pre-fix
```
Then load the SoC and watch whether `pwm1` moves.

## Changes on this branch

All committed. Nothing is pushed upstream, so any of it can still be reworded,
split, or dropped.

### DT — unchanged from mainline behaviour, deliberately

The `rockchip,temp-trips` conversion was reverted (see the IMPORTANT section
above). The three board DTs now differ from their base only by the PWM period
fix and the nine-level ladder. **The cooling-maps are the right mechanism here**:
under step_wise they work, they honour the DT `hysteresis` property, they are
tunable without a driver change, and they are upstreamable, none of which is
true of the Rockchip-specific `rockchip,temp-trips` bypass.

For reference if the governor question ever comes back: `rockchip,temp-trips`
works because `rockchip-system-monitor` is bound to `soc-thermal`
(`rk3588s.dtsi:2200`) and polls `thermal_zone_get_temp()` every 200 ms
(`THERMAL_POLLING_DELAY`), notifying `pwm_fan_thermal_notifier_call()`
independently of any governor. It takes priority in `pwm_fan_probe()`, which
returns before registering the cooling device — so adding it *disables* the
cooling-map route rather than complementing it.

### Drivers

- `drivers/thermal/gov_step_wise.c` — **the actual fix for the Rock 5B.** Made
  hysteresis real: added `thermal_trip_is_engaged()` as a latch, plus a branch
  that keeps throttling while the temperature sits inside
  `(trip_temp - hyst, trip_temp)` and the trip is already engaged. The rising
  edge still fires at the exact trip temperature; only the step back down is
  delayed.
  - **Blast radius: every thermal zone on every platform using step_wise**,
    including CPU/GPU throttling on these very boards, since they share
    `soc_thermal` with the fan. Cooling states now hold about one hysteresis
    band longer. That is the documented intent of the property, but it is a
    real behaviour change — this is the commit to scrutinise on hardware.
  - Base dtsi passive trips use `hysteresis = <2000>`, so CPU/GPU throttling
    states will hold ~2 C longer than before. Watch for any effect on sustained
    clocks under load.
- `drivers/hwmon/pwm-fan.c` — `pwm_fan_temp_to_state()` had **zero** hysteresis,
  a pure ascending threshold scan. Added `PWM_FAN_TEMP_HYST` (2000 mC), applied
  only to trips at or below the currently occupied state.
  - **Does not affect the Rock 5B/5B+/5T at all** — they use cooling-maps, not
    `rockchip,temp-trips`. This one is for the boards listed under "Remaining
    boards" that do use that path. Untestable on a Rock 5B.
  - Interaction to be aware of: `rockchip_system_monitor_thermal_update()`
    already suppresses falling-temperature notifications until the drop exceeds
    2000 mC from `last_temp`. The two compose, giving an effective falling
    deadband of roughly 2-4 C. Tune `PWM_FAN_TEMP_HYST` with that in mind.
- `drivers/thermal/gov_power_allocator.c` — `get_governor_trips()` no longer
  `break`s on a hot/critical trip, so trips described after one are still seen.
  Genuine order-dependence bug, but **changes nothing on these boards** since
  they do not use this governor. Pure robustness; safe to split out or drop.

### Verification done

- All three DTBs compile (`cpp` + `dtc`, exit 0) and were confirmed byte-identical
  to their post-PWM-fix state after the revert.
- All three .c files pass `gcc -fsyntax-only -std=gnu11` with kernel include
  flags, exit 0.
- **No real kernel build, no boot, no hardware measurement.** No aarch64
  cross-compiler was available and objtool needs libelf, which was not
  installed. Syntax and type checking only.

## Open questions

1. **Does the step_wise change actually settle the fan?** This is the one to
   test first. Load the SoC, sweep the temperature across 60 C and 65 C, and
   watch `/sys/class/hwmon/hwmon*/pwm1` for hunting. Expect the state to hold
   until the temperature falls a full 5 C below the trip that set it.
2. **Watch CPU/GPU throttling for regressions** from the same change — see the
   blast-radius note above. If it causes trouble, the alternative is to scope
   the hysteresis to active trips only, or to make it opt-in per zone.
3. **The pwm-fan hysteresis value (2000 mC) is a guess** and cannot be validated
   on a Rock 5B. Test on a board that uses `rockchip,temp-trips` — Rock 5A is
   the easiest.
4. **Verify the governor on every target before assuming**, with
   `cat /sys/class/thermal/thermal_zone0/policy`. Do not trust
   `rockchip_linux_defconfig` — that mistake cost a full round trip on this
   branch. Boards that *do* default to power_allocator cannot drive a pwm-fan
   from cooling-maps at all, and would need either `rockchip,temp-trips` or
   power-actor ops added to pwm-fan.
5. **Rejected for now:** forcing `policy=step_wise` via udev (userspace, not a
   kernel fix), and teaching `thermal_of` to parse a `governor` DT property
   (correct in general, largest blast radius).

## Remaining boards

These lists were built for the power_allocator theory. Reread them with the
correction above in mind: on a step_wise system the cooling-map boards are
**not** broken, they simply inherit the `gov_step_wise.c` hysteresis fix for
free, with no DT change needed. Only boards that genuinely default to
power_allocator have the "fan pinned at max" problem — check each target's
`policy` before assuming either way.

Cooling-map boards (get the step_wise fix automatically; no DT work):

- `rk3588-rock-5-itx.dts`, `rk3588s-rock-5c.dts`, `rk3588s-radxa-e52c.dts`,
  `rk3588s-radxa-e54c.dts`, `rk3588s-radxa-nx5-io.dts`, `rk3576-rock-4d.dts`,
  `rk3576-radxa-cm4-io.dts`

`rockchip,temp-trips` boards (get the pwm-fan hysteresis fix instead; these are
the only ones where that change is observable):
`rk3588s-rock-5a.dts`, `rk3588s-radxa-cm5-io.dts`,
`rk3588-orangepi-5-ultra.dts`, `rk3566-orangepi-3b-v2.1.dts`,
`rk3576-recomputer-rk3576-devkit.dts`.

Worth checking on the cooling-map boards: several use `THERMAL_NO_LIMIT` for
both bounds on a single trip, which under step_wise ramps the fan one level per
poll all the way to maximum once that trip is crossed, rather than tracking
temperature proportionally. Functional, but crude — the 5B's explicit per-trip
`<&fan0 N N+1>` ladder is the better pattern.

Still-audible PWM carriers found in the earlier sweep, unfixed:

- 16.667 kHz (60000 ns): `rk3576-rock-4d.dts`, `rk3576-radxa-cm4-io.dts`,
  `rk3576-recomputer-rk3576-devkit.dts`
- 4 kHz (250000 ns): `rk3588-blade3-v101-linux.dts`, `rk3588s-lubancat-4.dts`
- **50 Hz** (20000000 ns): `rk3588-orangepi-5-ultra.dts`,
  `rk3566-orangepi-3b-v2.1.dts` — audible flutter, worst in the tree
- ~40 boards sit at 20 kHz (50000 ns). Right at the edge of audibility and the
  de-facto convention in this tree; deliberately left alone.
