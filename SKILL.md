---
name: procedural-grass
description: >
  Generate procedural grass fields in Three.js using WebGPU compute with automatic
  WebGL2 fallback. Covers instanced blade geometry with bezier-curve profiles, multi-layer
  wind simulation, subsurface scattering approximation, distance-based LOD and density
  falloff, interactive displacement from players/objects, and configurable grass types
  (lawn, meadow, wheat, reeds, savanna, tundra). Use when building grass systems, fields,
  meadows, prairies, ground cover vegetation, or any scene requiring dense animated plant
  coverage. Triggers: "procedural grass", "grass field", "grass blades", "meadow",
  "grass rendering", "grass shader", "wind grass", "instanced grass", "grass LOD",
  "ground cover", "lawn", "wheat field", "tall grass".
---

# Procedural Grass

Generate dense, animated, visually rich procedural grass in Three.js with a WebGPU-first
pipeline and automatic WebGL2 fallback.

## Architecture Overview

```
┌──────────────────────────────────────────────────────┐
│                   Grass Pipeline                      │
│                                                      │
│  1. Blade Geometry ── bezier-curved triangle strip    │
│  2. Placement ─────── terrain-aware scatter + density │
│  3. Instancing ────── InstancedMesh / storage buffer  │
│  4. Wind ──────────── layered noise displacement      │
│  5. Shading ───────── SSS approx + color variation    │
│  6. LOD ───────────── density fade + blade simplify   │
│  7. Interaction ───── radial push from world objects  │
├──────────────────────────────────────────────────────┤
│  WebGPU path: compute placement + storage buffers     │
│  WebGL path:  CPU placement + InstancedMesh           │
└──────────────────────────────────────────────────────┘
```

## Blade Geometry

Each grass blade is a **tapered triangle strip** shaped along a quadratic bezier curve.
This gives natural curvature with minimal vertex count.

### Blade Mesh Generator

```javascript
function createBladeGeometry(segments = 4, width = 0.06, height = 1.0, curvature = 0.3) {
  // segments+1 cross-sections, 2 verts each, plus 1 tip vertex
  const vertCount = (segments + 1) * 2 + 1;
  const positions = new Float32Array(vertCount * 3);
  const uvs = new Float32Array(vertCount * 2);
  const indices = [];

  for (let i = 0; i <= segments; i++) {
    const t = i / segments;
    // Quadratic bezier: p0=(0,0), p1=(curvature, 0.5), p2=(0, 1)
    const x = 2 * (1 - t) * t * curvature;           // lateral curve
    const y = t * height;                              // vertical
    const w = width * (1 - t * 0.8);                   // taper

    const vi = i * 2;
    // Left vertex
    positions[(vi) * 3]     = x - w * 0.5;
    positions[(vi) * 3 + 1] = y;
    positions[(vi) * 3 + 2] = 0;
    uvs[(vi) * 2]     = 0;
    uvs[(vi) * 2 + 1] = t;
    // Right vertex
    positions[(vi + 1) * 3]     = x + w * 0.5;
    positions[(vi + 1) * 3 + 1] = y;
    positions[(vi + 1) * 3 + 2] = 0;
    uvs[(vi + 1) * 2]     = 1;
    uvs[(vi + 1) * 2 + 1] = t;
  }

  // Tip vertex
  const tipIdx = (segments + 1) * 2;
  const tipX = 2 * 0.5 * 0.5 * curvature; // t≈midpoint approximation
  positions[tipIdx * 3]     = curvature * 0.5;
  positions[tipIdx * 3 + 1] = height;
  positions[tipIdx * 3 + 2] = 0;
  uvs[tipIdx * 2]     = 0.5;
  uvs[tipIdx * 2 + 1] = 1.0;

  // Triangle strip indices
  for (let i = 0; i < segments; i++) {
    const a = i * 2, b = i * 2 + 1, c = (i + 1) * 2, d = (i + 1) * 2 + 1;
    indices.push(a, b, c, b, d, c);
  }
  // Tip triangles
  const lastL = segments * 2, lastR = segments * 2 + 1;
  indices.push(lastL, lastR, tipIdx);

  const geometry = new THREE.BufferGeometry();
  geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
  geometry.setAttribute('uv', new THREE.BufferAttribute(uvs, 2));
  geometry.setIndex(indices);
  geometry.computeVertexNormals();
  return geometry;
}
```

