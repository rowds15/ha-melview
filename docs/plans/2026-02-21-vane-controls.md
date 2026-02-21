# Vane Controls Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add vertical and horizontal vane (air direction) controls to the MelView Home Assistant integration using native climate swing modes and optional select entities.

**Architecture:** Extend the existing MelViewDevice API layer to parse vane capability flags and handle AV/AH commands, add swing mode support to the MelViewClimate entity, create optional MelViewVaneSelect entities for standalone dashboard use, all capability-gated and excluded for ERV units.

**Tech Stack:** Home Assistant 2024.12+ (ClimateEntityFeature.SWING_HORIZONTAL_MODE), Python asyncio, aiohttp, MelView cloud API

---

## Task 1: Add vane position constants to melview.py

**Files:**
- Modify: `custom_components/melview/melview.py:1-36`

**Step 1: Add vane position mapping constants**

Add these constants after the existing `LOSSNAY_PRESETS` dictionary (after line 36):

```python
VERTICAL_VANE = {
    0: "Auto",
    1: "1",
    2: "2",
    3: "3",
    4: "4",
    5: "5",
    7: "Swing",
}

HORIZONTAL_VANE = {
    0: "Auto",
    1: "1",
    2: "2",
    3: "3",
    4: "4",
    5: "5",
    8: "Split",
    12: "Swing",
}
```

**Step 2: Verify constants are correct**

Check that:
- VERTICAL_VANE has 7 entries (0, 1-5, 7)
- HORIZONTAL_VANE has 8 entries (0, 1-5, 8, 12)
- All values are strings for Home Assistant UI display

**Step 3: Commit**

```bash
git add custom_components/melview/melview.py
git commit -m "add vane position constants

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 2: Parse vane capability flags in MelViewDevice

**Files:**
- Modify: `custom_components/melview/melview.py:134-156`

**Step 1: Add vane capability attributes to __init__**

In the `MelViewDevice.__init__()` method (around line 150), after `self.temp_ranges = {}`, add:

```python
self.has_vertical_vane = False
self.has_horizontal_vane = False
self.has_swing = False
self.has_auto_vane = False
self.vertical_vane_keyed = {}
self.horizontal_vane_keyed = {}
```

**Step 2: Parse capability flags in async_refresh_device_caps**

In `async_refresh_device_caps()`, after the `halfdeg` check (around line 196), add:

```python
# Vane capabilities
self.has_vertical_vane = self._caps.get("hasairdir", 0) == 1
self.has_horizontal_vane = self._caps.get("hasairdirh", 0) == 1
self.has_swing = self._caps.get("hasswing", 0) == 1
self.has_auto_vane = self._caps.get("hasairauto", 0) == 1

# Create reverse lookups for vane positions
if self.has_vertical_vane:
    self.vertical_vane_keyed = {v: k for k, v in VERTICAL_VANE.items()}
if self.has_horizontal_vane:
    self.horizontal_vane_keyed = {v: k for k, v in HORIZONTAL_VANE.items()}
```

**Step 3: Verify parsing logic**

Check that:
- Capability flags default to False if not present in API response
- Reverse lookups only created if capability exists
- Pattern matches existing `fan_keyed` implementation

**Step 4: Commit**

```bash
git add custom_components/melview/melview.py
git commit -m "parse vane capability flags from API

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 3: Add vane control methods to MelViewDevice

**Files:**
- Modify: `custom_components/melview/melview.py:508-516` (after async_set_lossnay_preset)

**Step 1: Add async_set_vertical_vane method**

After the `async_set_lossnay_preset()` method, add:

```python
async def async_set_vertical_vane(self, position_label: str) -> bool:
    """Set vertical vane position by label."""
    if not await self.async_is_power_on():
        if not await self.async_power_on():
            return False

    code = self.vertical_vane_keyed.get(position_label)
    if code is None:
        _LOGGER.error("Vertical vane position %s not supported", position_label)
        return False

    return await self.async_send_command(f"AV{code}")

async def async_set_horizontal_vane(self, position_label: str) -> bool:
    """Set horizontal vane position by label."""
    if not await self.async_is_power_on():
        if not await self.async_power_on():
            return False

    code = self.horizontal_vane_keyed.get(position_label)
    if code is None:
        _LOGGER.error("Horizontal vane position %s not supported", position_label)
        return False

    return await self.async_send_command(f"AH{code}")
```

**Step 2: Verify method signatures**

