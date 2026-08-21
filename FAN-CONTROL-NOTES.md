# Rockchip PWM fan control — working notes

Scratch notes for the `rock-5b-fan-pwm-frequency` branch. Not for upstream
submission as-is. Delete before sending patches.

Target: Armbian **rk35xx / vendor** — `armbian/linux-rockchip` at
`rk-6.1-rkr5.1`, built with `config/kernel/linux-rk35xx-vendor.config` and
patched from `patch/kernel/rk35xx-vendor-6.1/`. Everything below is about that
tree. The `current` and `edge` branches are a different kernel entirely
(`LINUXFAMILY=rockchip64`, mainline 6.18 / 7.1) — see "Scope: vendor only".

## Status

| Area | State |
|---|---|
| PWM carrier frequency (coil whine) | **Done**, committed — 5B, 5B+, 5T |
| Nine-level cooling ladder on 5B | **Done**, committed — confirmed correct, see below |
| Hysteresis was dead code | **Done**, committed — `gov_step_wise.c` |
| pwm-fan vendor path had no hysteresis | **Done**, committed — affects other boards |
| Verification against Armbian's build | **Done** — see that section |
| Real kernel build / boot test | **Not started** — now unblocked, see "Open questions" |
| Other affected boards | **Not started** — see "Remaining boards" |

## The governor is step_wise — confirmed, with the mechanism

The Rock 5B on the Armbian vendor kernel runs **step_wise**. Confirmed on the
target and traced back through Armbian's build to the reason why:

- `config/kernel/linux-rk35xx-vendor.config` contains **no**
  `CONFIG_THERMAL_DEFAULT_GOV_*` line at all, and no
  `CONFIG_THERMAL_GOV_STEP_WISE` line either.
- `drivers/thermal/Kconfig` declares the choice `default
  THERMAL_DEFAULT_GOV_STEP_WISE`, and that symbol `select`s
  `THERMAL_GOV_STEP_WISE`.
- So `olddefconfig` fills in step_wise, which pulls the governor in. The built
  image agrees: `/boot/config-6.1.115-vendor-rk35xx` has
  `CONFIG_THERMAL_DEFAULT_GOV_STEP_WISE=y` and `CONFIG_THERMAL_GOV_STEP_WISE=y`.

Under step_wise the cooling-map route works correctly, so the DT stays as it
was and the fan fix is in the governor plus two DT value corrections.

**Do not reason from `arch/arm64/configs/rockchip_linux_defconfig`.** Armbian
never uses it. An earlier pass on this branch read line 283 of that file,
concluded the board used a different governor, decided the cooling-maps were
inert, and converted the fan to `rockchip,temp-trips`. That work was committed
and then **reverted** (see the revert commit on this branch). The premise was
false and it cost a full round trip. Always confirm on the target:

```sh
cat /sys/class/thermal/thermal_zone0/policy       # soc-thermal: step_wise here
cat /sys/class/thermal/thermal_zone0/available_policies
```

This generalises: every Armbian rk35xx-vendor board shares that one config
file, so **they are all step_wise**. No per-board governor check is needed
within this family.

One fragility to keep in mind: it is step_wise *by omission*, not by intent.
Anyone adding an explicit `CONFIG_THERMAL_DEFAULT_GOV_*` to
`linux-rk35xx-vendor.config` — or a `savedefconfig` round-trip that imports one
from `rockchip_linux_defconfig` — silently flips the premise and re-breaks the
fan. Worth a comment in that config if these patches are upstreamed to Armbian.

### Why cooling-maps, not `rockchip,temp-trips`

The reverted conversion is not worth redoing. Under step_wise the cooling-maps
work, they honour the DT `hysteresis` property, they are tunable without a
driver change, and they are upstreamable — none of which is true of the
Rockchip-specific `rockchip,temp-trips` bypass. That bypass also *replaces*
rather than complements the cooling-map route: `pwm_fan_probe()` takes it first
and returns before registering the cooling device.

The findings from that pass that remain true and are worth keeping:

