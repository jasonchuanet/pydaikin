![PyPI - License](https://img.shields.io/pypi/l/pydaikin?color=green)
![PyPI - Downloads](https://img.shields.io/pypi/dw/pydaikin?label=downloads&logo=pypi&logoColor=white)
![PyPI - Version](https://img.shields.io/pypi/v/pydaikin?label=version&logo=pypi&logoColor=white)

[![Test Python package](https://github.com/fredrike/pydaikin/actions/workflows/pytest.yml/badge.svg)](https://github.com/fredrike/pydaikin/actions/workflows/pytest.yml)
[![Pylint](https://github.com/fredrike/pydaikin/actions/workflows/pylint.yml/badge.svg)](https://github.com/fredrike/pydaikin/actions/workflows/pylint.yml)
[![codecov](https://codecov.io/github/fredrike/pydaikin/graph/badge.svg?token=DFEYF4L0J2)](https://codecov.io/github/fredrike/pydaikin)
![GitHub Pull Requests](https://img.shields.io/github/issues-pr/fredrike/pydaikin?logo=github)
![GitHub Issues](https://img.shields.io/github/issues/fredrike/pydaikin?logo=github)

# PyDaikin

PyDaikin is a Python library for controlling Daikin air conditioners. It provides both a standalone command-line interface and a Python API for integrating Daikin AC control into your applications.

## Supported Devices

The following Daikin WiFi modules are currently supported:

* **BRP069Axx/BRP069Bxx/BRP069B4x/BRP072Axx** - Standard WiFi adapters
* **BRP15B61 (AirBase)** - Uses a similar protocol to BRP069Axx
* **BRP072B/Cxx** - Requires HTTPS access and an authentication key
* **BRP084** - Devices with firmware version 2.8.0 (uses a different API structure)
* **SKYFi** - Uses a different protocol and requires a password

## Quick Start

Here's a simple example for connecting to a Daikin air conditioner:

```python
import asyncio
import logging

import aiohttp
from pydaikin.daikin_base import Appliance
from pydaikin.factory import DaikinFactory

HOST = "10.1.1.21"

logging.basicConfig(level=logging.INFO)
_LOGGER = logging.getLogger(__name__)

async def main():
    async with await DaikinFactory(HOST) as device:
        await device.update_status()
        device.show_sensors()

if __name__ == "__main__":
    asyncio.run(main())
```

The `DaikinFactory` automatically detects your device type and firmware version, creating the appropriate device instance.

## Firmware Version 2.8.0 Support

Firmware version 2.8.0 introduces a different API structure compared to earlier versions. PyDaikin automatically detects and handles this firmware version.

### API structure

Unlike the older CGI-based adapters (BRP069 family), firmware 2.8.0 exposes a single JSON endpoint, `POST /dsiot/multireq`:

* **Status reads** are batch requests containing four `op: 2` entries (indoor unit, outdoor unit, weekly energy and adapter info).
* **Writes** are `op: 3` entries that walk the `dgc_status` tree down to the parameter to change.

Full request/response examples are in [`docs/sample_requests/brp084/`](docs/sample_requests/brp084/README.md).

### Power semantics

The behaviour of `device.set(...)` was validated against the unit's behaviour (see `tests/test_daikin_brp084.py`):

* `device.set({})` turns the unit **on** (the Home Assistant toggle-switch path)
* `device.set({'mode': 'off'})` turns the unit **off**
* `device.set({'mode': 'cool'})` turns the unit **on** and switches mode
* `device.set({'stemp': 25.0})` sets the temperature only; a powered-off unit stays off

### Supported features

* **Comfort airflow, Econo, Outdoor-quiet and Powerful toggles** - Powerful and the trio (Comfort/Econo/Outdoor-quiet) are mutually exclusive, matching the remote's last-button-pressed-wins behaviour
* **Discrete vertical vane control** - `off`, `down` (floor) and `swing`
* **Swing** on the vertical and horizontal axes, per mode
* **Sensors** - indoor/outdoor temperature, humidity, per-head working state, indoor-unit Delta-D load command, shared outdoor-unit instantaneous power, decoded indoor/outdoor model strings and adapter firmware/API version
* **Energy data** - daily runtime and weekly consumption
* **Sub-zero outdoor temperatures** - sensor values are decoded as little-endian signed hex, so negative readings are handled correctly
* **Auto mode without a setpoint** - some units (e.g. the Urusara X, model `W-SRTA322F`) only expose cool/hot setpoints and report no auto setpoint (`e_3001/p_1D`); the target temperature then degrades to `--` instead of failing `init()` (issue [#72](https://github.com/fredrike/pydaikin/issues/72))

**Confirmed working with:**

* FTKM20YVMA
* FTXM46WVMA
* FTXV80WVMA
* FTXA25C2V1BW
* FTXA50C2V1BW

If you have a device with firmware 2.8.0 that is not working correctly, please open an issue with your device model and debug logs.

### BRP084 runtime telemetry

BRP084 devices expose two useful read-only indoor-unit values:

* `is_working` is decoded from indoor `e_3003/p_02` and indicates whether that
  head is actively working, independently of its configured HVAC mode.
* `idu_delta_d_command` is decoded from indoor `e_3003/p_2A`. It is exposed
  under the Skyport-derived field name because it tracks the indoor unit's
  load demand, but it is not presented as a settable demand-control limit.

`is_cooling` and `is_heating` combine `is_working` with the configured mode.
Outdoor `e_A005/p_01` is decoded using its DSIOT binary-step metadata as shared
instantaneous outdoor-unit power in watts.

The BRP084's dedicated Econo and Powerful controls are also exposed on the
Home Assistant climate entity as the standard `eco` and `boost` presets.
Selecting `none` disables the active preset; enabling either preset follows the
unit's built-in mutual-exclusion behavior.

### Testing the fork in Home Assistant with HACS

This fork includes a development override of Home Assistant's built-in
`daikin` integration under `custom_components/daikin`. It is based on Home
Assistant 2026.8.3 and installs this fork's `master` branch as its `pydaikin`
requirement.

1. Merge and push these changes to the fork's `master` branch.
2. In HACS, open **Integrations**, choose the three-dot menu, then
   **Custom repositories**. Add `https://github.com/jasonchuanet/pydaikin` as
   an **Integration** repository.
3. Install **Daikin AC (BRP084 runtime telemetry)** and restart Home Assistant.
4. Reload each Daikin device or restart once more if dependency installation
   occurred during startup.
5. Inspect the climate entity's `hvac_action`: a unit configured for cooling or
   heating now reports `idle` when `is_working` is false. The preset selector
   offers **Eco** and **Boost** on BRP084 units. Compare the diagnostic **Indoor
   unit Delta-D command** sensor against Skyport while changing setpoint, fan
   speed and humidity conditions.

To return to the built-in integration, remove this custom repository from HACS,
delete the custom integration when prompted, and restart Home Assistant. Existing
Daikin config entries and entity IDs are reused because the domain remains
`daikin`.

## Unsupported Devices

The following device and firmware combinations are not currently supported:

* BRP069C4x with firmware version 2.0.0

## About

PyDaikin was originally created by Yari Adan and is currently maintained by Fredrik Erlandsson.
