# Plan: L84V24P200 / L84V24P100 LED Driver Light Support

## Background

The Lightwave L84V24P200 (200W) and L84V24P100 (100W) LED drivers are physical devices that can drive:
- **CCT LED strips** – controllable colour temperature (warm/cool) + brightness
- **RGBW LED strips** – controllable hue, saturation, white channel + brightness

The Lightwave app exposes full control. However, the current `homeassistant-lightwave-smart` integration only exposes these devices as **Switch** entities with no light control. This plan documents why, and how to fix it.

---

## How the Integration Works

### Architecture

```
Lightwave Cloud API (WebSocket)
        │
lightwave_smart Python library  (PyPI: lightwave-smart 2.0.2)
        │                       (source: /lightwave-smart/lightwave_smart/)
        │
homeassistant-lightwave-smart HACS integration
        │                       (source: /custom_components/lightwave_smart/)
        │
Home Assistant entities (light, switch, sensor, climate, cover, lock, event, binary_sensor)
```

### Key library concepts

- **Devices** (`LWRFDevice`) – physical hardware, identified by `device_id`. Has a `product_code` (e.g. `L84V24P200`).
- **Feature sets** (`LWRFFeatureSet`) – logical channel within a device (a 2-channel device has 2 feature sets). Identified by `featureset_id`. Each featureset has a `name` (user-facing, e.g. "Kitchen LED Strip").
- **Features** (`LWRFFeature`) – individual capabilities within a featureset (e.g. `switch`, `dimLevel`, `power`, `rgbColor`, `colorTemperature`). Feature states are read from/written to the API by `featureId`.

### Device type detection (in `device.py` → `LWRFFeatureSet`)

```python
def is_switch(self): return has_feature('switch') and not has_feature('dimLevel') and not has_feature('socketSetup')
def is_outlet(self): return has_feature('socketSetup')
def is_light(self):  return has_feature('dimLevel')   # <-- key method
def has_led(self):   return has_feature('rgbColor') and not self.virtual_product_code
```

### HA entity selection (in `light.py`)

```python
link.get_lights()    # → featuresets where is_light() = True → LWRF2Light entity (dimmer)
link.get_sockets()   # → featuresets where is_outlet() = True → checks has_led() for indicator
link.get_hubs()      # → checks has_led() for indicator
# NOTE: get_switches() is NOT checked for LED/colour features
```

---

## Root Cause

The LED driver featuresets do **not** expose a `dimLevel` feature.

- `is_light()` → `False` (no `dimLevel`) → NOT added as a light entity  
- `is_switch()` → `True` (has `switch`, no `dimLevel`, no `socketSetup`) → added as a switch entity

The colour/temperature features (e.g. `rgbColor`, `colorTemperature`) are present on the featureset, but:
1. The HA integration only checks `has_led()` for `get_lights()`, `get_sockets()`, and `get_hubs()` — **not** `get_switches()`
2. Even if it did, the `LWRF2LED` entity class is designed for LED indicators on switch faceplates, not for driving LED strips as a primary light source

### What the device exposes (best current understanding)

Based on the Lightwave app UI and the feature naming convention in the library:

| Device type | Expected Lightwave API features |
|---|---|
| CCT LED driver | `switch`, `colorTemperature`, `dimLevel`(?), `power`, `energy`, `lock`, `uiButtonPair` |
| RGBW LED driver | `switch`, `rgbColor`, `dimLevel`(?), `power`, `energy`, `lock`, `uiButtonPair` |

> ⚠️ **CRITICAL UNKNOWN**: The actual feature names returned by the API for these LED driver devices are not yet confirmed. `dimLevel` may or may not be present. The features shown in the Lightwave app (brightness slider + colour temp OR hue + saturation) may map differently to API feature names. This **must be verified before implementation**.

---

## Step 0: Confirmed Feature Names (RESOLVED)

Both L84V24P200 devices expose the following Lightwave API features (confirmed via `lwrf_*` state attributes):

