---
name: liquid-glass-compose-legacydroid
description: >
  Reference for building apps with Liquid Glass Compose effects inside the LegacyDroid AOSP source tree.
  Covers the Soong-built backdrop/shapes libraries, module wiring, and Compose API surface.
  DIFFERENT from the standard liquid-glass-compose skill — this one targets the AOSP soong build
  environment with prebuilt Compose AARs, platform_apis, and platform-signed APKs.
  Use when building or reviewing any app, SystemUI overlay, or system component in LegacyDroid
  that uses liquid glass / glassmorphism effects.
---

# Liquid Glass Compose — LegacyDroid (AOSP Soong)

LegacyDroid ships Liquid Glass (by kyant0) as two local `android_library` modules plus prebuilt Compose 1.7.0 AARs. Apps link them via `static_libs` in `Android.bp` — no Maven, no Gradle, no `implementation()`.

If you're working outside an AOSP tree or using Gradle, use the standard `liquid-glass-compose` skill instead.

> **Broken feature:** The lock screen SDF (Signed Distance Field) clock shader does not render correctly. If a request involves this effect, stop and inform the user it's broken — not planned for fix.

---

## Module dependency graph

```
LiquidGlassDemo (android_app, platform cert)
  └─ LiquidGlassCatalog (android_library)
       ├─ LiquidGlassBackdrop (android_library)
       │    ├─ LiquidGlassShapes (android_library)
       │    ├─ liquidglass-compose-foundation
       │    ├─ liquidglass-compose-ui
       │    ├─ liquidglass-compose-ui-graphics
       │    └─ androidx.compose.runtime_runtime
       ├─ LiquidGlassShapes
       ├─ kotlinx-coroutines-android
       ├─ liquidglass-activity-compose
       ├─ liquidglass-compose-animation / animation-core
       ├─ liquidglass-compose-foundation
       ├─ liquidglass-material-ripple
       ├─ liquidglass-compose-ui / ui-graphics / ui-util
       └─ androidx.compose.runtime_runtime
```

Module names in `Android.bp`:

| Module | Source path | Purpose |
|---|---|---|
| `LiquidGlassShapes` | `frameworks/libs/shapes/` | G2 continuous-curvature shapes (`Capsule`, `RoundedRectangle`, `UnevenRoundedRectangle`) |
| `LiquidGlassBackdrop` | `frameworks/libs/backdrop/` | Core glass effect engine (`drawBackdrop`, blur, lens, vibrancy, highlights, shadows) |
| `LiquidGlassCatalog` | `frameworks/libs/backdrop/app/` | Reusable component library (buttons, toggles, sliders, tabs, dialogs) |
| `LiquidGlassDemo` | `packages/apps/LiquidGlassDemo/` | System app shell that hosts the catalog |

---

## Setting up a new app module

### Minimal `Android.bp`

```blueprint
android_app {
    name: "MyGlassApp",
    srcs: ["src/**/*.kt"],
    platform_apis: true,
    certificate: "platform",
    manifest: "AndroidManifest.xml",
    resource_dirs: ["res"],

    static_libs: [
        "LiquidGlassCatalog",
        "androidx.core_core-ktx",
    ],

    kotlincflags: [
        "-opt-in=androidx.compose.foundation.ExperimentalFoundationApi",
        "-opt-in=androidx.compose.runtime.ExperimentalComposeApi",
    ],
}
```

`LiquidGlassCatalog` transitively pulls in `LiquidGlassBackdrop`, `LiquidGlassShapes`, and all Compose dependencies. One line covers the full stack.

### If you only need shapes (no glass effects)

```blueprint
static_libs: ["LiquidGlassShapes"]
```

### If you need backdrop but not the catalog components

```blueprint
static_libs: [
    "LiquidGlassBackdrop",
    "liquidglass-compose-foundation",
    "androidx.compose.runtime_runtime",
    "liquidglass-compose-ui",
    "liquidglass-compose-ui-graphics",
]
```

### Kotlin compiler flags

Always include both unless your module has no Compose:

