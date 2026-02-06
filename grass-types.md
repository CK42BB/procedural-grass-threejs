# Grass Types Reference

Detailed species profiles for procedural grass generation. Each type includes blade
geometry parameters, color palettes, density settings, wind behavior, and biome context.

## Anatomy of a Grass Blade

```
          tip (pointed or rounded)
          ╱
     ────╱──── ← width narrows toward tip
    │         │
    │  blade  │ ← curvature = lateral bend
    │  body   │
    │         │
    └────┬────┘ ← width at base
         │
      ───┴─── root (fixed to ground)
```

Key parameters:
- **width**: Base width in world units
- **height**: Full blade height
- **curvature**: Lateral bezier control point offset (0 = straight, 1 = heavily curved)
- **taper**: How quickly width narrows (0.8 = standard, 0.95 = very pointy)
- **segments**: Triangle strip subdivisions (more = smoother curve)

## Type Profiles

### Lawn Grass (Festuca / Ryegrass)

Manicured, short, dense, uniform. Think golf courses, suburban yards.

```javascript
const LAWN = {
  blade: { width: 0.025, height: 0.25, curvature: 0.08, segments: 3 },
  placement: {
    density: 100,           // Very dense, carpet-like
    heightVariation: 0.15,  // Mowed = low variation
    clumpiness: 0.1,        // Very uniform distribution
  },
  color: {
    base:     0x1a6b12,     // Deep green at root
    tip:      0x3da82e,     // Lighter green at tip
    dry:      0x7d9a3c,     // Slight yellowing when stressed
    dryAmount: 0.05,
    variation: 0.08,        // Minimal per-blade color shift
  },
  wind: {
    baseStrength: 0.1,      // Barely moves
    gustStrength: 0.15,
    stiffness: 0.9,         // Short = stiff
  },
  lighting: {
    sssStrength: 0.3,       // Moderate translucency
    aoStrength: 0.4,
  },
  biomes: ['temperate', 'suburban', 'sports fields'],
};
```

### Meadow Grass (Mixed Wildgrass)

Natural, moderate height, varied, gently swaying. Classic pastoral field.

```javascript
const MEADOW = {
  blade: { width: 0.055, height: 0.9, curvature: 0.3, segments: 5 },
  placement: {
    density: 35,
    heightVariation: 0.4,   // Natural = varied heights
    clumpiness: 0.4,        // Some clustering
  },
  color: {
    base:     0x2e6b1f,
    tip:      0x7cb83a,
    dry:      0xa89b4a,
    dryAmount: 0.12,
    variation: 0.2,         // Noticeable per-blade differences
  },
  wind: {
    baseStrength: 0.35,
    gustStrength: 0.7,
    stiffness: 0.5,
  },
  lighting: {
    sssStrength: 0.5,
    aoStrength: 0.6,
  },
  biomes: ['temperate', 'forest edge', 'hillside'],
};
```

### Tall Prairie Grass (Big Bluestem / Switchgrass)

Tall, dramatic, deep movement. North American prairie character.

```javascript
const TALL_GRASS = {
  blade: { width: 0.08, height: 1.8, curvature: 0.5, segments: 7 },
  placement: {
    density: 18,
    heightVariation: 0.5,
    clumpiness: 0.5,        // Grows in tufts
  },
  color: {
    base:     0x3a6b2e,
    tip:      0x95b84a,     // Bright yellow-green tips
    dry:      0xb89d5a,     // Autumn: bronze/amber
    dryAmount: 0.2,
    variation: 0.25,
    // Seasonal: multiply dryAmount by season factor (0=spring, 1=autumn)
  },
  wind: {
    baseStrength: 0.5,
    gustStrength: 1.0,      // Dramatic sway
    stiffness: 0.3,         // Very flexible
  },
  lighting: {
    sssStrength: 0.6,       // Strong backlit glow
    aoStrength: 0.7,
  },
  biomes: ['prairie', 'steppe', 'riverbank'],
};
```

### Wheat / Grain Field

Golden, uniform height, heavy heads that droop. Agricultural character.

