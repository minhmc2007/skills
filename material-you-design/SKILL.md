---
name: material-you-design
description: Reference for implementing Google's Material You dynamic-color design system on Android, as specified for AOSP integration — the wallpaper-to-65-color-API pipeline, tonal palette math (chroma/hue per accent and neutral group), theme style variants (TONAL_SPOT/VIBRANT/EXPRESSIVE/SPRITZ), ThemePicker/WallpaperPicker stub APK format, motion conventions (overscroll stretch, ripple), and widget API priorities. Use this whenever building or reviewing a custom Android ROM's theming layer, SystemUI overlays, a ThemePicker/WallpaperPicker fork, dynamic-color support in an app, or any AOSP-based UI work (e.g. InfinityX, LegacyDroid, or similar ROM trees) that should follow Material You conventions. Also use for general Material You / dynamic-color UI design questions even outside AOSP source trees.
---

# Material You Design (AOSP Reference)

Source: [AOSP — Material You Design](https://source.android.com/docs/core/display/material) (Android 12+). This skill distills that spec into an actionable reference for ROM/SystemUI theming work and general Material You app design.

Material You covers three integration surfaces on Android:
1. **Dynamic color** — wallpaper/theme-derived color extraction
2. **Motion** — overscroll stretch + ripple feedback
3. **Widgets** — sizing, rounding, and dynamic-color APIs for home screen widgets

Use the section that matches the task. If you're touching `frameworks/base/packages/SystemUI` theming, `ThemePicker`/`WallpaperPicker`, or a ROM's overlay/RRO layer, read **Dynamic Color** in full before writing code.

---

## 1. Dynamic Color

**Core rule: don't reinvent the extraction/expansion logic.** Use AOSP's own algorithm so third-party apps (most use Android's Material Components library) render consistently across devices. A hardcoded allowlist in `material-components-android` gates which OEM builds get proper dynamic-color treatment — custom ROMs should match stock AOSP behavior rather than diverge.

### The 4-step pipeline

1. **User picks a wallpaper or theme** via the OEM's picker.
2. **A single seed color is chosen** — either:
   - *Wallpaper & style*: AOSP logic extracts the seed from the wallpaper (most-frequent suitable color by default).
   - *Basic colors / device theme*: Android auto-selects a seed meeting requirements.
3. **AOSP expands the one seed color into 5 tonal palettes × 13 tones = 65 color attributes.**
4. **UI (system and third-party apps) consumes the 65 attributes consistently.** Recommendation: use the *same* palette for system UI and OEM first-party apps.

### Seed color constraints
- Minimum CAM16 chroma of **5**, so muted/near-neutral wallpapers still produce a usable accent (read/modify via `Cam#fromInt` / `Cam#getInt`).
- Extraction entry point: `ThemeOverlayController#mOnColorsChangedListener` in `frameworks/base/packages/SystemUI/src/com/android/systemui/theme/ThemeOverlayController.java`, driven by `WallpaperManager#onWallpaperColorsChanged`.
- To surface alternate candidate seeds in a theme picker UI (not just the top pick): `ColorScheme#getSeedColors(wallpaperColors: WallpaperColors)`.

### Palette expansion formulas (from one seed → 65 colors)

Each palette has 13 tone stops (0–1000 luminance range). Formulas below are chroma/hue deltas applied to the seed's CAM16 values:

| Palette | Chroma | Hue |
|---|---|---|
| `system_accent1` | 40 for tones 0/10/50/100, else 48 | same as seed |
| `system_accent2` | 16 | same as seed |
| `system_accent3` | 32 | seed hue **+60°** |
| `system_neutral1` | 4 | same as seed |
| `system_neutral2` | 8 | same as seed |

Validate with CTS: `atest SystemPalette`.

### Theme style variants (Android 13+)

`android.theme.customization.accent_color` is deprecated as of Android 13 in favor of `android.theme.customization.theme_style`:

- `TONAL_SPOT` — default Material You theme since Android 12 (S)
- `VIBRANT` — accent2/accent3 analogous to accent1
- `EXPRESSIVE` — highly chromatic
- `SPRITZ` — desaturated, near-grayscale

Set via `Settings.Secure.THEME_CUSTOMIZATION_OVERLAY_PACKAGES`:
```json
{
    "android.theme.customization.system_palette": "B1611C",
    "android.theme.customization.theme_style": "EXPRESSIVE"
}
```

### Android 12 and earlier (custom theme picker path)
Push a seed directly, skipping wallpaper extraction, via the same settings key:
```json
{
    "android.theme.customization.system_palette": "746BC1",
    "android.theme.customization.accent_color": "746BC1"
}
```

### Required Android 12 framework patches
If bringing dynamic color to an Android 12-based tree, cherry-pick (search these commit subjects in `frameworks/base`):
- Add monet to AOSP
- Adjust chroma/lstar filter for source colors
- Use max chroma of 40 at tone 90

Strongly recommended follow-ups: fixes for boot-color sysprop race conditions (two rounds), overlay theme-change listener support, FeatureFlags → flag package migration, multi-user theming, wallpaper-color-after-reboot fix, tertiary-hue calculation fix, and blocking background apps from changing the theme.

### Devices without wallpaper color extraction
Fall back to the default Material palette rather than leaving dynamic color half-implemented:
- Disable `flag_monet` in `frameworks/base/packages/SystemUI/res/values/flags.xml`.
- Still offer a preset-based theme picker so users retain some personalization.

### ThemePicker / WallpaperPicker stub APK format
WallpaperPicker only shows a color section when **both**:
- `flag_monet` is `true`, and
- a stub system APK's package name is declared in `themes_stub_package` in `packages/apps/ThemePicker/res/values/override.xml`.

A sample stub lives in `packages/apps/ThemePicker/themes`. The stub only needs resources — no code. Format:

`res/xml/*.xml` — list of preset bundle IDs and their display names:
```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <array name="color_bundles">
        <item>color1</item>
        <item>color2</item>
    </array>
    <string name="bundle_name_color1">Blue</string>
    <string name="bundle_name_color2">Red</string>
</resources>
```
Translate by adding matching strings under `res/values-<lang>/`.

Color values (same file or separate resource file) — each bundle needs matching primary/secondary entries:
```xml
<resources>
    <color name="color_primary_color1">#0000FF</color>
    <color name="color_secondary_color1">#0000FF</color>
</resources>
```

### Applying colors in UI (system or app-side)
```
// Accent — most foreground elements
@android:color/system_accent1_0 … 1000   // primary foreground group
@android:color/system_accent2_0 … 1000   // secondary accent / surfaces
@android:color/system_accent3_0 … 1000   // tertiary, playful/analogous

// Neutral — background elements
@android:color/system_neutral1_0 … 1000  // primary background group
@android:color/system_neutral2_0 … 1000  // elevated surfaces
```

### Material Components role mapping (light / dark)

| Role | Theme attr | Light | Dark |
|---|---|---|---|
| Primary | `colorPrimary` | accent1_600 | accent1_200 |
| On Primary | `colorOnPrimary` | accent1_0 | accent1_800 |
| Secondary | `colorSecondary` | accent2_600 | accent2_200 |
| On Secondary | `colorOnSecondary` | accent2_0 | accent2_800 |
| Error | `colorError` | red_600 (fixed) | red_200 (fixed) |
| On Error | `colorOnError` | white (fixed) | red_900 (fixed) |
| Background | `android:colorBackground` | neutral1_10 | neutral1_900 |
| On Background | `colorOnBackground` | neutral1_900 | neutral1_100 |
| Surface | `colorSurface` | neutral1_10 | neutral1_900 |
| On Surface | `colorOnSurface` | neutral1_900 | neutral1_100 |

State-layer/content roles follow the same accent1/accent2/accent3/neutral1/neutral2 groups at specific tones — check the source doc's full state table if implementing custom Material state layers (hover/press/focus content vs. layer colors differ slightly by role).

### Getting listed for third-party dynamic color support
Beyond the on-device integration, OEMs need to file a support ticket with `Build.MANUFACTURER` since Material Components for Android gates dynamic color behind a hardcoded device allowlist. Relevant for anyone shipping a ROM that wants upstream Material apps to pick up dynamic color automatically.

---

## 2. Motion

Two motion primitives must feel consistent with stock AOSP:

### Overscroll (edge stretch)
- On devices where `ActivityManager.isHighEndGfx()` is true: use the non-linear stretch effect (the springy edge-distortion look introduced in Android 12).
- On low-end devices: fall back to a simplified linear stretch to reduce GPU/CPU load.
- To support it in custom views/libraries, upgrade to:
  - `androidx.recyclerview:recyclerview:1.3.0-alpha01`+ for `RecyclerView`
  - `androidx.core:core:1.7.0-alpha01`+ for `NestedScrollView` / `EdgeEffectCompat`
  - `androidx.viewpager:viewpager:1.1-alpha01`+ for `ViewPager`
- UX rules for custom `EdgeEffect` layouts:
  - While content is mid-stretch, users should only be able to interact with the stretch itself — not tap buttons/content underneath.
  - If the user touches content while the `EdgeEffect` animation is still playing, let them "catch" it and continue manipulating the stretch (current pull distance via `EdgeEffectCompat.getDistance()`).
  - Use `onPullDistance()` to read/consume pull distance for a smooth stretch→scroll handoff once the finger passes the original position.
  - For nested scrolling: while stretched, the stretch should consume touch motion before the nested parent, otherwise a direction change mid-gesture can scroll the parent instead of releasing the stretch.

### Ripple (touch feedback)
- Android 12+ ripple is softer/subtler with smoother fill-in animation than earlier versions.
- No specific integration steps are required (it's largely automatic via standard widgets), but test on-device for visual regressions versus stock behavior.

---

## 3. Widgets

Two responsibilities:

**Platform side** (ROM/framework): support the modern widget developer APIs for layout, sizing, and style params (e.g. corner-radius), and let users freely configure/resize widgets.

**App side** (first-party widgets you ship): adopt current widget API features. Priority checklist (P0 = do first):

| Area | Change | Priority |
|---|---|---|
| Home screen UX | Scalable widget previews | P1 |
| | Widget description text | P1 |
| | Easier personalization | P2 (optional) |
| | Smoother transitions | **P0** |
| | Avoid tap-response via notification trampolines | **P0** |
| Widget guidelines | Refine widget sizes/layout | P2 |
| | Apply dynamic color | **P0** |
| | Rounded corners | **P0** |
| | New compound buttons | P2 |
| Code simplification | Simplified RemoteViews collections | P2 |
| | Simplified RemoteViews runtime | P2 |

---

## Applying this to a ROM tree

When working in an AOSP-derived tree (e.g. a LineageOS-based or AOSP-16-based ROM):
- Dynamic color logic lives under `frameworks/base/packages/SystemUI/src/com/android/systemui/theme/`.
- Feature flag: `frameworks/base/packages/SystemUI/res/values/flags.xml` (`flag_monet`).
- Theme picker integration point: `packages/apps/ThemePicker/` (stub APK under `themes/`, config under `res/values/override.xml`).
- If a ROM ships its own theme engine (per-app overlays, custom accent pickers, etc.), keep the 65-attribute contract (`system_accent1/2/3`, `system_neutral1/2`, each 0–1000) intact so unmodified third-party apps and stock system apps still theme correctly — diverging from this contract is the most common cause of "some apps don't follow my theme" bugs.
- When debugging inconsistent theming across apps, check whether the app in question uses Material Components' `DynamicColors` helper — apps not on the OEM allowlist (see above) may need the ROM to patch that allowlist or ship it via an RRO.
