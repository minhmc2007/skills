---
name: liquid-glass-compose
description: Implement Apple-style "liquid glass" UI effects in Jetpack Compose using the Backdrop library (io.github.kyant0:backdrop) — refraction/lens distortion, real-time blur, vibrancy, highlight/shadow/inner-shadow, and physically-springy touch interactions (press squish, drag-following highlight, damped toggle/slider). Use when building or reviewing glassmorphic Android UI components (glass cards, buttons, toggles, sliders, bottom tabs, nested "glass-on-glass" surfaces) in Compose.
---

# Liquid Glass in Jetpack Compose (Backdrop library)

Reference for `io.github.kyant0:backdrop` (v2.0.0) + `io.github.kyant0:shapes` (v1.2.0) — a Compose library that renders real-time refractive glass surfaces (blur + lens distortion + vibrancy) with physically-animated touch feedback, similar to iOS/visionOS "Liquid Glass".

## When to reach for this
Any Compose UI element that should look like frosted/refractive glass over dynamic content: cards, buttons, toggles, sliders, tab bars, dialogs, nested glass panels. Requires `minSdk 26+`; blur needs API 31+, lens refraction needs API 33+ (both degrade gracefully below that — see Fallback behavior).

## Setup

```kotlin
// app/build.gradle.kts
dependencies {
    implementation("io.github.kyant0:backdrop:2.0.0")
    implementation("io.github.kyant0:shapes:1.2.0") // Capsule, ContinuousCapsule, etc.
}
```

Kotlin files need an explicit `package` declaration when using AGP's built-in Kotlin support (AGP 9.x) — don't omit it even for single-file setups.

## Core mental model

1. **A `Backdrop` is the thing being refracted/blurred** — usually whatever's drawn behind the glass (a background, a scrollable content layer). Create one with `rememberLayerBackdrop { drawRect(...); drawContent() }` and attach it to the source content with `Modifier.layerBackdrop(backdrop)`.
2. **`Modifier.drawBackdrop(...)` is what turns a composable into a glass surface** reading from that backdrop.
3. Glass elements read from the *same* backdrop instance so they all refract the same underlying content consistently.

## `drawBackdrop` signature (2.0.0)

```kotlin
Modifier.drawBackdrop(
    backdrop: Backdrop,
    shape: () -> Shape = { RectangleShape },
    effects: BackdropEffectScope.() -> Unit = {},
    highlight: (() -> Highlight?)? = null,        // com.kyant.backdrop.highlight.Highlight
    shadow: (() -> Shadow?)? = null,               // com.kyant.backdrop.shadow.Shadow
    innerShadow: (() -> InnerShadow?)? = null,     // com.kyant.backdrop.shadow.InnerShadow
    exportedBackdrop: LayerBackdrop? = null,       // re-share this surface as a backdrop for nested glass, avoids infinite loop
    layerBlock: GraphicsLayerScope.() -> Unit = {},// scale/translate the whole layer (for press/drag squish)
    onDrawBackdrop: DrawScope.(drawBackdrop: DrawScope.() -> Unit) -> Unit = { it() },
    onDrawSurface: (DrawScope.() -> Unit)? = null, // tint/overlay drawn on top of the refracted content
)
```

## Effects (inside the `effects` block, order matters: colorFilter → blur → lens)

| Effect | Signature | Notes |
|---|---|---|
| `vibrancy()` | — | boosts saturation ×1.5 |
| `blur(radius)` | `radius: Float` (px) | Gaussian blur, **API 31+** |
| `lens(refractionHeight, refractionAmount, depthEffect, chromaticAberration)` | floats + booleans | edge refraction, **API 33+** |
| `colorFilter(colorFilter)` | `ColorFilter` | use `ColorFilter.tint(Color(...))`, not a raw `Color` |
| `colorControls(brightness, contrast, saturation)` | floats | |
| `opacity(alpha)` | `Float` | |

`exposureAdjustment()` and `gammaAdjustment()` are **not available** in 2.0.0 (unresolved reference if you try).

`dp` → `px` conversion: `BackdropEffectScope` has no `Density` scope, so pre-convert:
```kotlin
val density = LocalDensity.current
val br = with(density) { 12.dp.toPx() }
```

## Backdrop variants