```javascript
const WHEAT = {
  blade: { width: 0.035, height: 1.15, curvature: 0.55, segments: 5 },
  placement: {
    density: 55,            // Planted rows = dense
    heightVariation: 0.1,   // Agricultural = uniform
    clumpiness: 0.05,       // Rows, not clumps
    rowAlignment: true,     // Optional: align to grid
    rowSpacing: 0.15,
  },
  color: {
    base:     0x8b7d3c,     // Straw gold
    tip:      0xd4c462,     // Bright gold
    dry:      0xc8a83a,     // Fully ripe
    dryAmount: 0.7,         // Naturally golden
    variation: 0.1,
  },
  wind: {
    baseStrength: 0.25,
    gustStrength: 0.55,
    stiffness: 0.4,         // Heavy head makes it droop and sway
  },
  lighting: {
    sssStrength: 0.65,      // Beautiful backlit golden glow
    aoStrength: 0.5,
  },
  biomes: ['agricultural', 'temperate plains'],
};
```

### Savanna Grass (Elephant Grass / Fountain Grass)

Sparse, tough, warm-toned. African/Australian grassland character.

```javascript
const SAVANNA = {
  blade: { width: 0.05, height: 0.75, curvature: 0.2, segments: 4 },
  placement: {
    density: 12,            // Sparse with bare patches
    heightVariation: 0.35,
    clumpiness: 0.7,        // Distinct tufts with gaps
    clumpRadius: 0.8,       // Tight clumps
  },
  color: {
    base:     0x7d6b3a,     // Tan/brown base
    tip:      0xc4a85a,     // Dry golden tips
    dry:      0x9b8650,
    dryAmount: 0.5,
    variation: 0.15,
  },
  wind: {
    baseStrength: 0.25,
    gustStrength: 0.4,
    stiffness: 0.6,         // Tough, wiry
  },
  lighting: {
    sssStrength: 0.4,
    aoStrength: 0.5,
  },
  biomes: ['savanna', 'dry grassland', 'bushveld'],
};
```

### Reed / Cattail (Phragmites / Typha)

Tall, straight, grows near water. Distinctive vertical character.

```javascript
const REEDS = {
  blade: { width: 0.04, height: 2.2, curvature: 0.15, segments: 6 },
  placement: {
    density: 25,
    heightVariation: 0.3,
    clumpiness: 0.6,
    waterProximity: true,   // Only place near water bodies
    maxDistFromWater: 3.0,
  },
  color: {
    base:     0x4a6b2e,     // Dark green
    tip:      0x7da84a,     // Medium green
    dry:      0x8b8650,     // Autumn brown
    dryAmount: 0.1,
    variation: 0.12,
  },
  wind: {
    baseStrength: 0.3,
    gustStrength: 0.5,
    stiffness: 0.55,        // Tall but fairly stiff stems
  },
  lighting: {
    sssStrength: 0.45,
    aoStrength: 0.65,
  },
  biomes: ['wetland', 'marsh', 'riverbank', 'lake shore'],
};
```

### Tundra Grass (Arctic Sedge / Cotton Grass)

Very short, tough, wind-battered. Sparse ground cover in harsh environments.

```javascript
const TUNDRA = {
  blade: { width: 0.03, height: 0.18, curvature: 0.04, segments: 2 },
  placement: {
    density: 20,
    heightVariation: 0.2,
    clumpiness: 0.8,        // Survives in tight clusters
  },
  color: {
    base:     0x5a6b3a,     // Muted olive
    tip:      0x7d8b5a,     // Pale green
    dry:      0x8b8660,     // Bleached
    dryAmount: 0.25,
    variation: 0.1,
  },
  wind: {
    baseStrength: 0.5,      // Strong wind
    gustStrength: 1.0,      // Fierce gusts
    stiffness: 0.85,        // Short = resistant
  },
  lighting: {
    sssStrength: 0.25,
    aoStrength: 0.3,
  },
  biomes: ['tundra', 'alpine', 'subarctic'],
};
```