**Segment count guide**: 3 segments for distant LOD, 4–5 for mid-range, 6–8 for close-up hero grass.

## Instance Data Layout

Each blade instance stores placement and variation data packed into instance attributes.

```javascript
function createGrassInstanceData(count) {
  return {
    // vec4: x, y, z, rotation
    positionRotation: new Float32Array(count * 4),
    // vec4: scaleX, scaleY, tilt, colorVariation
    scaleAndVariation: new Float32Array(count * 4),
  };
}
```

## Placement System

### CPU Placement (WebGL path)

Scatter blades on terrain with density modulation, slope rejection, and jittered grid for
uniform distribution without clumping.

```javascript
function placeGrassOnTerrain({
  terrainSize, maxHeight, heightFn, noiseFn,
  density = 40,        // blades per unit² at max density
  minHeight = 0.05,    // normalized terrain height
  maxSlopeAngle = 0.6, // radians, reject steep slopes
  seed = 0,
} = {}) {
  const gridStep = 1 / Math.sqrt(density);
  const halfSize = terrainSize / 2;
  const instances = [];

  // Seeded random for reproducibility
  let rng = seed;
  const random = () => { rng = (rng * 16807 + 0) % 2147483647; return rng / 2147483647; };

  for (let gx = -halfSize; gx < halfSize; gx += gridStep) {
    for (let gz = -halfSize; gz < halfSize; gz += gridStep) {
      // Jitter within grid cell
      const x = gx + (random() - 0.5) * gridStep;
      const z = gz + (random() - 0.5) * gridStep;

      // Normalized terrain coordinates
      const nx = x / terrainSize + 0.5;
      const nz = z / terrainSize + 0.5;
      if (nx < 0 || nx > 1 || nz < 0 || nz > 1) continue;

      const h = heightFn(nx, nz);
      if (h < minHeight) continue;

      // Slope check via finite difference
      const eps = gridStep * 0.5;
      const hx = heightFn(nx + eps / terrainSize, nz);
      const hz = heightFn(nx, nz + eps / terrainSize);
      const slope = Math.atan(Math.sqrt((hx - h) ** 2 + (hz - h) ** 2) * maxHeight / eps);
      if (slope > maxSlopeAngle) continue;

      // Density modulation via noise (patches and bare spots)
      const densityNoise = noiseFn(x * 0.05, z * 0.05);
      if (densityNoise < -0.2) continue; // bare patches
      if (random() > (densityNoise * 0.5 + 0.7)) continue;

      const y = h * maxHeight;
      const rotation = random() * Math.PI * 2;
      const scaleX = 0.7 + random() * 0.6;
      const scaleY = 0.6 + random() * 0.8;
      const tilt = (random() - 0.5) * 0.3;
      const colorVar = random();

      instances.push({ x, y, z, rotation, scaleX, scaleY, tilt, colorVar });
    }
  }
  return instances;
}
```

### Building the InstancedMesh

```javascript
function buildGrassField(instances, bladeGeometry, material, maxCount) {
  const count = Math.min(instances.length, maxCount);
  const mesh = new THREE.InstancedMesh(bladeGeometry, material, count);

  const posRot = new Float32Array(count * 4);
  const scaleVar = new Float32Array(count * 4);

  for (let i = 0; i < count; i++) {
    const inst = instances[i];
    posRot[i * 4]     = inst.x;
    posRot[i * 4 + 1] = inst.y;
    posRot[i * 4 + 2] = inst.z;
    posRot[i * 4 + 3] = inst.rotation;
    scaleVar[i * 4]     = inst.scaleX;
    scaleVar[i * 4 + 1] = inst.scaleY;
    scaleVar[i * 4 + 2] = inst.tilt;
    scaleVar[i * 4 + 3] = inst.colorVar;
  }

  const geo = mesh.geometry.clone();
  geo.setAttribute('aPositionRotation',
    new THREE.InstancedBufferAttribute(posRot, 4));
  geo.setAttribute('aScaleVariation',
    new THREE.InstancedBufferAttribute(scaleVar, 4));
  mesh.geometry = geo;
  mesh.frustumCulled = false; // Grass displacement may extend outside bounds
  return mesh;
}
```

