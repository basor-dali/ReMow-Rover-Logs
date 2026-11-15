# Field session 2026-11-15 — run manifest and context for analysis

Written the same day, from the logs in this repo plus facts that are **not**
in any log file: which code commit each run used, and what changed on the
hardware between runs. Numbers were computed with ad-hoc scripts from the
nav CSVs and run logs; recompute before trusting a specific figure.

Code repo: `basor-dali/ReMow-Rover-Tests`. Nav CSV and run log share a
timestamp; the run log may be one second earlier (e.g. log `10-46-55`, CSV
`10-46-56`).

## Where things live in this repo

| what | path |
|---|---|
| Repeat run logs | `Repeat/Logs/readRecordedRun-YYYY-MM-DD_HH-MM-SS.log` |
| Navigation CSVs (one row per 5 Hz control epoch) | `Repeat/navigation_log/YYYY-MM-DD_HH-MM-SS_navigation_log.csv` |
| Recorded routes being replayed | `Record/Data/55_20260510-134004_GPSData.csv` (40.7 m, has corners), `Record/Data/66_20260830-114454_GPSData.csv` (37.2 m, one gentle loop, max 36° per vertex — **not** a steering test route) |
| Keypad menu logs (incl. background heading capture) | `selectMode/selectMode_YYYYMMDD-HHMMSS.log` |
| Static base coordinates used as ENU origin | `config.json` |

`rel_east` / `rel_north` in the nav CSV are the rover's **absolute** lat/lon
converted to ENU against `config.json` — not the dual-antenna baseline.

## Code state per run