- The board's trip nodes are named `trip-point@N`, the base dtsi's are
  `trip-point-N`. Different names, so they **merge** rather than override.
  The compiled DTB has 11 trips in DT order: `[0]` 75 C passive,
  `[1]` 85 C passive, `[2]` 115 C critical, `[3..10]` the eight board active
  trips. `thermal_of.c` does not sort them. Harmless under step_wise, which
  processes every trip index independently.
- `thermal_of.c` never parses a governor property (verified: no match in the
  file), so the governor cannot be selected from DT in this tree — only by
  kconfig default or by writing `/sys/class/thermal/thermal_zone0/policy`.

## Fan behaviour, before and after

Rock 5B under step_wise. "Before" is `rk-6.1-rkr5.1`, "after" is the tip of this
branch. Not measured on hardware — derived from the DT and the governor code.

### Duty vs. temperature (rising)

| SoC temp | Before: state -> duty | Before % | After: state -> duty | After % |
|---|---|---|---|---|
| < 45 C | 0 -> 0 | off | 0 -> 0 | off |
| 45-50 C | 1 -> 64 | 25% | 1 -> 128 | 50% |
| 50-55 C | 2 -> 128 | 50% | 2 -> 146 | 57% |
| 55-60 C | 3 -> 192 | 75% | 3 -> 164 | 64% |
| 60-65 C | **4 -> 255** | **100%** | 4 -> 182 | 71% |
| 65-70 C | 4 -> 255 *(map unbound)* | 100% | 5 -> 201 | 79% |
| 70-75 C | 4 -> 255 *(map unbound)* | 100% | 6 -> 219 | 86% |
| 75-80 C | 4 -> 255 *(map unbound)* | 100% | 7 -> 237 | 93% |
| >= 80 C | 4 -> 255 *(map unbound)* | 100% | 8 -> 255 | 100% |

The old `cooling-levels = <0 64 128 192 255>` gave `max_state = 4`, so the four
maps requesting states 5-8 (`<&fan0 4 5>` .. `<&fan0 7 8>`) were rejected by
`thermal_zone_bind_cooling_device()` with `-EINVAL`. Visible on an unpatched
kernel:

```sh
dmesg | grep "Failed to bind"    # 4 x soc-thermal with pwm-fan: -22
```

Note what the defect actually was: the fan was **not** under-cooling. It slammed
to 100% at 60 C and stayed there, so the unbound maps made no thermal
difference — there was nothing above full speed to reach. The problem was noise,
not temperature. The commit message for `dd3ccc4ac62a` describes it as the fan
being "unable to respond", which is not right; reword if that commit is ever
sent upstream.

### Falling behaviour — this is what the hysteresis fix changes

| | Before | After |
|---|---|---|
| Steps down at | the same temperature it stepped up at | 5 C below the trip that set it |
| duty 182 (entered at 60 C) | n/a, was already at 255 | holds until < 55 C |
| duty 164 (entered at 55 C) | dropped as soon as temp < 55 C | holds until < 50 C |
| At a boundary with sensor noise | flips one level per poll (1 s), audible hunting | stable, single step |

`gov_step_wise.c` computed `throttle` as a bare `tz->temperature >= trip_temp`
and never called `get_trip_hyst()`, so `hysteresis = <5000>` was inert and the
up-threshold and down-threshold were the same number. The 5 C trip spacing was
never the problem: a trip now disengages exactly where the one below it engages,
giving a clean monotone ladder.

### Everything else

| Behaviour | Before | After | Commit |
|---|---|---|---|
| PWM carrier | 16.667 kHz, audible whine at every intermediate duty (64/128/192) | 40 kHz, inaudible | `2b96fa19ff6c` |
| Whine at 0% / 100% | none (no switching either way) | none | - |
| First spin-up step | 64 (25%), below reliable start for many 2-wire fans; can stall and buzz | 128 (50%), starts cleanly | `dd3ccc4ac62a` |
| Usable speed steps | 4 (of 8 maps) | 8 | `dd3ccc4ac62a` |
| Ramp 60 -> 80 C | flat at 100% | 71 -> 79 -> 86 -> 93 -> 100% | `dd3ccc4ac62a` |