Check that:
- Both methods follow the same pattern as `async_set_speed()` and `async_set_lossnay_preset()`
- Both require unit to be powered on (auto-power on if needed)
- Both validate position label exists in keyed dictionary
- Both use `async_send_command()` with the correct command prefix (AV/AH)
- Both return bool for success/failure

**Step 3: Commit**

```bash
git add custom_components/melview/melview.py
git commit -m "add vane control methods to API layer

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 4: Add swing mode support to MelViewClimate

**Files:**
- Modify: `custom_components/melview/climate.py:1-8` (imports)
- Modify: `custom_components/melview/climate.py:31-46` (__init__)
- Modify: `custom_components/melview/climate.py:47-57` (async_added_to_hass)

**Step 1: Add SWING_HORIZONTAL_MODE import**

Update the imports at the top of climate.py (around line 5-9):

```python
from homeassistant.components.climate.const import (
    ClimateEntityFeature,
    HVACAction,
    HVACMode,
)
```

Add import for vane constants:

```python
from .melview import MODE, VERTICAL_VANE, HORIZONTAL_VANE
```

**Step 2: Initialize vane flags in __init__**

In `MelViewClimate.__init__()`, after `self._target_step = 1.0`, add:

```python
self._has_vertical_vane = False
self._has_horizontal_vane = False
```

**Step 3: Set vane flags in async_added_to_hass**

In `async_added_to_hass()`, after the halfdeg check, add:

```python
# Check vane capabilities
self._has_vertical_vane = getattr(self._device, "has_vertical_vane", False)
self._has_horizontal_vane = getattr(self._device, "has_horizontal_vane", False)
```

**Step 4: Verify initialization**

Check that:
- Flags default to False in __init__
- Flags set from device capabilities after async_added_to_hass
- Pattern matches existing halfdeg capability check

**Step 5: Commit**

```bash
git add custom_components/melview/climate.py
git commit -m "initialize vane capability flags in climate entity

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 5: Add swing mode feature flags to MelViewClimate

**Files:**
- Modify: `custom_components/melview/climate.py:58-68` (supported_features property)

**Step 1: Add swing mode to supported features**

Update the `supported_features` property:

```python
@property
def supported_features(self):
    """Let HASS know feature support"""
    features = (
        ClimateEntityFeature.FAN_MODE
        | ClimateEntityFeature.TURN_ON
        | ClimateEntityFeature.TURN_OFF
    )
    if self.hvac_mode in (HVACMode.AUTO, HVACMode.HEAT, HVACMode.COOL, HVACMode.DRY):
        features |= ClimateEntityFeature.TARGET_TEMPERATURE
    if self._has_vertical_vane:
        features |= ClimateEntityFeature.SWING_MODE
    if self._has_horizontal_vane:
        features |= ClimateEntityFeature.SWING_HORIZONTAL_MODE
    return features
```

**Step 2: Verify feature flags**

Check that:
- SWING_MODE added only if has_vertical_vane
- SWING_HORIZONTAL_MODE added only if has_horizontal_vane
- Existing features (FAN_MODE, TURN_ON/OFF, TARGET_TEMPERATURE) unchanged

**Step 3: Commit**

```bash
git add custom_components/melview/climate.py
git commit -m "add swing mode feature flags to climate

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 6: Implement vertical swing mode properties and method

**Files:**
- Modify: `custom_components/melview/climate.py:217` (after async_turn_off, before async_setup_entry)

**Step 1: Add swing_mode property**

After the `async_turn_off()` method, add:

```python
@property
def swing_mode(self) -> str | None:
    """Return current vertical vane position."""
    if not self._has_vertical_vane:
        return None
    code = self.coordinator.data.get("airdir")
    return VERTICAL_VANE.get(code)

@property
def swing_modes(self) -> list[str] | None:
    """Return available vertical vane positions."""
    if not self._has_vertical_vane:
        return None
    return list(VERTICAL_VANE.values())

async def async_set_swing_mode(self, swing_mode: str) -> None:
    """Set vertical vane position."""
    _LOGGER.debug("Set vertical vane: %s", swing_mode)
    if await self._device.async_set_vertical_vane(swing_mode):
        await self.coordinator.async_refresh()
