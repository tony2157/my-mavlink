# The ARRC dialect (`arrc.xml`)

`message_definitions/v1.0/arrc.xml` is the MAVLink dialect of the RFSonde system of the
Advanced Radar Research Center (ARRC): the onboard-SDR companion computer (component 191,
`MAV_COMP_ID_ONBOARD_COMPUTER`), the autopilot (ArduPilot, branch `ArduPilot-4.7-ARRC`) and the
ground station (Ludifly). The autopilot routes the companion's messages to and from the GCS and logs
the measurement feed.

- `arrc.xml` includes `common.xml` (since `20eeff0a`), so it generates on its own.
- `ardupilotmega.xml` includes `arrc.xml`, so the `ardupilotmega` dialect that all three consumers
  use carries every ARRC message, enum and command.
- `<dialect>2</dialect>`; MAVLink 2 is required (ids above 255, extension fields).

## Id blocks

| Block | Where defined | Content |
|---|---|---|
| Messages 47000-47019 | `arrc.xml` | Reserved for ARRC. Defined: 47000 `ARRC_CC_STATUS`, 47001 `ARRC_SDR_STATUS`, 47002 `ARRC_SDR_CONFIG`, 47003 `ARRC_SDR_MEAS_CONFIG`, 47004 `ARRC_SDR_MEAS`, 47005 `ARRC_SDR_TRACE`, 47006 `ARRC_SCRIPT_INFO`, 47007 `ARRC_WAVEFORM_INFO`, 47008 `ARRC_PORT_INFO`, 47009 `ARRC_NAV_IMU`, 47010 `ARRC_NAV_MAG`, 47011 `ARRC_NAV_BARO`, 47012 `ARRC_NAV_GPS`, 47013 `ARRC_NAV_ECEF`, 47014 `ARRC_GIMBAL_POINTING`. 47015-47019 are free. |
| `MAV_CMD` 47000-47009 | `arrc.xml` (extends the `MAV_CMD` enum of `common.xml`) | Reserved for ARRC. Defined: 47000 `MAV_CMD_ARRC_SDR_CONTROL`, 47001 `MAV_CMD_ARRC_TX_ARM`, 47002 `MAV_CMD_ARRC_SCRIPT`, 47003 `MAV_CMD_ARRC_RECORD`, 47004 `MAV_CMD_ARRC_PROFILE`, 47005 `MAV_CMD_ARRC_WAVEFORM_GEN`. 47006-47009 are free. |
| Message 228 `ARRC_SENSOR_RAW` | `arrc.xml` | The RF measurement downlink, autopilot to GCS. It was added to `ardupilotmega.xml` in `ab4355ca` and moved to `arrc.xml` in `1499dbf3`, which added the extension fields `snr_db`, `flags` and `n_avg`; its `crc_extra` (166) did not change. |
| Message 227 `CASS_SENSOR_RAW` | `ardupilotmega.xml` | The BLISS/CASS raw sensor message (LB5900 power as `app_datatype` 4 in Ludifly). Not part of the ARRC dialect and not covered by the dialect hash. |

The enum blocks (`ARRC_*`) carry a description on every enum and every entry, derived from the
companion's code: what each value makes the system do. The `MAV_CMD` block has no enum-level
description, because it extends the enum of `common.xml`.

## Dialect hash

The companion reports in `ARRC_CC_STATUS.dialect_hash` the hash of the dialect it was built with, and
Ludifly compares it with the hash of its own pymavlink build; a difference means the two sides decode
the ARRC messages differently, and Ludifly refuses the SDR controls. Both compute it the same way
(`rfsonde_cc/protocol.py` `dialect_hash()` in RFSonde_Computer, `local_dialect_hash()` in
`src/ludifly/core/sdr_telemetry.py` in RFSonde_GUI):

```python
import zlib
from pymavlink.dialects.v20 import ardupilotmega as mavlink

parts = []
for mid in [228] + list(range(47000, 47020)):          # in id order
    cls = mavlink.mavlink_map.get(mid)
    if cls is not None:                                  # free ids contribute nothing
        parts.append(f"{mid}:{cls.msgname}:{cls.crc_extra};")
dialect_hash = zlib.crc32("".join(parts).encode()) & 0xFFFFFFFF
```