```blueprint
kotlincflags: [
    "-opt-in=androidx.compose.foundation.ExperimentalFoundationApi",
    "-opt-in=androidx.compose.runtime.ExperimentalComposeApi",
],
```

---

## Shapes API (`com.kyant.shapes`)

Pure `Shape` implementations. No composables, no setup, no `CompositionLocal` providers.

### Types

```kotlin
// Uniform corner radius, G2 continuous curvature by default
class RoundedRectangle(
    val cornerRadius: Dp,
    val style: RoundedCornerStyle = RoundedCornerStyle.Continuous
) : RoundedRectangularShape

// Per-corner radius
class UnevenRoundedRectangle(
    topStart: Dp = 0.dp,
    topEnd: Dp = 0.dp,
    bottomEnd: Dp = 0.dp,
    bottomStart: Dp = 0.dp,
    val style: RoundedCornerStyle = RoundedCornerStyle.Continuous
) : RoundedRectangularShape

// Max-radius capsule (minDimension / 2)
class Capsule(
    val style: RoundedCornerStyle = RoundedCornerStyle.Continuous
) : RoundedRectangularShape

// Zero-radius rectangle
object Rectangle : RoundedRectangularShape

enum class RoundedCornerStyle {
    Circular,    // standard Android arc corners
    Continuous   // G2 curvature (iOS-style squircle)
}
```

### Interpolation

```kotlin
fun lerp(
    start: RoundedRectangularShape,
    stop: RoundedRectangularShape,
    fraction: Float,
    style: RoundedCornerStyle = start.style ?: RoundedCornerStyle.Continuous
): RoundedRectangularShape
```

Use this for animated shape transitions (e.g. rectangle → capsule on press).

### Usage

Shapes are standard Compose `Shape` objects. Use them anywhere a `Shape` is accepted:

```kotlin
Modifier.clip(Capsule())
Modifier.border(1.dp, color, RoundedRectangle(16.dp))
drawOutline(shape.createOutline(size, layoutDirection, this), brush)
```

---

## Backdrop API (`com.kyant.backdrop`)

### Core workflow

1. Create a `LayerBackdrop` via `rememberLayerBackdrop()`
2. Attach it to your content source via `Modifier.layerBackdrop(backdrop)`
3. Apply effects via `Modifier.drawBackdrop(backdrop, shape, effects)`

```kotlin
// 1. Create backdrop
val backdrop = rememberLayerBackdrop()

// 2. Register content source
Image(
    painter = painterResource(R.drawable.wallpaper),
    contentDescription = null,
    modifier = Modifier.layerBackdrop(backdrop).fillMaxSize()
)

// 3. Draw glass on top
Box(
    modifier = Modifier.drawBackdrop(
        backdrop = backdrop,
        shape = { Capsule() },
        effects = {
            vibrancy()
            blur(2f.dp.toPx())
            lens(12f.dp.toPx(), 24f.dp.toPx())
        },
        highlight = { Highlight.Default },
        shadow = { Shadow.Default }
    )
)
```

### `drawBackdrop` parameters

```kotlin
fun Modifier.drawBackdrop(
    backdrop: Backdrop,
    shape: () -> Shape,
    effects: BackdropEffectScope.() -> Unit,
    highlight: (() -> Highlight?)? = { Highlight.Default },
    shadow: (() -> Shadow?)? = { Shadow.Default },
    innerShadow: (() -> InnerShadow?)? = null,
    layerBlock: (GraphicsLayerScope.() -> Unit)? = null,
    exportedBackdrop: LayerBackdrop? = null,      // export for nested glass
    onDrawBehind: (DrawScope.() -> Unit)? = null,
    onDrawBackdrop: DrawScope.(drawBackdrop: DrawScope.() -> Unit) -> Unit = { it() },
    onDrawSurface: (DrawScope.() -> Unit)? = null,
    onDrawFront: (DrawScope.() -> Unit)? = null
): Modifier
```

For glass without highlights/shadows, use `drawPlainBackdrop` (same signature minus highlight/shadow/innerShadow).

### Effects DSL (`BackdropEffectScope`)

Chain effects inside the `effects` lambda. Order matters — each effect wraps the previous.