| Feature name | RGBW device value | CCT device value | Notes |
|---|---|---|---|
| `switch` | 0 | 0 | On/off |
| `dimLevel` | 100 | 100 | Brightness 0-100 |
| `colourTemperature` | 300 | 300 | **Mireds** — confirmed: 1,000,000/K = mireds exactly |
| `colourHue` | 171 | 0 | Hue 0-360 |
| `colourSaturation` | 19 | 100 | Saturation 0-100 |
| `colourSetup` | **2** | **3** | **Key discriminator**: 2=RGBW, 3=CCT |
| `colourAnimation` | 0 | 0 | |
| `colourCalibrationMax` | 4294967295 | 65535 | Hardware calibration, not useful for us |
| `colourCalibrationMin` | 0 | 0 | Hardware calibration |
| `colourOutputMap` | 1347 | 65313 | Hardware mapping |
| `power` | 0 | 0 | Watts |
| `energy` | 0 | 8 | Wh |
| `uiButton` | null | null | |
| `uiButtonPair` | null | null | |

**Key findings:**
- Both devices have `dimLevel` → `is_light()` returns True → they ARE already picked up as `LightEntity` via `link.get_lights()`
- The existing `LWRF2Light` class only supports `ColorMode.BRIGHTNESS` — it ignores colour features
- Both devices have `colourHue`, `colourSaturation`, `colourTemperature`, `colourSetup`
- `colourSetup` state determines mode: **2 = RGBW** (uses hue+sat+brightness), **3 = CCT** (uses colour temp+brightness)
- `colourTemperature` is in **mireds** — confirmed: `1,000,000 / K = mireds` exactly
- **Confirmed colour temperature range (CCT, L84V24P200)**: 2000K (500 mireds) warm → 9800K (102 mireds) cool
- **Edge case**: at 0% brightness, `lwrf_switch` stays `1` (on) but `lwrf_dimLevel: 0`. Must treat `dimLevel==0` as off in `is_on`.
- **Critical**: writing `dimLevel=0` to the device **destroys colour state** — `colourHue` and `colourSaturation` both reset to 0 on the hardware itself. Using `switch=0` to turn off **preserves** colour. Therefore `async_turn_on(brightness=0)` must redirect to `async_turn_off()` (switch=0) rather than writing dimLevel=0.
- **Hue range**: 0–360 (wraps — both top and bottom of rainbow slider = red = hue 0)
- **Saturation range**: 0–100 (0 = pure white, 100 = full colour)
- **Brightness preserves hue/sat**: confirmed stable across all brightness levels >0
- **Lock feature**: `lwrf_protection: 0` = unlocked, `lwrf_protection: 1` = locked (handled by existing `lock.py`, separate from light entity)

---

## Step 1: No Library Changes Required

The HA integration can detect LED driver featuresets directly via feature presence checks on `featureset.features` (a plain dict accessible from the integration layer). No changes needed in `lightwave-smart` library for initial implementation.