## Wind System

Multi-layered wind combines a global directional flow, turbulent gusts, and per-blade
high-frequency flutter.

```javascript
class WindSystem {
  constructor() {
    this.direction = new THREE.Vector2(1, 0.3).normalize();
    this.baseStrength = 0.4;
    this.gustStrength = 0.8;
    this.gustFrequency = 0.3;
    this.time = 0;
  }

  update(deltaTime) {
    this.time += deltaTime;
  }

  // Returns uniform values for shaders
  getUniforms() {
    return {
      windTime: this.time,
      windDir: this.direction,
      windBase: this.baseStrength,
      windGust: this.gustStrength,
      windGustFreq: this.gustFrequency,
    };
  }
}
```

Wind is applied in the vertex shader — see `references/blade-shaders.md` for the full
multi-layer wind displacement implementation.

**Wind layers**:
1. **Global sway**: Low-frequency sinusoidal along wind direction. Affects all blades uniformly.
2. **Gust waves**: Medium-frequency noise waves that roll across the field, creating visible "wind fronts".
3. **Turbulence**: Per-blade high-frequency flutter from hash-based variation.
4. **Height modulation**: Displacement scales with blade UV.y² — roots stay fixed, tips move most.

## Shading

### Grass Material (WebGL — ShaderMaterial)

The grass shader handles:
- Per-instance color variation (base + tip gradient)
- Subsurface scattering approximation (light through blades)
- Ambient occlusion at blade roots
- Distance fade to alpha for LOD blending

```javascript
function createGrassMaterial(params = {}) {
  const {
    baseColor = new THREE.Color(0x3a7d2c),
    tipColor = new THREE.Color(0x8bbf40),
    dryColor = new THREE.Color(0xc4a84b),
    dryAmount = 0.0,
    sssStrength = 0.5,
    aoStrength = 0.6,
    fadeStart = 60,
    fadeEnd = 80,
  } = params;

  return new THREE.ShaderMaterial({
    uniforms: {
      baseColor:    { value: baseColor },
      tipColor:     { value: tipColor },
      dryColor:     { value: dryColor },
      dryAmount:    { value: dryAmount },
      sssStrength:  { value: sssStrength },
      aoStrength:   { value: aoStrength },
      sunDir:       { value: new THREE.Vector3(0.5, 0.8, 0.3).normalize() },
      sunColor:     { value: new THREE.Color(0xfff4e5) },
      ambientColor: { value: new THREE.Color(0x4488aa) },
      windTime:     { value: 0 },
      windDir:      { value: new THREE.Vector2(1, 0.3) },
      windBase:     { value: 0.4 },
      windGust:     { value: 0.8 },
      windGustFreq: { value: 0.3 },
      fadeStart:    { value: fadeStart },
      fadeEnd:      { value: fadeEnd },
      cameraPos:    { value: new THREE.Vector3() },
      // Interactive displacement (up to 4 objects)
      pushPositions: { value: [new THREE.Vector3(), new THREE.Vector3(),
                               new THREE.Vector3(), new THREE.Vector3()] },
      pushRadii:     { value: [0, 0, 0, 0] },
    },
    vertexShader: GRASS_VERT,   // See references/blade-shaders.md
    fragmentShader: GRASS_FRAG, // See references/blade-shaders.md
    side: THREE.DoubleSide,
    transparent: true,
    depthWrite: true,
    alphaTest: 0.1,
  });
}
```

Full GLSL vertex and fragment shaders are in `references/blade-shaders.md`.

