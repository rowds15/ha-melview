# Vane Controls Design

**Date:** 2026-02-21
**Feature:** Add vertical and horizontal vane (air direction) controls to MelView integration

## Overview

The MelView API already exposes vertical (`airdir`) and horizontal (`airdirh`) vane position data and accepts commands (`AV{value}`, `AH{value}`) to control them. The integration currently ignores these fields. This feature adds full vane control support.

## Requirements

1. Support both vertical (up/down) and horizontal (left/right) vane positioning
2. Expose via Home Assistant climate entity's native swing mode features (HA 2024.12+)
3. Also provide standalone select entities for dashboard/automation use
4. Capability-gated: only create entities for vanes the unit supports
5. Exclude ERV units (vane controls are A/C-only)

## API Details

### State Fields (from `unitcommand.aspx`)
- `airdir` — vertical vane position (integer)
- `airdirh` — horizontal vane position (integer)

### Capability Flags (from `unitcapabilities.aspx`)
- `hasairdir` — unit supports vertical vane (0 or 1)
- `hasairdirh` — unit supports horizontal vane (0 or 1)
- `hasswing` — unit supports swing mode (0 or 1)
- `hasairauto` — unit supports auto air direction (0 or 1)

### Command Codes
- **`AV{value}`** — set vertical vane position
  - `0` = Auto
  - `1`-`5` = Positions (1=top, 5=bottom)
  - `7` = Swing (oscillate)
- **`AH{value}`** — set horizontal vane position
  - `0` = Auto
  - `1`-`5` = Positions (1=leftmost, 5=rightmost)
  - `8` = Split (both sides)
  - `12` = Swing (oscillate)

### Known Quirk
Setting horizontal swing (`AH12`) also activates vertical swing. Horizontal swing must be disabled before vertical vane position can be changed.

## Architecture

### 1. API Layer (`melview.py`)

**Constants** (module level):
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

**`MelViewDevice` changes**:
- Store capability flags as boolean attributes during `async_refresh_device_caps()`:
  - `self.has_vertical_vane = self._caps.get("hasairdir", 0) == 1`
  - `self.has_horizontal_vane = self._caps.get("hasairdirh", 0) == 1`
  - `self.has_swing = self._caps.get("hasswing", 0) == 1`
  - `self.has_auto_vane = self._caps.get("hasairauto", 0) == 1`
- Create reverse lookups (like `fan_keyed`):
  - `self.vertical_vane_keyed = {v: k for k, v in VERTICAL_VANE.items()}`
  - `self.horizontal_vane_keyed = {v: k for k, v in HORIZONTAL_VANE.items()}`
- Add methods:
  ```python
  async def async_set_vertical_vane(self, position_label: str) -> bool:
      """Set vertical vane by label (e.g., 'Auto', '3', 'Swing')."""
      code = self.vertical_vane_keyed.get(position_label)
      if code is None:
          _LOGGER.error("Invalid vertical vane position: %s", position_label)
          return False
      return await self.async_send_command(f"AV{code}")

  async def async_set_horizontal_vane(self, position_label: str) -> bool:
      """Set horizontal vane by label."""
      code = self.horizontal_vane_keyed.get(position_label)
      if code is None:
          _LOGGER.error("Invalid horizontal vane position: %s", position_label)
          return False
      return await self.async_send_command(f"AH{code}")
  ```

**State access** (read from coordinator data):
- `airdir` and `airdirh` already returned by `unitcommand.aspx`, no API changes needed

### 2. Climate Entity (`climate.py`)

**`MelViewClimate` changes**:

**Initialization**:
- Initialize swing availability flags to `False` in `__init__`
- During `async_added_to_hass()`, after capabilities load:
  ```python
  self._has_vertical_vane = getattr(self._device, "has_vertical_vane", False)
  self._has_horizontal_vane = getattr(self._device, "has_horizontal_vane", False)
  ```

**Supported features**:
```python
@property
def supported_features(self):
    features = (existing features...)
    if self._has_vertical_vane:
        features |= ClimateEntityFeature.SWING_MODE
    if self._has_horizontal_vane:
        features |= ClimateEntityFeature.SWING_HORIZONTAL_MODE
    return features
```

**Vertical vane properties**:
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
    if await self._device.async_set_vertical_vane(swing_mode):
        await self.coordinator.async_refresh()