```kotlin
effects = {
    blur(8f.dp.toPx())
    vibrancy()
    lens(
        refractionHeight = 16f.dp.toPx(),
        refractionAmount = 32f.dp.toPx(),
        depthEffect = true,
        chromaticAberration = false
    )
    colorControls(brightness = 0.2f, contrast = 1f, saturation = 1.5f)
}
```

| Effect | Signature | Notes |
|---|---|---|
| `blur` | `blur(radius: Float, edgeTreatment: TileMode = Clamp)` | Gaussian blur |
| `lens` | `lens(refractionHeight, refractionAmount, depthEffect, chromaticAberration)` | Glass refraction |
| `vibrancy` | `vibrancy()` | Saturation boost (~1.5x) |
| `colorFilter` | `colorFilter(colorFilter: ColorFilter)` | Arbitrary color filter |
| `opacity` | `opacity(alpha: Float)` | Transparency |
| `colorControls` | `colorControls(brightness, contrast, saturation)` | Per-channel adjustment |
| `effect` | `effect(renderEffect: RenderEffect)` | Raw Android RenderEffect |
| `runtimeShaderEffect` | `runtimeShaderEffect(key, shaderString, uniformName, block)` | Custom AGSL shader |

### Backdrop creators

```kotlin
@Composable fun rememberLayerBackdrop(
    graphicsLayer: GraphicsLayer = rememberGraphicsLayer(),
    onDraw: ContentDrawScope.() -> Unit = { drawContent() }
): LayerBackdrop

@Composable fun rememberCanvasBackdrop(
    onDraw: DrawScope.() -> Unit
): Backdrop

@Composable fun rememberBackdrop(
    backdrop: Backdrop,
    onDraw: DrawScope.(drawBackdrop: DrawScope.() -> Unit) -> Unit
): Backdrop

@Composable fun rememberCombinedBackdrop(backdrop1: Backdrop, backdrop2: Backdrop): Backdrop
@Composable fun rememberCombinedBackdrop(b1: Backdrop, b2: Backdrop, b3: Backdrop): Backdrop
@Composable fun rememberCombinedBackdrop(vararg backdrops: Backdrop): Backdrop

fun emptyBackdrop(): Backdrop
```

- `rememberLayerBackdrop` — captures Compose content as backdrop source (most common)
- `rememberCanvasBackdrop` — draws arbitrary content (solid color, custom shapes)
- `rememberBackdrop` — wraps an existing backdrop with transform logic
- `rememberCombinedBackdrop` — composites multiple sources into one

### Highlight

```kotlin
data class Highlight(
    val width: Dp = 0.5f.dp,
    val blurRadius: Dp = width / 2f,
    val alpha: Float = 1f,
    val style: HighlightStyle = HighlightStyle.Default
) {
    companion object {
        val Default: Highlight   // directional, angle-aware
        val Ambient: Highlight   // omnidirectional
        val Plain: Highlight     // flat solid
    }
}

interface HighlightStyle {
    data class Default(val color: Color, val blendMode: BlendMode, val angle: Float, val falloff: Float)
    data class Ambient(val intensity: Float)
    data class Plain(val color: Color, val blendMode: BlendMode)
}
```

`HighlightStyle.Default` accepts an `angle` parameter — use with accelerometer data for gravity-responsive highlights (see Control Center demo).

### Shadow

```kotlin
data class Shadow(
    val radius: Dp = 24.dp,
    val offset: DpOffset = DpOffset(0.dp, radius / 6f),
    val color: Color = Color.Black.copy(alpha = 0.1f),
    val alpha: Float = 1f,
    val blendMode: BlendMode = DrawScope.DefaultBlendMode
)

data class InnerShadow(
    val radius: Dp = 24.dp,
    val offset: DpOffset = DpOffset(0.dp, radius),
    val color: Color = Color.Black.copy(alpha = 0.15f),
    val alpha: Float = 1f,
    val blendMode: BlendMode = DrawScope.DefaultBlendMode
)
```

### Nested glass (exported backdrop)