### Grass Node Material (WebGPU — TSL)

```javascript
import { attribute, cameraPosition, color, dot, float as tslFloat, max as tslMax,
         mix, normalize as tslNormalize, positionWorld, smoothstep, uniform,
         vec2, vec3, vec4, MeshStandardNodeMaterial } from 'three/tsl';

function createGrassNodeMaterial(params = {}) {
  const material = new MeshStandardNodeMaterial();
  material.side = THREE.DoubleSide;

  const uv = attribute('uv');
  const colorVar = attribute('aScaleVariation').w;
  const heightT = uv.y;

  const base = color(params.baseColor ?? 0x3a7d2c);
  const tip = color(params.tipColor ?? 0x8bbf40);

  // Height gradient + per-instance variation
  let grassColor = mix(base, tip, heightT);
  grassColor = mix(grassColor, color(params.dryColor ?? 0xc4a84b),
                   colorVar.mul(tslFloat(params.dryAmount ?? 0)));

  // Root ambient occlusion
  const ao = mix(tslFloat(1.0 - (params.aoStrength ?? 0.6)), tslFloat(1), heightT);
  grassColor = grassColor.mul(ao);

  material.colorNode = grassColor;
  material.roughnessNode = tslFloat(0.8);
  material.metalness = 0;
  return material;
}
```

## LOD System

### Distance-Based Density

Rather than rendering all blades at full density everywhere, partition into rings
and reduce count with distance.

```javascript
class GrassLODManager {
  constructor(scene, terrain, options = {}) {
    this.scene = scene;
    this.rings = [
      { radius: 20,  density: 1.0,  segments: 5, label: 'near' },
      { radius: 45,  density: 0.4,  segments: 3, label: 'mid' },
      { radius: 80,  density: 0.1,  segments: 2, label: 'far' },
    ];
    this.meshes = [];
    this.grassMaterial = options.material;
  }

  build(instances) {
    // Sort instances by distance from origin (re-sorted each update)
    for (const ring of this.rings) {
      const bladeGeo = createBladeGeometry(ring.segments);
      const ringInstances = instances.filter(inst => {
        const d = Math.sqrt(inst.x ** 2 + inst.z ** 2);
        const prevRadius = this.rings[this.rings.indexOf(ring) - 1]?.radius ?? 0;
        return d >= prevRadius && d < ring.radius;
      });

      // Thin by density factor
      const thinned = ringInstances.filter((_, i) =>
        i % Math.round(1 / ring.density) === 0
      );

      if (thinned.length > 0) {
        const mesh = buildGrassField(thinned, bladeGeo, this.grassMaterial, thinned.length);
        this.scene.add(mesh);
        this.meshes.push(mesh);
      }
    }
  }

  dispose() {
    for (const mesh of this.meshes) {
      this.scene.remove(mesh);
      mesh.geometry.dispose();
    }
    this.meshes = [];
  }
}
```

**For moving cameras**: rebuild LOD rings when camera moves beyond a threshold (e.g. 10 units),
or use shader-based distance fade (simpler, no geometry rebuild):

```glsl
// In fragment shader — fade alpha by distance
float dist = length(cameraPos - worldPos);
float fade = 1.0 - smoothstep(fadeStart, fadeEnd, dist);
if (fade < 0.01) discard;
gl_FragColor.a *= fade;
```

## Interactive Displacement

Push grass aside when players or objects move through it.

```javascript
class GrassInteraction {
  constructor(material, maxPushers = 4) {
    this.material = material;
    this.pushers = [];
    this.maxPushers = maxPushers;
  }

  addPusher(object, radius = 1.5) {
    if (this.pushers.length >= this.maxPushers) return;
    this.pushers.push({ object, radius });
  }

  update() {
    const positions = this.material.uniforms.pushPositions.value;
    const radii = this.material.uniforms.pushRadii.value;

    for (let i = 0; i < this.maxPushers; i++) {
      if (i < this.pushers.length) {
        const p = this.pushers[i];
        positions[i].copy(p.object.position);
        radii[i] = p.radius;
      } else {
        radii[i] = 0;
      }
    }
  }
}
```