### Tropical Grass (Pampas / Sugarcane)

Tall, lush, broad-bladed. Tropical and subtropical character.

```javascript
const TROPICAL = {
  blade: { width: 0.1, height: 1.5, curvature: 0.45, segments: 6 },
  placement: {
    density: 25,
    heightVariation: 0.45,
    clumpiness: 0.5,
  },
  color: {
    base:     0x1a7a1a,     // Vivid deep green
    tip:      0x4ab82e,     // Bright lime
    dry:      0x6b8b3a,
    dryAmount: 0.05,        // Rarely dry
    variation: 0.2,
  },
  wind: {
    baseStrength: 0.3,
    gustStrength: 0.6,
    stiffness: 0.35,        // Broad and flexible
  },
  lighting: {
    sssStrength: 0.55,      // Lush translucency
    aoStrength: 0.7,
  },
  biomes: ['tropical', 'subtropical', 'jungle edge'],
};
```

## Clumping Algorithm

Many grass types grow in clumps rather than uniformly. Apply clump-based density:

```javascript
function applyClumping(x, z, clumpiness, clumpRadius = 1.0, noiseFn) {
  if (clumpiness <= 0) return 1.0;

  // Voronoi-like clump centers from noise
  const cx = Math.floor(x / clumpRadius);
  const cz = Math.floor(z / clumpRadius);
  let minDist = Infinity;

  for (let dx = -1; dx <= 1; dx++) {
    for (let dz = -1; dz <= 1; dz++) {
      const seed = (cx + dx) * 127 + (cz + dz) * 311;
      const ox = (cx + dx + fract(Math.sin(seed) * 43758.5453)) * clumpRadius;
      const oz = (cz + dz + fract(Math.sin(seed * 1.7) * 43758.5453)) * clumpRadius;
      const d = Math.sqrt((x - ox) ** 2 + (z - oz) ** 2);
      minDist = Math.min(minDist, d);
    }
  }

  const clumpFactor = 1.0 - Math.min(minDist / clumpRadius, 1.0);
  return Math.pow(clumpFactor, clumpiness * 2);
}
```

## Seasonal Color Modulation

Shift grass colors based on a `season` parameter (0 = spring, 0.5 = summer, 1 = autumn).

```javascript
function seasonalColor(preset, season) {
  const springGreen = new THREE.Color(preset.color.base);
  const summerGreen = new THREE.Color(preset.color.tip);
  const autumnGold = new THREE.Color(preset.color.dry);

  const base = springGreen.clone().lerp(summerGreen, Math.min(season * 2, 1));
  const tip = summerGreen.clone().lerp(autumnGold, Math.max((season - 0.5) * 2, 0));
  const dryAmount = preset.color.dryAmount + Math.max(season - 0.6, 0) * 1.5;

  return { baseColor: base, tipColor: tip, dryAmount: Math.min(dryAmount, 1) };
}
```

## Mixed Grass Fields

Combine multiple types for natural variation. Layer 2–3 types with different heights:

```javascript
function createMixedField(terrain, biome = 'temperate') {
  const layers = [];

  if (biome === 'temperate') {
    layers.push({ type: LAWN,      weight: 0.5, heightRange: [0.05, 0.4] });
    layers.push({ type: MEADOW,    weight: 0.35, heightRange: [0.1, 0.6] });
    layers.push({ type: TALL_GRASS, weight: 0.15, heightRange: [0.2, 0.5] });
  } else if (biome === 'savanna') {
    layers.push({ type: SAVANNA,   weight: 0.7, heightRange: [0.0, 0.5] });
    layers.push({ type: TALL_GRASS, weight: 0.3, heightRange: [0.1, 0.4] });
  } else if (biome === 'wetland') {
    layers.push({ type: MEADOW,    weight: 0.4, heightRange: [0.05, 0.3] });
    layers.push({ type: REEDS,     weight: 0.6, heightRange: [0.0, 0.08] });
  }

  // Each layer becomes its own InstancedMesh for different blade geometries
  return layers;
}
```
