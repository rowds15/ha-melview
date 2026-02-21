# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Home Assistant custom integration for AU/NZ Mitsubishi Electric air conditioners and Lossnay ERV units with Wi-Fi adapters (MAC-568IF-E, MAC-578IF-E, MAC-588IF-E). Distributed via HACS. Cloud-polling with optional local LAN command delivery.

## Validation Commands

There is no local test suite. Validation is done via GitHub Actions:

- **hassfest**: `docker run --rm -v $(pwd)/custom_components:/github/workspace/custom_components ghcr.io/home-assistant/hassfest`
- **HACS validation**: runs via GitHub Actions on push/PR
- **Linting/formatting**: `ruff check custom_components/` and `ruff format custom_components/`

Test by running the integration in a live Home Assistant instance.

## Architecture

All code lives in `custom_components/melview/`.

### Data Flow

```
MelView Cloud API (api.melview.net)
        |
MelViewAuthentication  (login, cookie/session management)
        |
MelView  (fetches device list)
        |
MelViewDevice  (per-unit: state cache with 30s lease, capabilities cache, command sending)
        |
MelViewCoordinator  (DataUpdateCoordinator, 30s poll, proxies attribute access to MelViewDevice)
        |
MelViewBaseEntity  (shared device_info, coordinator wiring)
        |
    Platforms:
    - climate.py  → MelViewClimate (A/C units only, not ERV)
    - fan.py      → MelViewLossnayFan (ERV units only)
    - sensor.py   → temperature sensors (current temp for all; outdoor/supply/exhaust/efficiency for ERV)
    - switch.py   → MelViewZoneSwitch (ducted zone control)
```

### Device Type Branching

Device type is detected via `get_unit_type()` from capabilities. If `"ERV"`: fan entity + ERV sensors are created. Otherwise: climate entity. Zone switches are created for any device with zones.

### Local Command Flow

Commands go cloud-first (to get a local command key), then the key is pushed to the device over LAN at `http://{device_ip}/smart`. Internet is always required.

### API Layer (`melview.py`)

This is the core API client — not a Home Assistant file. It handles authentication, device state, capabilities caching, and command dispatch. The 30-second state lease prevents sending commands against stale data. On 401, it auto-re-authenticates and retries (with a flag to prevent loops).

## Key Conventions

- **Version**: calendar versioning `YYYY.M.V` in `manifest.json`. Bump version on every change.
- **Commit style**: short lowercase messages (e.g., `fix dry mode temp range`, `bump version`, `add device cleanup`)
- **Formatting**: `ruff check` and `ruff format` before committing
- **Entity naming**: `has_entity_name = True` (modern HA pattern), unique IDs based on device ID
- **Async throughout**: all HA-facing methods are `async_*`, API calls use `aiohttp`
- **Capitalization**: "MelView" (capital M, capital V) consistently in class names
- **Config flow**: supports initial setup, reconfigure (password update), and reauth
- **Options**: `CONF_LOCAL` (local LAN commands, default True), `CONF_SENSOR` (temperature sensor entities, default True)

## API Constants

- Base URL: `https://api.melview.net/api/`
- API version: 3, App version: `"6.5.2090"`
- Headers include a Safari user-agent string (defined in `const.py`)

## Temperature Handling

- Supports 0.5°C steps when enabled in the MelView app
- Min/max temperatures are dynamic per HVAC mode (COOL, HEAT, DRY uses COOL range)
- Temperature ranges come from device capabilities