```

**Step 2: Verify swing mode implementation**

Check that:
- swing_mode returns None if capability not supported
- swing_mode reads from coordinator.data["airdir"]
- swing_modes returns list of all VERTICAL_VANE values
- async_set_swing_mode calls device method and refreshes coordinator
- Pattern matches existing fan_mode implementation

**Step 3: Commit**

```bash
git add custom_components/melview/climate.py
git commit -m "implement vertical vane swing mode

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 7: Implement horizontal swing mode properties and method

**Files:**
- Modify: `custom_components/melview/climate.py:217` (after vertical swing methods)

**Step 1: Add swing_horizontal_mode properties**

After the vertical swing mode methods, add:

```python
@property
def swing_horizontal_mode(self) -> str | None:
    """Return current horizontal vane position."""
    if not self._has_horizontal_vane:
        return None
    code = self.coordinator.data.get("airdirh")
    return HORIZONTAL_VANE.get(code)

@property
def swing_horizontal_modes(self) -> list[str] | None:
    """Return available horizontal vane positions."""
    if not self._has_horizontal_vane:
        return None
    return list(HORIZONTAL_VANE.values())

async def async_set_swing_horizontal_mode(self, swing_horizontal_mode: str) -> None:
    """Set horizontal vane position."""
    _LOGGER.debug("Set horizontal vane: %s", swing_horizontal_mode)
    if await self._device.async_set_horizontal_vane(swing_horizontal_mode):
        await self.coordinator.async_refresh()
```

**Step 2: Verify horizontal swing implementation**

Check that:
- swing_horizontal_mode reads from coordinator.data["airdirh"]
- swing_horizontal_modes returns HORIZONTAL_VANE values (including "Split")
- Pattern matches vertical swing implementation
- All properties/methods follow HA climate entity conventions

**Step 3: Commit**

```bash
git add custom_components/melview/climate.py
git commit -m "implement horizontal vane swing mode

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 8: Create select.py platform with vertical vane entity

**Files:**
- Create: `custom_components/melview/select.py`

**Step 1: Create select.py with imports and vertical vane entity**

Create the new file:

```python
"""MelView select entities."""

from __future__ import annotations

import logging

from homeassistant.components.select import SelectEntity
from homeassistant.config_entries import ConfigEntry
from homeassistant.core import HomeAssistant
from homeassistant.helpers.entity_platform import AddEntitiesCallback

from .const import CONF_SENSOR
from .coordinator import MelViewCoordinator
from .entity import MelViewBaseEntity
from .melview import VERTICAL_VANE, HORIZONTAL_VANE

_LOGGER = logging.getLogger(__name__)


class MelViewVerticalVaneSelect(MelViewBaseEntity, SelectEntity):
    """Vertical vane position select entity."""

    _attr_has_entity_name = True
    _attr_translation_key = "vertical_vane"

    def __init__(self, coordinator: MelViewCoordinator):
        """Initialize vertical vane select."""
        super().__init__(coordinator, coordinator.device)
        self._device = coordinator.device
        self._attr_unique_id = f"{coordinator.device.get_id()}_vertical_vane"

    @property
    def current_option(self) -> str | None:
        """Return current vertical vane position."""
        code = self.coordinator.data.get("airdir")
        return VERTICAL_VANE.get(code)

    @property
    def options(self) -> list[str]:
        """Return available vertical vane positions."""
        return list(VERTICAL_VANE.values())

    async def async_select_option(self, option: str) -> None:
        """Set vertical vane position."""
        _LOGGER.debug("Select vertical vane: %s", option)
        if await self._device.async_set_vertical_vane(option):
            await self.coordinator.async_refresh()
```

**Step 2: Verify vertical vane select entity**

Check that:
- Inherits from MelViewBaseEntity and SelectEntity
- Uses translation_key for localization
- Unique ID includes device ID and "_vertical_vane"
- current_option reads from coordinator.data["airdir"]
- options returns all VERTICAL_VANE values
- async_select_option calls device method and refreshes

**Step 3: Commit**

```bash
git add custom_components/melview/select.py
git commit -m "create select platform with vertical vane entity

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 9: Add horizontal vane select entity

**Files:**
- Modify: `custom_components/melview/select.py:45` (after MelViewVerticalVaneSelect)

**Step 1: Add horizontal vane select class**

After the `MelViewVerticalVaneSelect` class, add:

```python
class MelViewHorizontalVaneSelect(MelViewBaseEntity, SelectEntity):
    """Horizontal vane position select entity."""

    _attr_has_entity_name = True
    _attr_translation_key = "horizontal_vane"

    def __init__(self, coordinator: MelViewCoordinator):
        """Initialize horizontal vane select."""
        super().__init__(coordinator, coordinator.device)
        self._device = coordinator.device
        self._attr_unique_id = f"{coordinator.device.get_id()}_horizontal_vane"

    @property
    def current_option(self) -> str | None:
        """Return current horizontal vane position."""
        code = self.coordinator.data.get("airdirh")
        return HORIZONTAL_VANE.get(code)

    @property
    def options(self) -> list[str]:
        """Return available horizontal vane positions."""
        return list(HORIZONTAL_VANE.values())

    async def async_select_option(self, option: str) -> None:
        """Set horizontal vane position."""
        _LOGGER.debug("Select horizontal vane: %s", option)
        if await self._device.async_set_horizontal_vane(option):
            await self.coordinator.async_refresh()
```

**Step 2: Verify horizontal vane select**

Check that:
- Pattern matches vertical vane select
- Unique ID uses "_horizontal_vane" suffix
- Reads from coordinator.data["airdirh"]
- Returns HORIZONTAL_VANE values (includes "Split")

**Step 3: Commit**

```bash
git add custom_components/melview/select.py
git commit -m "add horizontal vane select entity

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 10: Add select platform setup function

**Files:**
- Modify: `custom_components/melview/select.py:90` (after horizontal vane class)

**Step 1: Add async_setup_entry function**

After the entity classes, add:

```python
async def async_setup_entry(
    hass: HomeAssistant,
    entry: ConfigEntry,
    async_add_entities: AddEntitiesCallback,
) -> None:
    """Set up MelView select entities."""
    coordinators: list[MelViewCoordinator] = entry.runtime_data
    entities = []

    # Only create select entities if sensor option enabled
    if entry.options.get(CONF_SENSOR, True):
        for coordinator in coordinators:
            # Skip ERV units
            if coordinator.device.get_unit_type() == "ERV":
                continue

            # Add vertical vane select if supported
            if getattr(coordinator.device, "has_vertical_vane", False):
                entities.append(MelViewVerticalVaneSelect(coordinator))

            # Add horizontal vane select if supported
            if getattr(coordinator.device, "has_horizontal_vane", False):
                entities.append(MelViewHorizontalVaneSelect(coordinator))

    async_add_entities(entities, update_before_add=True)
```

**Step 2: Verify setup function**

Check that:
- Gated by CONF_SENSOR option (reuses existing sensor toggle)
- Skips ERV units (vanes are A/C only)
- Only creates entities if capability flags are True
- Uses update_before_add=True like other platforms
- Pattern matches sensor.py and fan.py setup

**Step 3: Commit**

```bash
git add custom_components/melview/select.py
git commit -m "add select platform setup with capability gating

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 11: Add SELECT platform to __init__.py

**Files:**
- Modify: `custom_components/melview/__init__.py:1-20` (imports and PLATFORMS constant)

**Step 1: Add Platform.SELECT to PLATFORMS list**

Find the PLATFORMS constant (should be around line 15-20) and add SELECT:

```python
PLATFORMS: list[Platform] = [
    Platform.CLIMATE,
    Platform.SWITCH,
    Platform.SENSOR,
    Platform.FAN,
    Platform.SELECT,
]
```

**Step 2: Verify platform list**

Check that:
- SELECT added after FAN
- All existing platforms (CLIMATE, SWITCH, SENSOR, FAN) remain
- No other changes to __init__.py needed

**Step 3: Commit**

```bash
git add custom_components/melview/__init__.py
git commit -m "add select platform to integration

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 12: Add translations for vane select entities

**Files:**
- Modify: `custom_components/melview/translations/en.json:1-50`

**Step 1: Add select entity translations**

Add a new "select" section under the "entity" key in translations/en.json. The file should have this structure:

```json
{
  "config": {
    ... existing config translations ...
  },
  "options": {
    ... existing options translations ...
  },
  "entity": {
    "select": {
      "vertical_vane": {
        "name": "Vertical vane"
      },
      "horizontal_vane": {
        "name": "Horizontal vane"
      }
    }
  }
}
```

**Step 2: Verify JSON syntax**

Check that:
- Valid JSON (no trailing commas)
- "select" key added under "entity"
- Translation keys match _attr_translation_key values
- Existing translations unchanged

**Step 3: Validate JSON**

Run: `python -m json.tool custom_components/melview/translations/en.json`
Expected: Valid JSON output, no errors

**Step 4: Commit**

```bash
git add custom_components/melview/translations/en.json
git commit -m "add vane select translations

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 13: Bump version in manifest.json