- `rememberLayerBackdrop { drawRect(color); drawContent() }` + `Modifier.layerBackdrop(backdrop)` on the source — the standard case.
- `rememberCombinedBackdrop(b1, b2)` — merge multiple backdrops into one (e.g. a background + a nearby glass track, see toggle example below).
- `rememberCanvasBackdrop { drawing }` — coordinate-independent backdrop for pure-drawing content.
- `exportedBackdrop` param on `drawBackdrop` — lets a glass surface itself become a backdrop for glass nested inside it, without an infinite feedback loop ("glass-on-glass").

## Minimal glass card

```kotlin
val backdrop = rememberLayerBackdrop {
    drawRect(Color(0xFF0F0F1A))
    drawContent()
}

Box(Modifier.layerBackdrop(backdrop)) { /* your real background content */ }

Box(
    Modifier
        .fillMaxWidth()
        .height(120.dp)
        .drawBackdrop(
            backdrop = backdrop,
            shape = { RoundedCornerShape(24.dp) },
            effects = {
                vibrancy()
                blur(with(density) { 12.dp.toPx() })
                lens(with(density) { 24.dp.toPx() }, with(density) { 40.dp.toPx() }, depthEffect = true)
            },
            highlight = { Highlight.Default },
            shadow = { Shadow(radius = 24.dp, color = Color.Black.copy(alpha = 0.12f)) },
            innerShadow = { InnerShadow(radius = 6.dp) },
            onDrawSurface = { drawRect(Color.White.copy(alpha = 0.08f)) }
        )
)
```

## Nested glass ("glass-on-glass")

Export the outer glass surface as its own backdrop, then draw an inner glass element that reads from it:

```kotlin
val cardBackdrop = rememberLayerBackdrop()

Box(
    Modifier.drawBackdrop(
        backdrop = backdrop,
        shape = { RoundedCornerShape(28.dp) },
        effects = { vibrancy(); blur(...); lens(..., chromaticAberration = true) },
        exportedBackdrop = cardBackdrop,   // <- key line
        // ...highlight/shadow/innerShadow/onDrawSurface as usual
    )
) {
    Box(
        Modifier.drawBackdrop(
            backdrop = cardBackdrop,       // <- reads the outer glass, not the raw background
            shape = { Capsule() },
            effects = { vibrancy(); blur(...); lens(...) },
            // ...
        )
    )
}
```

## Tinted glass button (color via `BlendMode.Hue`)

Don't just draw a translucent colored rect — hue-blend then overlay, or the glass loses its refractive look:

```kotlin
onDrawSurface = {
    if (tint.isSpecified) {
        drawRect(tint, blendMode = BlendMode.Hue)
        drawRect(tint.copy(alpha = 0.75f))
    }
}
```

## Interactive touch feedback

Two reusable helper classes carry the "liquid" feel — press squish, a highlight that follows the finger, and spring-damped drag for toggles/sliders. Neither ships in the library; they're small enough to hand-write per project:

### 1. Press squish via `layerBlock` (buttons)
Track press position/progress with an `Animatable`-backed helper (`InteractiveHighlight` in the reference project), then in `layerBlock` translate/scale the glass layer toward the touch point using `tanh` to keep the offset bounded:

```kotlin
layerBlock = {
    val progress = interactiveHighlight.pressProgress
    val scale = lerp(1f, 1f + 4.dp.toPx() / size.height, progress)
    val maxOffset = size.minDimension
    val offset = interactiveHighlight.offset
    translationX = maxOffset * tanh(0.05f * offset.x / maxOffset)
    translationY = maxOffset * tanh(0.05f * offset.y / maxOffset)
    scaleX = scale + /* extra stretch along drag angle, clamped by aspect ratio */
    scaleY = scale + /* ... */
}
```
Drive it with a custom `pointerInput { inspectDragGestures(...) }` (see below) that snaps an `Offset` `Animatable` to the touch position on drag and animates a `pressProgress: Animatable<Float>` to 1 on press / 0 on release, using `spring(dampingRatio = 0.5f, stiffness = 300f, visibilityThreshold = ...)`.