Net: it used to be silent below 45 C, whine through three coarse steps, hit full
blast at 60 C, and hunt between levels whenever the temperature sat near a trip.
It should now ramp in eight inaudible steps across 45-80 C and hold each one
until the SoC has genuinely cooled 5 C.

Not shown in the tables, and the thing to watch: the step_wise change also
affects CPU and GPU throttling on this board, since they share `soc_thermal` and
the base dtsi passive trips use `hysteresis = <2000>`. Those cooling states now
hold about 2 C longer too.

## What was verified, and how

### Hysteresis was dead code

`gov_step_wise.c` computed `throttle` as a bare `tz->temperature >= trip_temp`
and never called `get_trip_hyst`. The `hysteresis` DT property fed only
`__thermal_zone_set_trips()` (an IRQ window that `rockchip_thermal.c` discards —
`rockchip_thermal_set_trips()` accepts `low` and `high` but passes only `high`
to `set_alarm_temp`, verified) and netlink notifications.

Note: 5 C hysteresis on 5 C trip spacing is *fine* once hysteresis works. Trip N
disengages exactly where trip N-1 engages — a clean monotone ladder, no chatter,
no gap. The spacing was never the bug.

Traced through the live configuration to be sure. `thermal_cdev_update()` takes
the max target across instances, so at 62 C the engaged trips 45/50/55/60 give
targets converging on 4 (the upper of the 60 C map). Falling to 58 C, the 60 C
trip is inside its band and stays engaged, holding state 4. Only below 55 C does
it drop to 3, where the 55 C trip's band holds it. Monotone the whole way down.
Without the fix each boundary chatters instead.

**This is the primary fix for the Rock 5B**, not a side improvement.

### Mainline implements the same semantic — so this is a backport, not a submission

Checked against the cached mainline tree
(`build/cache/sources/linux-kernel-worktree/7.2__rockchip64__arm64`):

```
vendor 6.1     if (tz->temperature >= trip_temp)                 /* raw trip temp */
mainline 7.2   bool throttle = tz->temperature >= trip_threshold;
```

where mainline's `trip_threshold` is `td->threshold`, maintained in
`thermal_core.c`: `td->threshold = td->trip.temperature` when the trip is not
reached, and `td->trip.temperature - td->trip.hysteresis` once it is.
`struct thermal_trip_desc` carries the `int threshold` field for exactly this.

That is the same behaviour this branch implements — rise at the trip, fall one
hysteresis band below — placed in the core rather than the governor. Two
consequences:

1. The design is **validated**: mainline converged on the same semantic
   independently.
2. It is **not upstreamable to mainline**, which already has it. This is a
   vendor-6.1 backport. Say so in the commit message or a reviewer will bounce
   it as already-fixed.

### Why commit "list all nine fan cooling levels" was right

`thermal_zone_bind_cooling_device()` rejects a map with
`upper > cdev->max_state` (returns `-EINVAL`). With the original five
`cooling-levels`, `max_state` was 4, so map4 through map7
(`<&fan0 4 5>` .. `<&fan0 7 8>`) all failed to bind. Extending the ladder to
nine levels fixed real breakage, and the `max_state` reasoning in the commit
message is correct — only the "unable to respond" clause is wrong, see above.

## Verification against Armbian's build

Done against `../build` (armbian/build). Board mapping:
`config/boards/rock-5b.conf` -> `BOARDFAMILY=rockchip-rk3588`,
`KERNEL_TARGET="current,edge,vendor"`; the `vendor` case in
`config/sources/families/rockchip-rk3588.conf` gives
`KERNELBRANCH=rk-6.1-rkr5.1`, `KERNELPATCHDIR=rk35xx-vendor-6.1`,
`LINUXFAMILY=rk35xx`.

**No conflict with Armbian patching:**

- `patch/kernel/rk35xx-vendor-6.1/` holds exactly three patches — `001-hid-sony.patch`
  and two bluetooth ones — touching `drivers/bluetooth/hci_ldisc.c`,
  `drivers/hid/hid-sony.c`, `include/net/bluetooth/hci.h`,
  `net/bluetooth/hci_sync.c`. Zero overlap with anything on this branch.
