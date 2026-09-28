# pi-data

Raw data from a Raspberry Pi 5 dual-gas logger (CO₂ and CH₄) used in headbox indirect calorimetry validation work at the University of Nebraska-Lincoln.

This repository is updated automatically by the Pi. **Do not edit files under `data/` by hand.** The Pi's local copy is the master record, and it overwrites this folder on every sync.

Maintainer: Ishrat Jandu

---

## System overview

| Component | Role | Interface | Actual sample period |
|---|---|---|---|
| Raspberry Pi 5 (`amiraspi`, Debian 13) | Logger host | | |
| Sensirion SCD30 | CO₂ (NDIR), temperature, RH | I²C bus 1, address `0x61`, 50 kHz | ~2.13 s (configured 2 s) |
| NGM-1 pellistor module | CH₄ (catalytic bead) | USB CDC serial (`/dev/serial/by-id/usb-ATMEL_ASF_CDC_Virtual_Com-if00`) | ~21.0 s (nominal 20 s) |

Both sample periods are set by each sensor's internal clock, not by the Pi. **All timing in analysis must come from the timestamp columns, never from the row index.**

### SCD30 wiring

| SCD30 pin | Pi 5 header pin |
|---|---|
| VDD | Pin 1 (3.3 V) |
| GND | Pin 6 |
| SDA | Pin 3 (GPIO2) |
| SCL | Pin 5 (GPIO3) |
| SEL, RDY, PWM | Not connected |

`/boot/firmware/config.txt` includes `dtparam=i2c_arm_baudrate=50000` for SCD30 clock-stretching compatibility.

### Sensor configuration

- SCD30 automatic self-calibration (ASC): **forced OFF** on every start.
- SCD30 ambient pressure compensation: off.
- SCD30 temperature offset: 0.00 °C (uncalibrated).
- The NGM-1's own `C:` percentage output is inverted relative to true CH₄ and is **not** used as the analytical variable. Use `adc_code`.

---

## Repository layout

```
data/
  run_YYYYmmdd_HHMMSS/
    scd30.csv     CO2 / T / RH, one row per SCD30 sample
    ngm1.csv      CH4 pellistor, one row per NGM-1 line
    health.csv    Pi health snapshot every 60 s
    events.csv    starts, stops, errors, reconnects, clock steps
```

A new `run_` folder is created each time the logger starts (boot, crash restart, or manual restart). A gap between runs is therefore visible in the folder list and in `events.csv`, never silently stitched over.

**Folder names use the Pi's clock at the moment the logger started.** Before the RTC battery was installed, a cold boot without network could produce a wrong folder name (see [Known issues](#known-issues)). The `events.csv` start line and the `monotonic_s` column are authoritative.

---

## File formats

Every data file carries three time columns, all captured together at the moment the row is written:

| Column | Meaning |
|---|---|
| `timestamp_iso` | Local wall-clock time with UTC offset, millisecond resolution (e.g. `2026-09-26T17:00:44.177-05:00`) |
| `unix_s` | Same instant as Unix time (s) |
| `monotonic_s` | Seconds since Pi boot. Never jumps, even when the wall clock is corrected. Use this to detect and repair clock steps. |

### `scd30.csv`

| Column | Unit | Notes |
|---|---|---|
| `seq` | | Per-run sequence number, starts at 1 |
| `co2_ppm` | ppm | |
| `temp_c` | °C | Includes sensor self-heating and heat from the Pi; reads above ambient |
| `rh_pct` | % | Referenced to the sensor's own (warm) temperature |
| `qc_flag` | | `range` if any value is outside physical limits, otherwise empty |

The first SCD30 sample after each start is discarded by the logger (stale buffer value) and does not appear in the file.

### `ngm1.csv`

| Column | Unit | Notes |
|---|---|---|
| `seq` | | Per-run sequence number |
| `adc_code` | ADC counts | **Analytical variable.** Rests near -700 in clean air and moves toward zero as CH₄ rises |
| `uo_mv` | mV | Bridge output voltage |
| `module_temp_c` | °C | Module temperature (integer) |
| `status` | | Module status code, `0` = normal |
| `module_c_pct_unused` | % | Module's own concentration estimate. Inverted, not used |
| `parsed` | 0/1 | `1` if the line matched the expected format |
| `dup` | 0/1 | `1` if identical to the previous line within 15 s (test-button reprint) |
| `raw_line` | | Exact text received from the module, always kept |

For analysis, filter to `parsed == 1 and dup == 0`. One unparsed partial line at the start of each run is normal: the logger opens the port mid-line.

### `health.csv` (every 60 s)