### 2. Finger-following highlight (AGSL shader, optional)
For a soft radial highlight that follows the touch point, use a `RuntimeShader` (guard with `isRuntimeShaderSupported()` and fall back to a flat translucent overlay on unsupported devices):
```glsl
uniform float2 size;
layout(color) uniform half4 color;
uniform float radius;
uniform float2 position;

half4 main(float2 coord) {
    float dist = distance(coord, position);
    float intensity = smoothstep(radius, radius * 0.5, dist);
    return color * intensity;
}
```
Draw with `drawRect(ShaderBrush(shader.asComposeShader()), blendMode = BlendMode.Plus)` inside `Modifier.drawWithContent { ...; drawContent() }`, feeding `size`/`position`/`color` from the same press-position `Animatable` used above.

### 3. Damped drag for toggles/sliders
For elements the user drags (toggle knob, slider thumb), track five springs together — value, velocity, press progress, scaleX, scaleY — so the knob squishes on press and stretches slightly in the direction of motion:
- `valueAnimationSpec = spring(dampingRatio = 1f, stiffness = 1000f, visibilityThreshold)`
- `pressProgressAnimationSpec = spring(1f, 1000f, 0.001f)`
- `scaleXAnimationSpec` / `scaleYAnimationSpec` — slightly lower damping (`~0.6–0.7f`, stiffness `~250f`) so the squish overshoots a little
- Track velocity via `VelocityTracker`, feed it into `layerBlock` to stretch the knob along its motion axis:
```kotlin
layerBlock = {
    scaleX = dampedDrag.scaleX
    scaleY = dampedDrag.scaleY
    val v = dampedDrag.velocity / 50f
    scaleX /= 1f - (v * 0.75f).fastCoerceIn(-0.2f, 0.2f)
    scaleY *= 1f - (v * 0.25f).fastCoerceIn(-0.2f, 0.2f)
}
```
On release, only settle `pressProgress`/scale back to resting once the value animation is within ~2.5% of target, so the squish doesn't cut off mid-settle:
```kotlin
val threshold = (valueRange.endInclusive - valueRange.start) * 0.025f
snapshotFlow { valueAnimation.value }
    .filter { abs(it - valueAnimation.targetValue) < threshold }
    .first()
```

### Custom drag gesture inspector
`Modifier.pointerInput` + the stock `detectDragGestures` reports gesture *end* only after a real drag; for a component that must react on **every** pointer-down (even a tap that never moves) as well as reads `dragAmount` per-frame, write a small `inspectDragGestures` on top of `awaitEachGesture`/`awaitFirstDown`/manual `awaitPointerEvent` loop, firing `onDragStart` immediately on first down (not on first move) and `onDrag(change, Offset.Zero)` once up front so consumers can initialize state before any movement happens.

## Combining a track + a knob backdrop (toggle pattern)

When a small glass knob sits on top of its own glass track, combine both backdrops so the knob refracts the track *and* the wider scene behind it:
```kotlin
val trackBackdrop = rememberLayerBackdrop()
// track: Modifier.layerBackdrop(trackBackdrop)
// knob:
backdrop = rememberCombinedBackdrop(
    backdrop,
    rememberBackdrop(trackBackdrop) { drawBackdrop ->
        // scale down what the knob reads from the track so it doesn't just look like a hole
        scale(scaleX, scaleY) { drawBackdrop() }
    }
)
```

## Fallback / performance behavior

- `blur()` silently requires API 31+; `lens()` requires API 33+. On older devices the effect block should be written so it still compiles/runs — check `isRuntimeShaderSupported()` before using any `RuntimeShader`-based custom highlight, and provide a flat-color fallback.
- Prefer one shared `Backdrop` per screen (attached once via `layerBackdrop`) over creating a new one per glass element — every `drawBackdrop` call re-samples the same backdrop cheaply, but multiple independent `rememberLayerBackdrop` sources each cost their own re-composition/re-draw.
- Keep `effects` ordering consistent (`colorFilter` → `blur` → `lens`) — reordering changes the visual result, not just performance.

## Common pitfalls

- Forgetting `exportedBackdrop` when nesting glass → the inner surface either shows nothing or triggers a feedback loop.
- Using a raw `Color` instead of `ColorFilter.tint(Color(...))` in `colorFilter()` — won't compile.
- Calling `exposureAdjustment()`/`gammaAdjustment()` — not present in 2.0.0, despite appearing in some library examples/docs.
- Omitting the explicit `package` declaration in `.kt` files under AGP 9.x's built-in Kotlin support — silent module resolution failures.
- Doing `dp.toPx()` directly inside `effects { }` — there's no `Density` receiver there; convert outside with `LocalDensity.current` first.