| commit | in force for | key config |
|---|---|---|
| `099b0e8` (main, morning) | all runs 10:28 → 12:56 | `guidance_mode=gary_arrows`, `gary_arrow_lead_m=4.0` (the 3.0 m `path_lookahead` is pure_pursuit-only and unused), `desired_turn_radius_scale=0.5`, `off_track_abort_enabled=False`, RoboClaw Buffer flag `0` |
| `528effc` (PR #63) | run 14:04:12 | as above but `gary_arrow_lead_m=3.0`; new `gary_lead_nearest_vertex=False`; startup banner now logs the real arrow lead |
| `d6d09ae` (PR #64) | **no runs yet** | RoboClaw Buffer flag `0→1` (execute immediately — `0` was queuing every command, including emergency neutral); new command slew limiter `actuator_command_slew_counts_per_epoch=100`; 3 new nav columns (`command_slew_limited`, `command_left/right_before_slew`); `actuator_lag_counts` now logged on every row |

**Authoritative source for the commit per boot:** `selectMode/boot-pull.log`
— `safe_pull.sh` logs the pinned commit at every boot. Confirmed:

| boot (local time) | commit |
|---|---|
| 2026-11-08 13:55 | `60ed4e3` |
| 2026-11-08 14:50 → pull | `099b0e8` |
| 2026-11-15 10:24, 10:28, 10:32, 10:36, 10:48, 10:50, 11:04, 11:07, 11:10, 11:14, 12:05, 12:58 | `099b0e8` |
| 2026-11-15 13:56 | `528effc` |
| 2026-11-15 ~15:00 (manual pull, no boot) | `d6d09ae` |

## systemd journal extracts (`journal/`)

The Pi keeps a persistent journal (756 MB, 44 boots). The run output is echoed
there as `rc.local[PID]:` lines, and it is the **only** place the Python
processes' stderr goes. Extracted here, with NetworkManager / sshd / DHCP
lines removed (this repo is public):

- `journal/boots.txt` — every retained boot, with its first line and the
  last non-INFO lines before it ended. This is the power-loss record.
- `journal/kernel.txt` — kernel log across all boots (USB enumerations,
  resets, storage errors, under-voltage). As of 2026-11-15: no resets, no
  storage errors, no under-voltage anywhere.
- `journal/rc_local_non_info.txt` — every `rc.local` line that is not an
  INFO echo: WARNING/ERROR lines, Python exceptions and stderr. Zero
  `Traceback` lines exist in the entire journal; the 36 Python exception
  lines are all one old `gps_heading_capture_thread` AttributeError
  (`constants.gps_serial_port`), from before that constant existed.
- `journal/journal_filtered.txt` — rc.local + kernel + systemd(pid 1) +
  journald lines, INFO echoes removed. Grep-able context for anything above.

Clock caveat applies to all of these: boots without Wi-Fi start on a restored
(wrong) clock until NTP corrects it.

## Hardware / setup timeline (not in any log)

1. **Morning (all runs to 12:56):** static base ZED-F9P had NMEA+UBX output
   enabled on its unused UART1/I2C/SPI ports (TX-buffer starvation risk);
   base SiK 915 MHz radio antenna lying **horizontal** on a table behind a
   cardboard box. Rover/moving-base antenna vertical → cross-polarised link.
2. **~12:15:** base config written: output disabled on UART1/I2C/SPI, GSV off
   USB; base power-cycled. 13-minute buffer monitor afterwards: peak 10 %,
   no alloc errors. **Did not fix the GNSS dropouts** (12:55 run was the
   worst of the day) — buffers were not the cause.
3. **~13:50:** base SiK antenna stood **vertical**, box removed.
4. **14:04 run:** zero GNSS waits, zero carrier transitions, zero position
   jumps. The link problem was the antenna orientation.
5. **~14:00:** Pi deliberately power-cycled by the operator (one clean boot).
6. **~15:00:** PR #64 deployed. No run on it yet.

Also: `journalctl --list-boots` shows **12 unclean boots between 10:24 and
12:57** — the Pi lost power repeatedly; three of them mid-run (10:49:05,
11:08:34, 11:10:48). No undervoltage was ever logged, so it is an abrupt
cut (connector/switch), not a sagging supply. Zero-byte run logs and
NUL-padded selectMode logs are this. The Pi restores a bogus clock at boot
(no retained RTC) and NTP corrects it later, so **timestamps from a boot
without Wi-Fi can be wrong**; within one file they stay consistent.

## Runs (route 55 unless stated)

GNSS waits = epochs held neutral for invalid GNSS; transitions = `Carrier
solution: fixed <-> float` lines; jumps = epoch-to-epoch position moves
> 0.5 m. **Every jump today landed within 4 s of a carrier transition, 5 of 7
in the same second as a `float -> fixed` re-fix** — RTK ambiguity
re-resolution, not USB (dmesg clean all day) and not base proximity.

| run | outcome | epochs | GNSS waits | transitions | jumps | cross-track med / max (m) | notes |
|---|---|---|---|---|---|---|---|
| 10:28:07 | **0-byte log** | — | — | — | — | — | power lost at start |
| 10:31:23 | completed 40.7 m | 208 | 74 | 4 | 1 (2.72) | 0.69 / 3.12 | |
| 10:46:56 | completed | 189 | **0** | **0** | **0** | **0.30 / 2.58** | best tracking of the day; GPS simply behaved |
| 10:48:18 | aborted early | — | — | — | — | — | 4.7 KB log |
| 10:49:16 | **0-byte log** | — | — | — | — | — | power lost at start |
| 11:06:47 | aborted early | — | — | — | — | — | 11 KB log |
| 11:08:09 | **0-byte log** | — | — | — | — | — | power lost while polling for RTK fix |
| 11:09:47 | **stopped at 28.6 m by power loss** | 121 | 31 | 6 | **5** (2.50, 2.47, 2.23, 1.99, 1.05) | 0.71 / **5.01** | **50 % of epochs at an actuator rail**; worst run; not a software abort — boot ended 11:10:48 |
| 11:12:31 | aborted early | — | — | — | — | — | |
| 11:13:36 | 16 rows | 16 | 14 | 3 | 1 (1.06) | 1.30 / 2.48 | |
| 12:55:35 | never moved | 9 | **125** | 3 | — | — | after base-config fix; worst GNSS of the day |
| 12:56:22 | completed | 182 | 41 | 2 | 1 (1.36) | 0.56 / 1.20 | tightest max of the day |
| **14:04:12 (route 66, `528effc`)** | **left the route at 28.6 m, operator stop** | 173 | **0** | **0** | **0** | 0.20 tracking → 4.0+ | see below |

### Run 14:04:12 — why it failed with perfect GPS

Tracked at ≤0.3 m to ep 140. At ep 106 Gary's law hit the near-vertex guard
(`GEO_INFEASIBLE` → straight) with 37° heading error, so the straight-heading
cap cut speed 1.10 → 0.50 m/s: a 462-count actuator step. From ep 109 the
actuator-lag speed limiter fired on 25 epochs; `desired_speed` flipped
0.50 / 1.09 on alternate epochs; actuator commands swung ~450 counts per
200 ms (p90 swing 756 vs 160–209 on the path-55 runs). The actuators were
in a LEFT-turn configuration while RIGHT was commanded (5 % of epochs
opposite-sign). The rover turned left into a right-hand bend, cross-track
grew 0.9 → 4.0 m in 5 s, the target-advance rules (`A<B`, `along>0`) had no
case to fire (rover 4 m beside and behind the arrow), and the law's
`DTR = s/tan(|HC|/2)` collapsed the radius as HC passed 90°.

Root cause found afterwards: every RoboClaw position command was sent with
Buffer = **0**, which the RoboClaw manual (Rev 5) defines as *queue behind
the move in progress*; only `1` executes immediately. With commands changing
every epoch and moves taking 3–5 epochs the actuators executed stale
commands (ep 151–160: command held at 1146 for ten epochs, read-back
oscillated 748→827→747→647→704→814→710→636). No delay+slew plant model fits
that trace, and the closed-loop simulator (non-queued plant) could not
reproduce the oscillation. Fixed in `d6d09ae`, **unvalidated in the field**.

Measured actuator step response that day (15 clean steps, 4 runs): ~1 epoch
dead time, then 19 / 86 / 77 / 22 / 5 counts per epoch; p99 of all
epoch-to-epoch motion 110 counts. Measured **with the queue in place** —
repeat after the fix.

## Things to know when reading the nav CSVs

- Pre-`d6d09ae` rows: `actuator_lag_counts` is blank on ~29 % of rows. That
  is a `"{:d}"`-on-float logging bug, **not** missing encoder read-back
  (`actual_*_actuator_position` is present on 100 % of rows).
- `desired_*_actuator_position` is what was sent (post every limiter). The
  limiter's own input is `actuator_lag_command_*_before_limit`; the shadow
  reference is `actuator_lag_reference_*`, logged **after** that epoch's
  advance (so the value the limiter used on row *i* is row *i−1*'s).
- Effective lookahead in gary_arrows ≠ `gary_arrow_lead_m`: the lead snaps to
  the first vertex at-or-past the lead (overshoots by up to one spacing), and
  the chord from the mower is longer again because `A<B` advances the reached
  arrow up to one spacing early. On route 66 at lead 3.0: arc lead med 3.93,
  chord med 4.62 (tracking epochs). `gary_lead_actual_m` holds the arc lead.
- The real mower runs faster than its calibration: GNRMC speed med 1.32 m/s
  (peaks 1.9) with actuators at ~430 counts, where the calibration says
  1.19 m/s is the full-forward rail. Treat `calculated_*_wheel_speed` as
  ~15 % low. Earlier circle-fit work found the machine over-turns (achieves
  ~0.73× the commanded radius).
- RoboClaw position PID read live: M1/M2 Kp 5.5, Ki 0, Kd 0, limits 420–1450.

## Open as of end of day

1. Field-validate `d6d09ae`: on the next run expect `actuator_lag_speed_limited`
   ≈ 0 (was 25), speed flips ≈ 0 (was 22), command swing p90 < 200 (was 756),
   `command_slew_limited` rare once moving; `ReadBuffers(0x80)` must stay 0/128
   while driving.
2. Pi power loss — hardware; wiggle-test the supply connectors.
3. Effective lead / trailing gap (above).
4. Speed and turn-rate calibration are both off; the simulator is a
   regression check only until the plant is re-measured.
5. Gary's law at |HC| > 90° collapses the radius; the feasibility rule floors
   it at 0.94 m. Not changed — Gary's call.