| Column | Meaning | Healthy value |
|---|---|---|
| `cpu_temp_c` | Pi SoC die temperature. **Not** air temperature | Plateau below ~70 °C |
| `throttled` | Firmware power and thermal flags (hex bitmask) | `0x0` in every row |
| `disk_free_mb` | Free storage | Stable |
| `ntp_synced` | Clock synced to network time | `yes` when online |
| `wall_minus_mono_s` | Wall clock minus monotonic clock | Constant (ms-level drift) |
| `clock_step_s` | Filled only when the wall clock jumped by more than 0.5 s | Empty |
| `scd_ok`, `scd_err`, `scd_reinits` | Cumulative SCD30 counts | `scd_ok` rising ~28/min, errors 0 |
| `ngm_ok`, `ngm_unparsed`, `ngm_reconnects` | Cumulative NGM-1 counts | `ngm_ok` rising ~3/min, reconnects 0 |

`throttled` bit meanings:

| Value | Meaning |
|---|---|
| `0x1` / `0x10000` | Undervoltage now / has occurred since boot |
| `0x2` / `0x20000` | Frequency capped now / has occurred |
| `0x4` / `0x40000` | Throttled now / has occurred |
| `0x8` / `0x80000` | Soft temperature limit now / has occurred |

Any nonzero value during a run should be reported.

### `events.csv`

| Column | Meaning |
|---|---|
| `level` | `INFO`, `WARN`, or `ERROR` |
| `source` | `main`, `scd30`, `ngm1`, `clock`, or `health` |
| `message` | Free text, including start arguments, sensor settings at init, errors, and clock steps |

---

## Loading the data (Python)

```python
import pandas as pd
from pathlib import Path

run = Path("data/run_20260926_170642")

scd = pd.read_csv(run / "scd30.csv", parse_dates=["timestamp_iso"])
ngm = pd.read_csv(run / "ngm1.csv", parse_dates=["timestamp_iso"])
ngm = ngm[(ngm.parsed == 1) & (ngm.dup == 0)]

# 1-minute bins on the time axis (never on row count)
scd_1min = scd.set_index("timestamp_iso")["co2_ppm"].resample("1min").mean()
ngm_1min = ngm.set_index("timestamp_iso")["adc_code"].resample("1min").mean()

# Measured sample periods
print("SCD30 median period (s):", scd.unix_s.diff().median())
print("NGM-1 median period (s):", ngm.unix_s.diff().median())
```

### Repairing a clock step

If `events.csv` contains `wall clock stepped +X s relative to monotonic`, rows written **before** that event carry wall-clock times that are off by `X` seconds. Correct them from the monotonic column:

```python
step_s = 89.082           # from events.csv
step_at_mono = 65.127     # monotonic_s of the clock-step event
before = scd.monotonic_s < step_at_mono
scd.loc[before, "unix_s"] += step_s
```

---

## How this repository is updated

- `gaslogger.service` (systemd) runs the logger from boot, independent of network, and restarts it within 5 s after any crash.
- `gasdata-sync.timer` runs `sync_to_github.sh` 3 minutes after boot and then every 10 minutes. It copies `~/data` into a local clone, commits, and pushes over SSH (port 443) with a repository-scoped deploy key.
- If the Pi is offline, commits queue locally and are pushed on the next successful run.
- If the local clone is corrupted (for example, by power loss during a commit), the script moves it aside, re-clones from GitHub, and re-copies all data. Nothing is lost, because `~/data` on the Pi is always the master copy.
- Git on the Pi is configured with `core.fsync all`.

**Data on GitHub lags the Pi by at most ~10 minutes while online.**

---

## Known issues

| Date | Issue | Effect on data | Status |
|---|---|---|---|
| 2026-09-26 | Undervoltage on non-official power supplies (laptop USB, third-party 5 V / 5 A adapter). Brownout reboot at 21:03 | Several short runs on 2026-09-26 ended by power events, not by the experiment | Resolved: official 27 W supply (5.1 V, 5 A PD). Verified > 5.1 V under full load, `throttled=0x0` |
| 2026-09-26 | No RTC battery. After the 21:03 reboot the clock started 89 s slow until NTP corrected it | `run_20260926_210305`: rows before the clock-step event are 89.082 s early (correctable, see above) | Open until RTC battery installed |
| 2026-09-28 | No RTC battery. After a cold boot the clock restored Saturday's time | `run_20260926_211307` was actually recorded on **2026-09-28** starting ~11:34 CDT; its folder name is wrong | Open until RTC battery installed |
| 2026-09-26 | Power loss corrupted the Pi's local git clone | No data lost; sync paused until repaired on 2026-09-28 | Resolved: self-repair added to sync script |

Runs recorded on 2026-09-26 and 2026-09-28 are bench bring-up and power-supply tests, not experimental data.

---

## Pre-deployment checklist

- [x] SCD30 on I²C, ASC off, 0 read errors
- [x] NGM-1 on USB serial, parser verified on real output
- [x] Logger runs as a service, survives crash and reboot
- [x] Automatic GitHub sync with offline queueing and self-repair
- [x] Official 27 W power supply verified under load
- [ ] ML-1220 RTC battery installed (timestamps survive power loss without network)
- [ ] Persistent systemd journal enabled
- [ ] 23+ h closed-enclosure soak test with `throttled=0x0` throughout
- [ ] Clock offset vs. calorimeter recorded at start and end of each run
- [ ] System response lag measured by step test