The vertex shader applies radial displacement away from each pusher — see
`references/blade-shaders.md` for the implementation.

## Complete Scene Assembly

```javascript
import * as THREE from 'three';

async function init() {
  // Renderer (WebGPU with WebGL fallback)
  const canvas = document.querySelector('#canvas');
  let renderer, gpuAvailable = false;

  try {
    const WebGPU = (await import('three/addons/capabilities/WebGPU.js')).default;
    if (WebGPU.isAvailable()) {
      const { default: WebGPURenderer } = await import(
        'three/addons/renderers/webgpu/WebGPURenderer.js'
      );
      renderer = new WebGPURenderer({ canvas, antialias: true });
      await renderer.init();
      gpuAvailable = true;
    }
  } catch (e) { /* fallback */ }

  if (!renderer) {
    renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
  }
  renderer.setSize(innerWidth, innerHeight);
  renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
  renderer.shadowMap.enabled = true;

  const scene = new THREE.Scene();
  scene.background = new THREE.Color(0x87ceeb);
  scene.fog = new THREE.FogExp2(0xc8e6c0, 0.008);

  const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.1, 200);
  camera.position.set(0, 5, 15);

  const { OrbitControls } = await import('three/addons/controls/OrbitControls.js');
  const controls = new OrbitControls(camera, renderer.domElement);
  controls.target.set(0, 1, 0);
  controls.enableDamping = true;

  // Lighting
  const sun = new THREE.DirectionalLight(0xfff4e5, 1.5);
  sun.position.set(30, 40, 20);
  sun.castShadow = true;
  scene.add(sun);
  scene.add(new THREE.AmbientLight(0x88aacc, 0.4));
  scene.add(new THREE.HemisphereLight(0x87ceeb, 0x4a7c3f, 0.3));

  // Ground plane
  const ground = new THREE.Mesh(
    new THREE.PlaneGeometry(100, 100),
    new THREE.MeshStandardMaterial({ color: 0x3d6b2e, roughness: 0.9 })
  );
  ground.rotation.x = -Math.PI / 2;
  ground.receiveShadow = true;
  scene.add(ground);

  // Noise (reuse from procedural-landscapes or inline)
  const noise = createNoise2D(42); // See procedural-landscapes skill

  // Flat terrain height function (for demo)
  const heightFn = (nx, nz) => 0.001;

  // Place grass
  const instances = placeGrassOnTerrain({
    terrainSize: 60, maxHeight: 1, heightFn, noiseFn: noise,
    density: 30, minHeight: 0, maxSlopeAngle: 1.5,
  });

  // Build grass
  const bladeGeo = createBladeGeometry(5, 0.06, 1.0, 0.3);
  const grassMat = createGrassMaterial();
  const grassMesh = buildGrassField(instances, bladeGeo, grassMat, 200000);
  scene.add(grassMesh);

  // Wind
  const wind = new WindSystem();

  // Animate
  const clock = new THREE.Clock();
  renderer.setAnimationLoop(() => {
    const dt = clock.getDelta();
    wind.update(dt);

    const wu = wind.getUniforms();
    grassMat.uniforms.windTime.value = wu.windTime;
    grassMat.uniforms.windDir.value.copy(wu.windDir);
    grassMat.uniforms.cameraPos.value.copy(camera.position);

    controls.update();
    renderer.render(scene, camera);
  });

  window.addEventListener('resize', () => {
    camera.aspect = innerWidth / innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(innerWidth, innerHeight);
  });
}

init();
```

## Grass Type Presets

Quick-start configurations for different grass types. Full species catalog with
blade profiles, dimensions, and color palettes in `references/grass-types.md`.