```kotlin
val innerBackdrop = rememberLayerBackdrop()

// Outer glass exports its rendered content
Modifier.drawBackdrop(
    backdrop = outerBackdrop,
    shape = { RoundedRectangle(32.dp) },
    effects = { blur(4f.dp.toPx()); lens(16f.dp.toPx(), 32f.dp.toPx()) },
    exportedBackdrop = innerBackdrop
)

// Inner glass consumes the exported backdrop
Modifier.drawBackdrop(
    backdrop = innerBackdrop,
    shape = { Capsule() },
    effects = { blur(2f.dp.toPx()); lens(8f.dp.toPx(), 16f.dp.toPx()) }
)
```

### Platform checks

```kotlin
fun isRenderEffectSupported(): Boolean  // API 31+ (Android S)
fun isRuntimeShaderSupported(): Boolean  // API 33+ (Android T)
```

Effects no-op gracefully on older APIs. AGSL shaders (`runtimeShaderEffect`) require API 33+.

---

## Catalog components (`com.kyant.backdrop.catalog`)

Pre-built glass components from `LiquidGlassCatalog`. Add `LiquidGlassCatalog` to `static_libs` to use them.

### LiquidButton

```kotlin
@Composable
fun LiquidButton(
    onClick: () -> Unit,
    backdrop: Backdrop,
    modifier: Modifier = Modifier,
    isInteractive: Boolean = true,
    tint: Color = Color.Unspecified,
    surfaceColor: Color = Color.Unspecified,
    content: @Composable RowScope.() -> Unit
)
```

### LiquidToggle

```kotlin
@Composable
fun LiquidToggle(
    checked: Boolean,
    onCheckedChange: (Boolean) -> Unit,
    backdrop: Backdrop,
    modifier: Modifier = Modifier,
    trackColor: Color = ...,
    checkedThumbColor: Color = ...,
    uncheckedThumbColor: Color = ...
)
```

### LiquidSlider

```kotlin
@Composable
fun LiquidSlider(
    value: Float,
    onValueChange: (Float) -> Unit,
    backdrop: Backdrop,
    modifier: Modifier = Modifier,
    valueRange: ClosedFloatingPointRange<Float> = 0f..1f,
    trackColor: Color = ...,
    accentColor: Color = ...
)
```

### LiquidBottomTabs

```kotlin
@Composable
fun LiquidBottomTabs(
    selectedTabIndex: Int,
    backdrop: Backdrop,
    modifier: Modifier = Modifier,
    tabs: @Composable () -> Unit
)

@Composable
fun LiquidBottomTab(
    selected: Boolean,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    icon: @Composable () -> Unit,
    label: @Composable () -> Unit
)
```

---

## Common patterns from the demo app

### Wallpaper-backed glass

The standard pattern — wallpaper image serves as backdrop source:

```kotlin
val backdrop = rememberLayerBackdrop()

Image(
    painter = painterResource(R.drawable.wallpaper),
    contentDescription = null,
    modifier = Modifier.layerBackdrop(backdrop).fillMaxSize()
)

// Glass overlay
Box(
    Modifier.drawBackdrop(
        backdrop = backdrop,
        shape = { Capsule() },
        effects = { vibrancy(); blur(2f.dp.toPx()); lens(12f.dp.toPx(), 24f.dp.toPx()) }
    )
)
```

### Solid-color backdrop (no wallpaper)

```kotlin
val backdrop = rememberCanvasBackdrop {
    drawRect(Color(0xFF1A1A2E))
}
```

### Glass with gravity-responsive highlight

```kotlin
val uiSensor = rememberUISensor()

Modifier.drawBackdrop(
    backdrop = backdrop,
    shape = { RoundedRectangle(itemSize / 2) },
    effects = { vibrancy(); lens(depthEffect = true) },
    highlight = {
        Highlight.Default.copy(
            style = HighlightStyle.Default(angle = uiSensor.gravityAngle)
        )
    }
)
```

Requires `UISensor` utility from the catalog (`utils/UISensor.kt`).

### Glass dialog overlay