```

**Horizontal vane properties** (same pattern):
```python
@property
def swing_horizontal_mode(self) -> str | None:
    ...

@property
def swing_horizontal_modes(self) -> list[str] | None:
    ...

async def async_set_swing_horizontal_mode(self, swing_horizontal_mode: str) -> None:
    ...
```

### 3. Select Entities (`select.py` — new platform)

**Two entity classes**:

```python
class MelViewVerticalVaneSelect(MelViewBaseEntity, SelectEntity):
    """Vertical vane position select."""

    _attr_has_entity_name = True
    _attr_translation_key = "vertical_vane"

    @property
    def current_option(self) -> str | None:
        code = self.coordinator.data.get("airdir")
        return VERTICAL_VANE.get(code)

    @property
    def options(self) -> list[str]:
        return list(VERTICAL_VANE.values())

    async def async_select_option(self, option: str) -> None:
        if await self.coordinator.device.async_set_vertical_vane(option):
            await self.coordinator.async_refresh()
```

```python
class MelViewHorizontalVaneSelect(MelViewBaseEntity, SelectEntity):
    """Horizontal vane position select."""

    _attr_has_entity_name = True
    _attr_translation_key = "horizontal_vane"

    # Same pattern as vertical
```

**Platform setup**:
```python
async def async_setup_entry(hass, entry, async_add_entities):
    coordinators = entry.runtime_data
    entities = []

    # Only add if sensor config option enabled (reuse existing CONF_SENSOR)
    if entry.options.get(CONF_SENSOR, True):
        for coordinator in coordinators:
            # Skip ERV units
            if coordinator.device.get_unit_type() == "ERV":
                continue

            if getattr(coordinator.device, "has_vertical_vane", False):
                entities.append(MelViewVerticalVaneSelect(coordinator))

            if getattr(coordinator.device, "has_horizontal_vane", False):
                entities.append(MelViewHorizontalVaneSelect(coordinator))

    async_add_entities(entities, update_before_add=True)
```

### 4. Integration Setup (`__init__.py`)

Add `Platform.SELECT` to platforms list:
```python
PLATFORMS: list[Platform] = [
    Platform.CLIMATE,
    Platform.SWITCH,
    Platform.SENSOR,
    Platform.FAN,
    Platform.SELECT,  # NEW
]
```

### 5. Translations (`translations/en.json`)

Add translation keys:
```json
{
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

### 6. Constants (`const.py`)

Import vane mappings from `melview.py` if needed elsewhere (or keep them module-scoped in `melview.py`).

## Implementation Strategy

### Phase 1: API Layer
1. Add vane position constants to `melview.py`
2. Parse capability flags in `async_refresh_device_caps()`
3. Add `async_set_vertical_vane()` and `async_set_horizontal_vane()` methods
4. Test commands work on live unit

### Phase 2: Climate Entity
1. Add swing mode support to `MelViewClimate`
2. Feature flags based on capabilities
3. Test in climate card UI

### Phase 3: Select Entities
1. Create `select.py` platform
2. Add select entities gated by `CONF_SENSOR` option
3. Update `__init__.py` to load select platform
4. Add translations

### Phase 4: Version & Documentation
1. Bump version in `manifest.json`
2. Update `CLAUDE.md` with vane control architecture
3. Test with `ruff check` and `ruff format`
4. Commit and validate via GitHub Actions

## Testing Approach

1. **Local testing**: Run in live Home Assistant with real MelView unit
2. **Verify capabilities**: Check debug logs for `hasairdir`/`hasairdirh` values
3. **Test commands**: Verify `AV` and `AH` commands send correctly and unit responds
4. **UI verification**: Confirm climate card shows swing modes, select entities appear
5. **Edge cases**: Test units without vane support (flags = 0), test ERV units excluded

## Version Management

Following existing pattern: calendar versioning in `manifest.json`. Bump to `2026.2.1`.

## Success Criteria

- [ ] Vertical vane control works via climate entity swing mode
- [ ] Horizontal vane control works via climate entity swing horizontal mode
- [ ] Select entities created for units with vane support
- [ ] No entities created for units without vane support
- [ ] ERV units excluded from vane controls
- [ ] Existing climate features still work
- [ ] Code passes `ruff check` and `ruff format`
- [ ] GitHub Actions validation passes