- Its `0000.patching_config.yaml` declares `dts-directories: dt ->
  arch/arm64/boot/dts/rockchip`, but that patch dir has **no `dt/`
  subdirectory**, so no board DTS is copied over ours. (Only
  `rv1126-vendor-6.1` and `genio-1200-vendor` have one.)
- Decisive: every file this branch touches is **byte-identical** between
  pristine `upstream/rk-6.1-rkr5.1` and Armbian's real patched build tree at
  `cache/sources/linux-kernel-worktree/6.1__rk35xx__arm64` — the three board
  DTS, `gov_step_wise.c` and `pwm-fan.c`. `thermal_core.c` and
  `gov_power_allocator.c` were compared too, as cross-checks; also identical.

**Kernel config facts that matter:**

- `CONFIG_ROCKCHIP_SYSTEM_MONITOR=y` — so the `rockchip,temp-trips` path is
  live, and the `pwm-fan.c` hysteresis patch is not dead code on the boards
  that use it.
- `CONFIG_PWM_ROCKCHIP_ONESHOT` is **not** set, so `dclk_div = 1` in
  `rockchip_pwm_config_v1()`. This is what the 40 kHz duty arithmetic assumed.
- `CONFIG_SENSORS_PWM_FAN=m`, `CONFIG_PWM_ROCKCHIP=y`, `CONFIG_ROCKCHIP_THERMAL=y`.

**Board sweep re-derived** by parsing inside each `pwm-fan` node (not the first
`pwms` line in each file — that catches backlights and gives wrong answers):

| nodes | period | carrier |
|---|---|---|
| 1 | 5000 ns | 200 kHz |
| 11 | 10000 ns | 100 kHz |
| 3 | 25000 ns | 40 kHz — the three boards fixed on this branch |
| 7 | 40000 ns | 25 kHz |
| 43 | 50000 ns | 20 kHz |
| 3 | 60000 ns | **16.667 kHz** |
| 2 | 250000 ns | **4 kHz** |
| 2 | 20000000 ns | **50 Hz** |

All three patched DTBs compile (`cpp` + `dtc`, exit 0). 5B+ and 5T correctly
keep their five-level ladder — they use `THERMAL_NO_LIMIT` in their maps, so
they never had the bind failure.

## Scope: vendor only

Rock 5B declares `KERNEL_TARGET="current,edge,vendor"`. `current` (6.18) and
`edge` (7.1) resolve through `rockchip64_common.inc` to `LINUXFAMILY=rockchip64`
— the mainline kernel, a different source tree. There the fan node lives in
`rk3588-rock-5b-5bp-5t.dtsi`, shared across 5B/5B+/5T:

```dts
cooling-levels = <0 120 150 180 210 240 255>;   /* 7 levels, no bind bug */
fan-supply = <&vcc5v0_sys>;                     /* regulator; absent in vendor */
pwms = <&pwm1 0 50000 0>;                       /* 20 kHz, not 16.667 */
```

with only two fan trips (55 C and 65 C, `hysteresis = <2000>`). So **none of the
three fixes apply to current/edge**: no whine bug, no bind bug, and hysteresis
already handled in the core. The DT work does not follow you if you switch
branch. Whether 20 kHz is quiet enough on the mainline branch is untested.

## How to confirm on hardware

```sh
cat /sys/class/thermal/thermal_zone0/policy          # expect: step_wise
cat /sys/class/thermal/thermal_zone0/type            # soc-thermal
grep . /sys/class/thermal/cooling_device*/type
grep . /sys/class/thermal/cooling_device*/cur_state
cat /sys/class/hwmon/hwmon*/pwm1
dmesg | grep "Failed to bind"                        # pre-fix: 4 x -22
```

Then load the SoC and watch whether `pwm1` tracks the ladder in the table above,
and whether each step holds until the temperature has fallen a full 5 C.

Note: if `policy` reads `user_space`, something set it by hand — it is not the
boot default. Set it back with
`echo step_wise | sudo tee /sys/class/thermal/thermal_zone0/policy` before
testing, or nothing in the cooling-map path will run at all.

## Changes on this branch