Detection logic (in `light.py`'s `async_setup_entry`):
```python
# LED driver featuresets have colourHue + colourSaturation + colourTemperature + colourSetup
# Regular dimmers only have dimLevel (and switch)
if (featureset.has_feature('colourHue')
        and featureset.has_feature('colourSetup')
        and featureset.has_feature('colourTemperature')):
    # → LED driver → use LWRF2LEDDriver
else:
    # → regular dimmer → use LWRF2Light (existing)
```

---

## Step 2: Add `LWRF2LEDDriver` Entity Class

### Why a single class for both CCT and RGBW

The HA `supported_color_modes` property must be set at entity creation time. The `colourSetup` feature state (which distinguishes CCT from RGBW) is populated by a background task (`async_update_featureset_states`) that may not have completed when platform setup runs. To avoid a race condition, a single entity class supports both `ColorMode.COLOR_TEMP` and `ColorMode.HS`, using `colourSetup` state to determine the active `color_mode` at runtime:

- `colourSetup == 3` → `color_mode = ColorMode.COLOR_TEMP` (CCT strip)
- `colourSetup == 2` → `color_mode = ColorMode.HS` (RGBW strip, controlled via hue+saturation)
- `colourSetup == None` (not yet loaded) → defaults to `ColorMode.COLOR_TEMP`

### Feature mapping

| HA parameter | Lightwave feature | Scale conversion |
|---|---|---|
| `brightness` (0-255) | `dimLevel` (0-100) | `dimLevel = round(brightness / 255 * 100)` |
| `color_temp_kelvin` | `colourTemperature` (mireds) | `mireds = round(1_000_000 / kelvin)` |
| `hs_color` hue (0-360) | `colourHue` (0-360) | direct (int) |
| `hs_color` saturation (0-100) | `colourSaturation` (0-100) | direct (int) |

### Colour temperature range

Defaulting to 2700 K–6500 K (370–154 mireds). **TODO**: verify by sliding to extremes in Lightwave app and reading `lwrf_colourTemperature` values.

### Unique ID

Reuses key `smartLightSwitch` (same as existing `LWRF2Light`) so HA entity registry treats this as an update to the existing entity, preserving automations and scene references.

### `async_setup_entry` routing (updated)

```python
for featureset_id, name in link.get_lights():
    featureset = link.featuresets[featureset_id]
    try:
        if (featureset.has_feature('colourHue')
                and featureset.has_feature('colourSetup')
                and featureset.has_feature('colourTemperature')):
            lights.append(LWRF2LEDDriver(name, featureset_id, link))
        else:
            lights.append(LWRF2Light(name, featureset_id, link, homekit))
    except Exception as e:
        _LOGGER.exception("Could not add light for %s", featureset_id)
```

### 2d. LED driver switch entity

LED drivers are picked up by `get_lights()` (has `dimLevel`) and NOT by `get_switches()` (excluded because they have `dimLevel`). No duplicate switch entity is created. The existing switch entity visible in the HA device view appears to be from `get_lights()` being treated as `LWRF2Light` with name "Switch" — this will be fixed by naming the new entity "Light".

---

## Step 3: Library Versioning

If changes are made to the `lightwave-smart` library:
1. Bump the version in `setup.py`
2. Update `manifest.json` `requirements` in the HA integration to reference the new version (or a local/git install during development)

During development, the library can be installed directly from the local checkout via HA's `custom_components` approach or by symlinking.

---

## Questions to Resolve

| # | Question | Status |
|---|---|---|
| 1 | What feature names does the API return for L84V24P200? | ✅ Confirmed |
| 2 | Is `dimLevel` present? | ✅ Yes (0-100) |
| 3 | What unit does `colourTemperature` use? | ✅ Mireds (300 mireds = 3333 K) |
| 4 | Does RGBW use packed `rgbColor` or separate H/S? | ✅ Separate: `colourHue` (0-360) + `colourSaturation` (0-100) |
| 5 | What are the min/max colour temperature values? | ✅ 2000K (500 mireds) warm → 9800K (102 mireds) cool |
| 6 | Should the existing Switch entity remain visible? | TBD — probably yes for automations |
| 7 | Does the library `is_switch()` pick up LED drivers? | ✅ No — `is_switch()` requires `not has_feature('dimLevel')`, LED drivers have `dimLevel` |

---

## Files to Change

| File | Change |
|---|---|
| `custom_components/lightwave_smart/light.py` | Add `LWRF2LEDDriver` class; update `async_setup_entry` routing |
| `custom_components/lightwave_smart/manifest.json` | No change needed (no library changes) |

Library changes are **not required** for initial implementation.

---

## Do We Need to Branch the Library Repo?

**No** — all changes are in the HA integration only (`custom_components/lightwave_smart/light.py`). The `lightwave-smart` library already provides `async_write_feature_by_name` which handles arbitrary feature writes. No library fork or version bump needed for this implementation.
