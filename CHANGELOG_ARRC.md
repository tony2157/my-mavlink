# ARRC dialect changelog

Dialect commits on `ArduPilot-4.7-ARRC-SDR` that touch `message_definitions/v1.0/arrc.xml`
(`git log --oneline origin/ArduPilot-4.7-ARRC..ArduPilot-4.7-ARRC-SDR -- message_definitions/v1.0/arrc.xml`),
newest first, with the dialect hash after each (see `README_ARRC.md` for the rule).

## Branch `ArduPilot-4.7-ARRC-SDR-WFfilter` (2026-10-27): waveform filter

Extension fields only; hash unchanged, `0x7EFB3788` (`ARRC_SDR_MEAS_CONFIG` `crc_extra` 94,
`ARRC_SDR_MEAS` 21). Consumers can be bumped one at a time; the autopilot routes and logs the longer
messages without a rebuild (no message-length check in its parser, and its decode zero-fills or cuts).

- `ARRC_SDR_MEAS_CONFIG`: `wf_filter` (new enum `ARRC_WF_FILTER`: OFF, MONITOR, ENFORCE) and
  `wf_tol_db`. A GCS that does not send them leaves the filter off.
- `ARRC_SDR_MEAS`: `n_wf_rej` (mismatched detections since the previous message) and `wf_match_db`.
- `ARRC_MEAS_FLAGS_WF_MISMATCH` (2048) and `ARRC_MEAS_FLAGS_WF_FILTERED` (4096), which also reach the
  GCS in the autopilot's `ARRC_SENSOR_RAW.flags`.

## Unreleased (`ArduPilot-4.7-ARRC-SDR` head)

Documentation only; hash `0x7EFB3788`, every `crc_extra` unchanged; no consumer bump needed.

- `arrc: profile delete moves the slots above down one`: `ARRC_PROFILE_ACTION_DELETE` says the
  companion renumbers the slots above the deleted one, so the saved profiles stay contiguous.
- `arrc: add README_ARRC.md and CHANGELOG_ARRC.md with the pairing table`.
- `arrc: describe every enum and entry, cpu_temp 127, 16-char hostname`: a description on the 28
  enums and 117 entries that had none, from the companion's code; `ARRC_CC_STATUS.cpu_temp` 127 means
  no thermal sensor, `hostname` is cut to 16 characters.

## `0f4028b7` (2026-10-15): report the data disk size and the data service in ARRC_CC_STATUS

Extension fields `disk_total_mb`, `data_port`, `hostname` on `ARRC_CC_STATUS`, and the
`ARRC_CC_FLAGS_DISK_LOW` flag. Hash unchanged, `0x7EFB3788` (`ARRC_CC_STATUS` `crc_extra` 192).

## `20eeff0a` (2026-10-15): include common.xml so the dialect generates on its own

`<include>common.xml</include>`, so a plain `pip install` of pymavlink from this repository builds the
`arrc` dialect. No message change; hash `0x7EFB3788`.

## `9bbb6154` (2026-10-14): report the recording in progress in ARRC_SDR_STATUS

Extension fields `rec_seconds`, `rec_size_mb`, `rec_trigger` on `ARRC_SDR_STATUS` and the
`ARRC_REC_TRIGGER` enum. Hash unchanged, `0x7EFB3788`.

## `d3a673e7` (2026-10-14): add autopilot sensor and gimbal streams for the companion

New messages 47009-47014 (`ARRC_NAV_IMU`, `ARRC_NAV_MAG`, `ARRC_NAV_BARO`, `ARRC_NAV_GPS`,
`ARRC_NAV_ECEF`, `ARRC_GIMBAL_POINTING`), autopilot to companion at the autopilot's log rates, and the
enums `ARRC_NAV_IMU_FLAGS`, `ARRC_NAV_GPS_FLAGS`, `ARRC_GIMBAL_MODE`, `ARRC_GIMBAL_FLAGS`. Hash
`0x639FB45F` -> `0x7EFB3788`.

## `1499dbf3` (2026-10-14): add RFSonde SDR companion dialect

New `arrc.xml`, included by `ardupilotmega.xml`: messages 47000-47008 (status, config, measurement,
trace and catalogue), `MAV_CMD` 47000-47005, and the `ARRC_*` enums. `ARRC_SENSOR_RAW` (228) moves
here from `ardupilotmega.xml` with the extension fields `snr_db`, `flags`, `n_avg`; its `crc_extra`
(166) is unchanged. Hash `0x639FB45F`.

Before this branch: `ab4355ca` (2025-08-18, branch `ArduPilot-4.7-ARRC`) added `CASS_SENSOR_RAW`
(227) and the first `ARRC_SENSOR_RAW` (228) to `ardupilotmega.xml`.

## Pairing table

Which commit of each consumer goes with which dialect commit. "verified" means checked in the git
history of that repository (the `modules/mavlink` submodule SHA at the ArduPilot commit, or the
`pymavlink` pin diff in `pyproject.toml`); "per VERIFICATION.md" or "per the project pairing record" means taken from the project record
`RFSonde_Computer/docs/VERIFICATION.md` and not re-checked in git.

| Dialect commit | Hash | my-ardupilot (`ArduPilot-4.7-ARRC`) | RFSonde_Computer (`main`) | RFSonde_GUI (`Ludifly-SDR`) |
|---|---|---|---|---|
| `1499dbf3` | `0x639FB45F` | `40293793a2` moves the submodule, `c3c5257604` integrates the SDR (submodule = `1499dbf3`, verified) | first commits, from `8404624` on, with pymavlink pinned to `1499dbf3` (verified: `782ae67` moves the pin off it) | `b79d2e9` pins `1499dbf3` (verified), with the four Ludifly-SDR commits `11e6160`, `8423447`, `c2f60dc`, `3b761d4` |
| `d3a673e7` | `0x7EFB3788` | `3fb61a97f6` moves the submodule, `f5a5e76e1f` sends the streams (submodule = `d3a673e7`, verified) | `782ae67` pins `d3a673e7` (verified); `5649443` is the paired release state per the project pairing record (VERIFICATION.md names the dialect, ArduPilot and GUI commits of section 4f, not this one) | `4a9d0ee` pins `d3a673e7` (verified) |
| `9bbb6154` | `0x7EFB3788` | `5948896571` (submodule = `9bbb6154`, verified) | `6d94d14` pins `9bbb6154` (verified); release `dc5ae36` tested on the LattePanda per VERIFICATION.md | `87d792f` pins `9bbb6154` (verified) |
| `20eeff0a` + `0f4028b7` | `0x7EFB3788` | `9ba126ff5a` (submodule = `0f4028b7`, verified) | `d0767a8` pins `0f4028b7` (verified) | `9963fe0` pins `0f4028b7` (verified) |
| this branch head (documentation) | `0x7EFB3788` | no bump needed | no bump needed | no bump needed |

Later commits on the consumer branches (my-ardupilot `23ed82ed2f`, `24fcf4bf8a`, `938924ff2f`;
RFSonde_Computer `e6666cc`..`757ad51`; RFSonde_GUI `3045d0f`..`f2fb806`) keep the `0f4028b7` pin.