All committed. Nothing is pushed, so any of it can still be reworded, split, or
dropped.

### DT

The three board DTs differ from their base only by the PWM period fix and, on
the 5B, the nine-level ladder. The `rockchip,temp-trips` conversion was
committed and reverted; see the revert commit.

### Drivers

- `drivers/thermal/gov_step_wise.c` — **the actual fix for the Rock 5B.** Made
  hysteresis real: added `thermal_trip_is_engaged()` as a latch, plus a branch
  that keeps throttling while the temperature sits inside
  `(trip_temp - hyst, trip_temp)` and the trip is already engaged. The rising
  edge still fires at the exact trip temperature; only the step back down is
  delayed. Matches mainline's semantic — see the backport note above.
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
- `drivers/thermal/gov_power_allocator.c` — **dropped from this series.** It
  was an unrelated robustness fix carried along from the reverted pass:
  `get_governor_trips()` no longer `break`s on a hot/critical trip, so trips
  described after one are still seen. A real order-dependence bug, but it
  changes nothing on these boards and does not belong here. Resurrect it as its
  own patch from the reflog if it is ever wanted.

### Verification done

- All three DTBs compile (`cpp` + `dtc`, exit 0) and were confirmed identical
  to their post-PWM-fix state after the revert.
- All three .c files pass `gcc -fsyntax-only -std=gnu11` with kernel include
  flags, exit 0.
- Cross-checked against Armbian's build — see that section.
- **No real kernel build, no boot, no hardware measurement.** See below: this is
  now unblocked.

## Open questions

1. **Build and boot it.** This is the next step and it is no longer blocked.
   The earlier note that no aarch64 cross-compiler was available was wrong —
   this host *is* aarch64 (`gcc -dumpmachine` -> `aarch64-linux-gnu`), so no
   cross-compiler is needed at all, and `libelf-dev`, `libssl-dev`, `bison`,
   `flex` and `bc` are all installed. A native build is possible now.
2. **Does the step_wise change actually settle the fan?** Load the SoC, sweep
   across 60 C and 65 C, watch `/sys/class/hwmon/hwmon*/pwm1` for hunting.
   Expect each state to hold until the temperature falls a full 5 C below the
   trip that set it.
3. **Watch CPU/GPU throttling for regressions** from the same change — see the
   blast-radius note above. If it causes trouble, the alternative is to scope
   the hysteresis to active trips only, or to make it opt-in per zone.
4. **The pwm-fan hysteresis value (2000 mC) is a guess** and cannot be validated
   on a Rock 5B. Test on a board that uses `rockchip,temp-trips` — Rock 5A is
   the easiest.
5. **The idle fan-off floor.** `cooling-levels[0] = 0` stops the fan below 45 C
   and restarts it at 45 C, dropping out again at 40 C. A board idling in the
   38-47 C range will cycle audibly. This is inherited stock behaviour, not
   something the branch introduced, but a non-zero floor is probably better in
   practice. Deliberately left for a later decision.
6. **Rejected for now:** forcing `policy=step_wise` via udev (userspace, not a
   kernel fix), and teaching `thermal_of` to parse a `governor` DT property
   (correct in general, largest blast radius).

## Remaining boards

All of these are step_wise on Armbian vendor — same config file, same default
(see the governor section). So the cooling-map boards are not broken; they
simply inherit the `gov_step_wise.c` hysteresis fix for free, with no DT change
needed.

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

Still-audible PWM carriers, unfixed (counts re-verified, see the sweep table):

- 16.667 kHz (60000 ns): `rk3576-rock-4d.dts`, `rk3576-radxa-cm4-io.dts`,
  `rk3576-recomputer-rk3576-devkit.dts`
- 4 kHz (250000 ns): `rk3588-blade3-v101-linux.dts`, `rk3588s-lubancat-4.dts`
- **50 Hz** (20000000 ns): `rk3588-orangepi-5-ultra.dts`,
  `rk3566-orangepi-3b-v2.1.dts` — audible flutter, worst in the tree
- 43 boards sit at 20 kHz (50000 ns). Right at the edge of audibility and the
  de-facto convention in this tree; deliberately left alone.