```javascript
const GRASS_PRESETS = {
  lawn: {
    width: 0.03, height: 0.3, curvature: 0.1, segments: 3,
    density: 80, baseColor: 0x2d7a1e, tipColor: 0x5cb33a,
    dryAmount: 0, windBase: 0.15, windGust: 0.2,
  },
  meadow: {
    width: 0.06, height: 1.0, curvature: 0.3, segments: 5,
    density: 35, baseColor: 0x3a7d2c, tipColor: 0x8bbf40,
    dryAmount: 0.1, windBase: 0.4, windGust: 0.8,
  },
  tallGrass: {
    width: 0.08, height: 1.8, curvature: 0.5, segments: 6,
    density: 20, baseColor: 0x4a7c3f, tipColor: 0xa8c94e,
    dryAmount: 0.15, windBase: 0.5, windGust: 1.0,
  },
  wheat: {
    width: 0.04, height: 1.2, curvature: 0.6, segments: 5,
    density: 50, baseColor: 0x8b7d3c, tipColor: 0xd4c462,
    dryAmount: 0.7, windBase: 0.3, windGust: 0.6,
  },
  savanna: {
    width: 0.05, height: 0.8, curvature: 0.2, segments: 4,
    density: 15, baseColor: 0x9b8b4a, tipColor: 0xd4c078,
    dryAmount: 0.6, windBase: 0.3, windGust: 0.5,
  },
  tundra: {
    width: 0.04, height: 0.2, curvature: 0.05, segments: 2,
    density: 25, baseColor: 0x6b7d4a, tipColor: 0x8b9d5a,
    dryAmount: 0.3, windBase: 0.6, windGust: 1.2,
  },
};

function createGrassFromPreset(presetName) {
  const p = GRASS_PRESETS[presetName];
  const geo = createBladeGeometry(p.segments, p.width, p.height, p.curvature);
  const mat = createGrassMaterial({
    baseColor: new THREE.Color(p.baseColor),
    tipColor: new THREE.Color(p.tipColor),
    dryAmount: p.dryAmount,
  });
  return { geometry: geo, material: mat, preset: p };
}
```

## Performance Guidelines

**Instance budget by platform**:
| Platform | Max Blades | Draw Calls |
|----------|-----------|------------|
| Mobile | 50K–100K | 1–3 |
| Desktop | 200K–500K | 1–5 |
| High-end + WebGPU | 500K–2M | 1–3 |

**Critical optimizations**:
- **Single draw call**: `InstancedMesh` renders all blades in one call. Never create individual meshes.
- **frustumCulled = false**: Wind displacement pushes blades outside bounding box. Disable frustum culling or expand bounds manually.
- **Geometry reuse**: One `BladeGeometry` shared across all instances per LOD ring.
- **Avoid per-frame JS loops over instances**: All animation happens in shaders via uniforms (time, wind). Instance data is static after placement.
- **Alpha test over alpha blend**: `alphaTest: 0.1` avoids costly transparent sorting. Use distance-fade `discard` in fragment shader.
- **Shadow casting**: Grass rarely needs to cast shadows. Skip `castShadow` for massive performance gain. If needed, use a simplified shadow-only mesh.

## Common Pitfalls

1. **Black grass / no lighting**: Grass uses `ShaderMaterial` which bypasses scene lights. Lighting is computed manually in the fragment shader. Ensure `sunDir`, `sunColor`, and `ambientColor` uniforms are set.
2. **Blades all face same direction**: Each instance needs a unique rotation in `aPositionRotation.w`. Random rotation + some alignment to wind direction looks natural.
3. **Z-fighting with ground plane**: Offset blade root Y slightly above ground (0.01 units) or use `polygonOffset` on ground material.
4. **Popping during LOD transitions**: Use alpha-based distance fade rather than abrupt show/hide. Overlap LOD ring boundaries.
5. **Wind looks mechanical**: Layer multiple frequencies. Add per-blade phase offset via hash of position. Vary gust speed over time.

## References

- `references/blade-shaders.md` — Complete GLSL vertex/fragment shaders for wind, SSS, interaction displacement, and WGSL compute placement.
- `references/grass-types.md` — Detailed species profiles with blade dimensions, color palettes, density settings, and biome associations.