**Files:**
- Modify: `custom_components/melview/manifest.json:11`

**Step 1: Update version to 2026.2.1**

Change the version line in manifest.json:

```json
"version": "2026.2.1"
```

**Step 2: Verify manifest**

Check that:
- Version follows calendar versioning (YYYY.M.V)
- All other fields unchanged
- Valid JSON syntax

**Step 3: Validate JSON**

Run: `python -m json.tool custom_components/melview/manifest.json`
Expected: Valid JSON output

**Step 4: Commit**

```bash
git add custom_components/melview/manifest.json
git commit -m "bump version to 2026.2.1

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 14: Update CLAUDE.md with vane control architecture

**Files:**
- Modify: `CLAUDE.md:60-80` (after "Temperature Handling" section)

**Step 1: Add vane controls section**

After the "Temperature Handling" section in CLAUDE.md, add:

```markdown
## Vane Controls

- Vertical vane (up/down) via `airdir` field, command `AV{0-7}`
- Horizontal vane (left/right) via `airdirh` field, command `AH{0-12}`
- Capabilities: `hasairdir`, `hasairdirh`, `hasswing`, `hasairauto`
- Exposed as climate entity `swing_mode` / `swing_horizontal_mode` (HA 2024.12+)
- Optional select entities for dashboard use (gated by `CONF_SENSOR`)
- Not available on ERV units (A/C only)
- Quirk: horizontal swing (`AH12`) also activates vertical swing
```

**Step 2: Verify documentation**

Check that:
- Covers key vane control concepts
- Mentions both climate and select entity exposure
- Notes ERV exclusion
- Includes the horizontal swing quirk
- Concise (follows CLAUDE.md style)

**Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "document vane control architecture

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 15: Run ruff format and check

**Files:**
- Check all: `custom_components/melview/*.py`

**Step 1: Run ruff format**

Run: `ruff format custom_components/`
Expected: Files formatted, or "no changes" if already compliant

**Step 2: Run ruff check**

Run: `ruff check custom_components/`
Expected: No errors or warnings

**Step 3: Fix any ruff issues**

If ruff reports issues:
- Fix import ordering
- Fix line length violations
- Fix any other style issues

**Step 4: Commit any formatting changes**

```bash
git add custom_components/melview/
git commit -m "ruff format and check

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 16: Final verification

**Files:**
- Verify: All modified files

**Step 1: Check all files exist**

Verify these files were created/modified:
- [x] `custom_components/melview/melview.py` - API layer with vane methods
- [x] `custom_components/melview/climate.py` - Swing mode support
- [x] `custom_components/melview/select.py` - New platform created
- [x] `custom_components/melview/__init__.py` - SELECT platform added
- [x] `custom_components/melview/translations/en.json` - Vane translations
- [x] `custom_components/melview/manifest.json` - Version bumped
- [x] `CLAUDE.md` - Architecture documented

**Step 2: Review git log**

Run: `git log --oneline -15`
Expected: 15 commits covering all tasks

**Step 3: Check git status**

Run: `git status`
Expected: Clean working tree (all changes committed)

**Step 4: Push to remote (if ready)**

Run: `git push origin master`
Expected: Pushes all commits to GitHub

---

## Post-Implementation Testing

These steps should be done in a live Home Assistant environment:

1. **Install integration**: Copy to custom_components and restart HA
2. **Check logs**: Enable debug logging for `custom_components.melview`
3. **Verify capabilities**: Check logs show `hasairdir`/`hasairdirh` values
4. **Test climate card**: Verify swing mode controls appear if supported
5. **Test select entities**: Verify vane select entities created (if CONF_SENSOR enabled)
6. **Test commands**: Set vane positions and verify `AV`/`AH` commands sent
7. **Test ERV exclusion**: Verify ERV units don't get vane entities
8. **Test units without vanes**: Verify no errors if capabilities = 0

## Success Criteria

- [ ] All 16 tasks completed
- [ ] All commits pushed to master
- [ ] Code passes ruff check/format
- [ ] Vertical vane works via climate swing_mode
- [ ] Horizontal vane works via climate swing_horizontal_mode
- [ ] Select entities created when capabilities present
- [ ] ERV units excluded from vane controls
- [ ] Existing climate features unaffected
- [ ] Version bumped to 2026.2.1
- [ ] CLAUDE.md updated with vane architecture