```kotlin
// Dim background
Modifier.drawWithContent {
    drawRect(Color.Black.copy(alpha = 0.4f))
    drawContent()
}

// Glass dialog
Box(
    Modifier.drawBackdrop(
        backdrop = backdrop,
        shape = { RoundedRectangle(48.dp) },
        effects = {
            colorControls(brightness = brightness, saturation = saturation)
            blur(32f.dp.toPx())
            lens(24f.dp.toPx(), 48f.dp.toPx(), depthEffect = true)
        },
        highlight = { Highlight.Plain },
        onDrawSurface = { drawRect(containerColor) }
    )
)
```

### Animated shape transition (press feedback)

```kotlin
val pressProgress = remember { Animatable(0f) }

LaunchedEffect(isPressed) {
    pressProgress.animateTo(
        targetValue = if (isPressed) 1f else 0f,
        animationSpec = spring(dampingRatio = 0.6f, stiffness = 300f)
    )
}

val shape = lerp(RoundedRectangle(16.dp), Capsule(), pressProgress.value)
```

### Custom AGSL shader effect

```kotlin
effects = {
    runtimeShaderEffect(
        key = "AlphaMask",
        shaderString = """
            uniform shader content;
            uniform float2 size;
            half4 main(float2 coord) {
                float alpha = smoothstep(size.y * 0.6, size.y, coord.y);
                return content.eval(coord) * half4(1, 1, 1, alpha);
            }
        """,
        uniformShaderName = "content"
    ) {
        setFloatUniform("size", size.width, size.height)
    }
}
```

Requires API 33+. Check `isRuntimeShaderSupported()` before use.

---

## Build gotchas

1. **`platform_apis: true`** — required for all liquid glass modules. The libraries use internal Compose APIs that aren't in the public SDK.

2. **`certificate: "platform"`** — demo app is platform-signed. Your app needs this if it accesses system-level features, but regular apps can use their own signing.

3. **Prebuilt Compose AARs** — Compose dependencies are prefixed `liquidglass-*` (not `androidx.compose.*`). These are prebuilt AARs in `frameworks/libs/backdrop/prebuilts/compose-ui-1.7.0/`. Don't add `androidx.compose` deps directly.

4. **Kotlin compiler opt-ins** — missing the `ExperimentalComposeApi` flag causes build failures on `rememberGraphicsLayer()` and related APIs. Always include both kotlincflags.

5. **minSdk 21, targetSdk 31** — the prebuilt Compose AARs target SDK 31. `lens()` and blur effects require API 31+ at runtime; AGSL shaders require API 33+. Graceful fallback on older devices.

6. **No Gradle in the tree** — the `build.gradle.kts` files exist for IDE support and upstream development only. The actual build uses Soong (`Android.bp`). Don't add Gradle dependencies expecting them to resolve.

7. **Manifest merging** — `LiquidGlassCatalog`'s manifest declares `MainActivity`. If your app module has its own manifest, make sure the activity declaration doesn't conflict. The demo app module omits `<activity>` entirely and inherits it from the catalog.

8. **Lock screen SDF texture is broken** — the SDF (Signed Distance Field) clock shader in the lock screen demo does not render correctly. Not planned for fix. If a request involves the lock screen SDF effect, stop and tell the user it's broken.

---

## Source locations

| What | Path |
|---|---|
| Shapes library | `frameworks/libs/shapes/shapes/src/main/kotlin/com/kyant/shapes/` |
| Shapes build | `frameworks/libs/shapes/Android.bp` |
| Backdrop library | `frameworks/libs/backdrop/backdrop/src/main/kotlin/com/kyant/backdrop/` |
| Backdrop build | `frameworks/libs/backdrop/Android.bp` |
| Prebuilt Compose AARs | `frameworks/libs/backdrop/prebuilts/compose-ui-1.7.0/` |
| Catalog components | `frameworks/libs/backdrop/app/src/main/kotlin/com/kyant/backdrop/catalog/` |
| Catalog build | `frameworks/libs/backdrop/app/Android.bp` |
| Demo app | `packages/apps/LiquidGlassDemo/` |
| Demo app build | `packages/apps/LiquidGlassDemo/Android.bp` |