The hash covers the message ids, names and `crc_extra` values, so it changes when a message is added
or removed or when any non-extension field changes, and it does not change when fields are added
after `<extensions/>`, when descriptions change, or when enums change. Values so far:

| Dialect commit | Hash | Messages |
|---|---|---|
| `1499dbf3` | `0x639FB45F` | 228, 47000-47008 |
| `d3a673e7` and later (`9bbb6154`, `20eeff0a`, `0f4028b7`, this branch head) | `0x7EFB3788` | 228, 47000-47014 |

## Generating the dialect

The Python dialect, as pymavlink builds it on install:

```sh
cd my-mavlink
python3 -m pymavlink.tools.mavgen --lang=Python --wire-protocol=2.0 \
    -o /tmp/arrc.py message_definitions/v1.0/arrc.xml
```

`pymavlink/` in this repository is a git submodule; when it is not checked out, run the same tool from
another pymavlink checkout (for example `PYTHONPATH=<ardupilot>/modules/mavlink python3
<ardupilot>/modules/mavlink/pymavlink/tools/mavgen.py ...`). The generator needs `lxml` and `future`.
Generating `ardupilotmega.xml` the same way gives the dialect the consumers use. Checking a change:
generate before and after, and compare the `crc_extra` of every message and the hash above.

## How the three consumers pin it

All three pin the same commit of this branch (`ArduPilot-4.7-ARRC-SDR`), by full SHA:

| Consumer | Pin | Bump |
|---|---|---|
| my-ardupilot, branch `ArduPilot-4.7-ARRC` | The `modules/mavlink` submodule (`.gitmodules`: url `https://github.com/tony2157/my-mavlink`, branch `ArduPilot-4.7-ARRC-SDR`). The firmware is generated from `ardupilotmega.xml` at the pinned commit. | `git -C modules/mavlink fetch && git -C modules/mavlink checkout <sha>`, then commit the submodule (`modules: update mavlink ...`) and rebuild the boards. |
| RFSonde_Computer, branch `main` | `pyproject.toml`: `pymavlink @ git+https://github.com/tony2157/my-mavlink.git@<sha>#subdirectory=pymavlink` (twice: `dependencies` and the `dev` extra). `packaging/build.sh` installs the same commit (`MAVLINK_COMMIT`, or `--mavlink-src` for a local checkout); `docs/DEPLOYMENT.md` names it. `rfsonde_engine/enums.py` mirrors the enum values and `tests/test_enums.py` checks them against the installed dialect. | Change the SHA in all three places (`packaging: pin pymavlink to my-mavlink <sha>`), `pip install --force-reinstall --no-deps` the pymavlink (pip keeps an installed pymavlink of the same version when only the pin moves), run the tests, rebuild the release. |
| RFSonde_GUI, branch `Ludifly-SDR` | `pyproject.toml`: the same `pymavlink @ git+...@<sha>#subdirectory=pymavlink` line. `src/ludifly/core/sdr_telemetry.py` holds the enum tables and the tests check them against the generated dialect. | Change the SHA (`pymavlink: pin my-mavlink <sha>`), reinstall pymavlink or build with `-Clean`, run the tests. |

Rules:

- A change that alters any `crc_extra` (a new message, a new or changed non-extension field) changes
  the dialect hash: bump the three consumers together, or the companion and Ludifly refuse each other.
- Extension fields (after `<extensions/>`) keep the `crc_extra` and the hash: an updated side reads the
  new fields, the others zero-fill or ignore them, so the consumers can be bumped one at a time.
- Documentation-only changes (descriptions) change nothing on the wire; the pins stay valid.
- Never change an existing id, name, type, order or enum value; add at the end of the free ranges.

`CHANGELOG_ARRC.md` lists every dialect commit on this branch and the consumer commits paired with it.
