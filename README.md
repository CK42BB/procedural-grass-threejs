# Procedural Grass — Claude Three.js Skill

A Claude Code skill for generating dense, animated procedural grass fields in Three.js, emphasizing **WebGPU compute with WebGL fallback**.

Part of the **Teaching Three.js** skill series.

## What This Skill Does

When installed, this skill teaches Claude Code how to generate procedural grass including:

- **Bezier-curved blade geometry** with tapered triangle strips and configurable segments
- **Instanced rendering** — hundreds of thousands of blades in a single draw call
- **Multi-layer wind system** — global sway, rolling gust waves, per-blade turbulence
- **Subsurface scattering** approximation for realistic backlit translucency
- **Terrain-aware placement** with slope rejection, density modulation, and jittered grids
- **Interactive displacement** — grass pushes aside from players/objects
- **Distance-based LOD** — density fade and blade simplification by camera distance
- **8 grass type presets** — lawn, meadow, tall prairie, wheat, savanna, reeds, tundra, tropical
- **GPU compute placement** via WGSL for WebGPU-accelerated scatter
- **Seasonal color** modulation and mixed multi-type fields

## Installation

### Claude Code (CLI)

```bash
claude install-skill path/to/procedural-grass
```

Or copy the `procedural-grass/` folder into your Claude Code skills directory.

### Manual Usage

Reference the skill files directly in conversations with Claude:

| File | Purpose |
|------|---------|
| `SKILL.md` | Main skill — blade geometry, placement, wind, materials, LOD, presets |
| `references/blade-shaders.md` | Complete GLSL vert/frag shaders, WGSL compute, TSL wind nodes |
| `references/grass-types.md` | 8 species profiles with dimensions, colors, density, biome maps |

## Quick Start Prompts

Try these with Claude after installing:

> "Create a meadow with procedural grass blowing in the wind"

> "Build a wheat field with golden backlit grass and interactive displacement"

> "Generate savanna-style grass with clumpy tufts and bare patches"

> "Create a mixed temperate grass field with three LOD rings and distance fade"

> "Add procedural grass to my existing terrain with WebGPU compute placement"

## Requirements

- **Three.js r170+** for WebGPU support and TSL node materials
- WebGPU-capable browser (Chrome 121+, Edge 121+) for GPU compute features
- WebGL2 fallback works in all modern browsers

## Skill Architecture

```
procedural-grass/
├── SKILL.md                          # Core skill (read first)
├── README.md                         # This file
└── references/
    ├── blade-shaders.md              # GLSL, WGSL, and TSL shader code
    └── grass-types.md                # Species profiles and presets
```

## Key Concepts

### Blade Geometry

Each grass blade is a **tapered triangle strip** shaped along a quadratic bezier curve. This gives natural curvature with minimal vertex count (as few as 7 vertices per blade at low LOD).

### Wind Layers

Wind is not a single sine wave — it's three layered frequencies that combine for realism:

| Layer | Frequency | Effect |
|-------|-----------|--------|
| Global sway | Low | Entire field leans together |
| Gust waves | Medium | Visible "fronts" rolling across |
| Turbulence | High | Individual blade flutter |

All wind is computed in the vertex shader — zero per-frame JavaScript loops.

### Performance Targets

| Platform | Max Blades | Approach |
|----------|-----------|----------|
| Mobile | 50K–100K | Single InstancedMesh, 3-segment blades |
| Desktop | 200K–500K | LOD rings, 5-segment blades |
| High-end + WebGPU | 500K–2M | GPU compute placement, storage buffers |

### Grass Type Presets

8 built-in presets with full geometry, color, wind, and density settings:

| Type | Height | Character |
|------|--------|-----------|
| Lawn | 0.25m | Dense, short, barely moves |
| Meadow | 0.9m | Natural pastoral, gentle sway |
| Tall Prairie | 1.8m | Dramatic deep movement |
| Wheat | 1.15m | Golden, drooping heads |
| Savanna | 0.75m | Sparse dry tufts |
| Reeds | 2.2m | Tall, straight, waterside |
| Tundra | 0.18m | Stubby, wind-battered |
| Tropical | 1.5m | Lush, broad, vivid green |

## Series: Teaching Three.js

Independent skills that compose for complete environments:

- [procedural-landscapes](../procedural-landscapes-threejs/) — terrain
- [procedural-clouds](../procedural-clouds-threejs/) — clouds
- [procedural-stars](../procedural-stars-threejs/) — nightime skies
- [procedural-weather](../procedural-weather-threejs/) — precipitation and atmosphere
- **procedural-grass** ← this skill.

**Companion skill**: [procedural-landscapes](../procedural-landscapes/) — terrain generation pairs naturally with this grass system.

## License

MIT — use freely in your projects.
