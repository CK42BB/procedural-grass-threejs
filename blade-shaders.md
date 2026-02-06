# Blade Shaders Reference

Complete shader implementations for grass blade rendering including wind displacement,
subsurface scattering, interactive push, and GPU compute placement.

## GLSL Vertex Shader

Full vertex shader with multi-layer wind, interactive displacement, and instance transforms.

```glsl
// grass.vert
precision highp float;

// Per-vertex
attribute vec3 position;
attribute vec3 normal;
attribute vec2 uv;

// Per-instance
attribute vec4 aPositionRotation; // xyz = world pos, w = Y rotation
attribute vec4 aScaleVariation;   // x = scaleX, y = scaleY, z = tilt, w = colorVar

// Uniforms
uniform mat4 projectionMatrix;
uniform mat4 modelViewMatrix;
uniform mat4 modelMatrix;
uniform vec3 cameraPosition;

// Wind
uniform float windTime;
uniform vec2  windDir;
uniform float windBase;
uniform float windGust;
uniform float windGustFreq;

// Interaction (up to 4 pushers)
uniform vec3  pushPositions[4];
uniform float pushRadii[4];

// LOD fade
uniform float fadeStart;
uniform float fadeEnd;

varying vec2  vUv;
varying vec3  vWorldPos;
varying vec3  vNormal;
varying float vColorVar;
varying float vFade;

// --- Noise utilities (GPU-friendly) ---
float hash(vec2 p) {
  return fract(sin(dot(p, vec2(127.1, 311.7))) * 43758.5453);
}

float noise2D(vec2 p) {
  vec2 i = floor(p);
  vec2 f = fract(p);
  f = f * f * (3.0 - 2.0 * f);
  float a = hash(i);
  float b = hash(i + vec2(1.0, 0.0));
  float c = hash(i + vec2(0.0, 1.0));
  float d = hash(i + vec2(1.0, 1.0));
  return mix(mix(a, b, f.x), mix(c, d, f.x), f.y);
}

// --- Wind computation ---
vec3 computeWind(vec3 worldPos, float heightFactor) {
  // Layer 1: Global sway (low frequency)
  float globalPhase = dot(worldPos.xz, windDir) * 0.5 + windTime * 1.2;
  vec2 globalSway = windDir * sin(globalPhase) * windBase;

  // Layer 2: Gust waves (medium frequency, rolling across field)
  float gustPhase = dot(worldPos.xz, windDir) * windGustFreq + windTime * 2.5;
  float gustEnvelope = noise2D(worldPos.xz * 0.02 + windTime * 0.3);
  gustEnvelope = smoothstep(0.3, 0.7, gustEnvelope);
  vec2 gustSway = windDir * sin(gustPhase) * windGust * gustEnvelope;

  // Layer 3: Turbulence (high frequency, per-blade)
  float bladeHash = hash(worldPos.xz * 10.0);
  float turbPhase = windTime * 3.0 + bladeHash * 6.28;
  vec2 turbulence = vec2(sin(turbPhase), cos(turbPhase * 0.7)) * 0.1;

  // Combine — scale by height² so roots stay fixed
  float h2 = heightFactor * heightFactor;
  return vec3((globalSway + gustSway + turbulence).x, 0.0,
              (globalSway + gustSway + turbulence).y) * h2;
}

// --- Interactive displacement ---
vec3 computePush(vec3 worldPos, float heightFactor) {
  vec3 totalPush = vec3(0.0);

  for (int i = 0; i < 4; i++) {
    if (pushRadii[i] <= 0.0) continue;
    vec3 delta = worldPos - pushPositions[i];
    delta.y = 0.0; // Only push horizontally
    float dist = length(delta);
    if (dist < pushRadii[i] && dist > 0.001) {
      float strength = 1.0 - smoothstep(0.0, pushRadii[i], dist);
      strength *= strength; // Quadratic falloff
      totalPush += normalize(delta) * strength * 1.5;
    }
  }

  return totalPush * heightFactor * heightFactor;
}

void main() {
  vUv = uv;
  vColorVar = aScaleVariation.w;

  // Instance transform
  vec3 pos = position;

  // Apply blade scale
  pos.x *= aScaleVariation.x;
  pos.y *= aScaleVariation.y;

  // Apply tilt
  float tilt = aScaleVariation.z;
  float cosT = cos(tilt);
  float sinT = sin(tilt);
  float tilted_y = pos.y * cosT - pos.z * sinT;
  float tilted_z = pos.y * sinT + pos.z * cosT;
  pos.y = tilted_y;
  pos.z = tilted_z;

  // Apply Y rotation
  float rot = aPositionRotation.w;
  float cosR = cos(rot);
  float sinR = sin(rot);
  vec3 rotated;
  rotated.x = pos.x * cosR - pos.z * sinR;
  rotated.y = pos.y;
  rotated.z = pos.x * sinR + pos.z * cosR;

  // World position (before wind)
  vec3 worldPos = rotated + aPositionRotation.xyz;

  // Apply wind and interaction
  float heightFactor = uv.y; // 0 at root, 1 at tip
  vec3 windOffset = computeWind(worldPos, heightFactor);
  vec3 pushOffset = computePush(worldPos, heightFactor);
  worldPos += windOffset + pushOffset;

  vWorldPos = worldPos;

  // Recompute normal with wind displacement
  vNormal = normalize(normal);
  // Approximate bent normal — tilt normal by wind amount
  vNormal.xz += (windOffset.xz + pushOffset.xz) * 0.3;
  vNormal = normalize(vNormal);

  // LOD distance fade
  float dist = distance(cameraPosition, worldPos);
  vFade = 1.0 - smoothstep(fadeStart, fadeEnd, dist);

  gl_Position = projectionMatrix * modelViewMatrix * vec4(worldPos, 1.0);
}
```

## GLSL Fragment Shader

Handles SSS approximation, color variation, AO, and distance fade.

```glsl
// grass.frag
precision highp float;

uniform vec3  baseColor;
uniform vec3  tipColor;
uniform vec3  dryColor;
uniform float dryAmount;
uniform float sssStrength;
uniform float aoStrength;
uniform vec3  sunDir;
uniform vec3  sunColor;
uniform vec3  ambientColor;
uniform vec3  cameraPosition;

varying vec2  vUv;
varying vec3  vWorldPos;
varying vec3  vNormal;
varying float vColorVar;
varying float vFade;

void main() {
  if (vFade < 0.01) discard;

  float heightT = vUv.y;

  // --- Base color ---
  vec3 color = mix(baseColor, tipColor, heightT);

  // Per-instance variation: slight hue/saturation shift
  color = mix(color, dryColor, vColorVar * dryAmount);

  // Random darkening/lightening
  float varShift = (vColorVar - 0.5) * 0.15;
  color *= 1.0 + varShift;

  // --- Ambient occlusion at roots ---
  float ao = mix(1.0 - aoStrength, 1.0, smoothstep(0.0, 0.3, heightT));
  color *= ao;

  // --- Lighting ---
  vec3 N = normalize(vNormal);
  vec3 L = normalize(sunDir);
  vec3 V = normalize(cameraPosition - vWorldPos);

  // Diffuse (wrap lighting for softness)
  float NdotL = dot(N, L);
  float diffuse = max(NdotL * 0.5 + 0.5, 0.0); // Half-Lambert wrap

  // --- Subsurface scattering approximation ---
  // When light comes from behind the blade, it glows through
  float sss = max(dot(-V, L), 0.0);
  sss = pow(sss, 3.0) * sssStrength;
  // Modulate SSS by height (tips scatter more light)
  sss *= heightT;

  // --- Specular (subtle sheen on blade surface) ---
  vec3 H = normalize(L + V);
  float spec = pow(max(dot(N, H), 0.0), 32.0) * 0.15;

  // Combine lighting
  vec3 lit = color * (sunColor * diffuse + ambientColor * 0.5) + sunColor * sss + sunColor * spec;

  // Distance fade
  gl_FragColor = vec4(lit, vFade);
}
```

## Embedding Shaders in JavaScript

Store shader strings as template literals in your module:

```javascript
const GRASS_VERT = `
  // ... paste vertex shader above ...
`;

const GRASS_FRAG = `
  // ... paste fragment shader above ...
`;

// Use in ShaderMaterial
const material = new THREE.ShaderMaterial({
  vertexShader: GRASS_VERT,
  fragmentShader: GRASS_FRAG,
  uniforms: { /* ... */ },
});
```

Or load from `.glsl` files at build time using a bundler plugin (vite-plugin-glsl, etc.).

## WGSL Compute Shader — GPU Placement

Place grass instances entirely on the GPU by reading a heightmap texture and writing
to a storage buffer.

```wgsl
// grass_placement.wgsl
struct BladeInstance {
  posRot: vec4<f32>,    // xyz = position, w = rotation
  scaleVar: vec4<f32>,  // x = scaleX, y = scaleY, z = tilt, w = colorVar
}

@group(0) @binding(0) var heightmap: texture_2d<f32>;
@group(0) @binding(1) var<storage, read_write> blades: array<BladeInstance>;
@group(0) @binding(2) var<storage, read_write> counter: atomic<u32>;

struct PlacementParams {
  terrainSize: f32,
  maxHeight: f32,
  gridStep: f32,
  gridCountX: u32,
  minNormalizedHeight: f32,
  maxSlopeAngle: f32,
  maxBlades: u32,
  seed: f32,
}
@group(0) @binding(3) var<uniform> params: PlacementParams;

fn hash2(p: vec2<f32>) -> f32 {
  return fract(sin(dot(p, vec2<f32>(127.1, 311.7))) * 43758.5453);
}

fn hashVec(p: vec2<f32>) -> vec2<f32> {
  return vec2<f32>(
    fract(sin(dot(p, vec2<f32>(127.1, 311.7))) * 43758.5453),
    fract(sin(dot(p, vec2<f32>(269.5, 183.3))) * 43758.5453)
  );
}

fn sampleHeight(uv: vec2<f32>) -> f32 {
  let dim = textureDimensions(heightmap);
  let coord = vec2<u32>(
    clamp(u32(uv.x * f32(dim.x)), 0u, dim.x - 1u),
    clamp(u32(uv.y * f32(dim.y)), 0u, dim.y - 1u)
  );
  return textureLoad(heightmap, coord, 0).r;
}

@compute @workgroup_size(64)
fn main(@builtin(global_invocation_id) gid: vec3<u32>) {
  let gridX = gid.x % params.gridCountX;
  let gridZ = gid.x / params.gridCountX;
  if (gridZ >= params.gridCountX) { return; }

  let halfSize = params.terrainSize * 0.5;
  let cellSeed = vec2<f32>(f32(gridX) + params.seed, f32(gridZ) + params.seed);
  let jitter = hashVec(cellSeed) - 0.5;

  let x = -halfSize + (f32(gridX) + jitter.x) * params.gridStep;
  let z = -halfSize + (f32(gridZ) + jitter.y) * params.gridStep;

  let nx = x / params.terrainSize + 0.5;
  let nz = z / params.terrainSize + 0.5;
  if (nx < 0.0 || nx > 1.0 || nz < 0.0 || nz > 1.0) { return; }

  let h = sampleHeight(vec2<f32>(nx, nz));
  if (h < params.minNormalizedHeight) { return; }

  // Slope check
  let eps = params.gridStep / params.terrainSize;
  let hx = sampleHeight(vec2<f32>(nx + eps, nz));
  let hz = sampleHeight(vec2<f32>(nx, nz + eps));
  let slope = atan(sqrt(pow(hx - h, 2.0) + pow(hz - h, 2.0)) * params.maxHeight / params.gridStep);
  if (slope > params.maxSlopeAngle) { return; }

  // Density modulation (simple noise)
  let densityN = hash2(cellSeed * 0.05);
  if (densityN < 0.3) { return; }

  // Claim an output slot atomically
  let idx = atomicAdd(&counter, 1u);
  if (idx >= params.maxBlades) { return; }

  let y = h * params.maxHeight;
  let rotation = hash2(cellSeed * 3.0) * 6.283185;
  let scaleX = 0.7 + hash2(cellSeed * 5.0) * 0.6;
  let scaleY = 0.6 + hash2(cellSeed * 7.0) * 0.8;
  let tilt = (hash2(cellSeed * 11.0) - 0.5) * 0.3;
  let colorVar = hash2(cellSeed * 13.0);

  blades[idx].posRot = vec4<f32>(x, y, z, rotation);
  blades[idx].scaleVar = vec4<f32>(scaleX, scaleY, tilt, colorVar);
}
```

### Dispatching the Compute Shader

```javascript
async function computeGrassPlacement(renderer, heightmapTexture, params) {
  const gridCount = Math.ceil(params.terrainSize / params.gridStep);
  const totalCells = gridCount * gridCount;
  const workgroups = Math.ceil(totalCells / 64);

  // Create storage buffer for blade instances
  const maxBlades = params.maxBlades ?? 500000;
  const bladeBuffer = new THREE.StorageBufferAttribute(
    new Float32Array(maxBlades * 8), 8 // 2 × vec4
  );

  // Dispatch compute
  // ... (bind heightmap, uniform buffer, storage buffer, dispatch workgroups)

  return bladeBuffer;
}
```

## TSL Wind Node (WebGPU)

Wind computation expressed in Three.js Shading Language for WebGPU node materials.

```javascript
import { Fn, attribute, float as tslFloat, sin, cos, dot, vec2, vec3,
         normalize as tslNorm, smoothstep, uniform, positionWorld,
         hash } from 'three/tsl';

const windNode = Fn(({ worldPos, heightFactor, time, windDir, windBase, windGust }) => {
  // Global sway
  const globalPhase = dot(worldPos.xz, windDir).mul(0.5).add(time.mul(1.2));
  const globalSway = windDir.mul(sin(globalPhase)).mul(windBase);

  // Gust
  const gustPhase = dot(worldPos.xz, windDir).mul(0.3).add(time.mul(2.5));
  const gustSway = windDir.mul(sin(gustPhase)).mul(windGust).mul(0.5);

  // Height modulation
  const h2 = heightFactor.mul(heightFactor);
  const windXZ = globalSway.add(gustSway).mul(h2);

  return vec3(windXZ.x, tslFloat(0), windXZ.y);
});
```

## Advanced: Bent Normal Technique

For more accurate lighting on wind-displaced blades, recompute the normal
based on the displacement derivative rather than approximating:

```glsl
// In vertex shader, after computing wind displacement
// Compute tangent of displaced blade curve
vec3 posCurrent = worldPos;
vec3 posBelow = worldPos;
posBelow.y -= 0.1 * aScaleVariation.y; // Slightly lower on blade

// Approximate wind at lower point
float lowerHeight = max(heightFactor - 0.1, 0.0);
vec3 windBelow = computeWind(posBelow, lowerHeight);
posBelow += windBelow;

vec3 bladeTangent = normalize(posCurrent - posBelow);
// Normal is perpendicular to tangent in the blade plane
vec3 bladeNormal = normalize(cross(bladeTangent, vec3(sin(rot), 0.0, cos(rot))));
vNormal = bladeNormal;
```

## Shader Uniform Update Pattern

Keep shader updates efficient by batching uniform writes:

```javascript
function updateGrassUniforms(material, wind, camera, pushers) {
  const u = material.uniforms;
  u.windTime.value = wind.time;
  u.windDir.value.copy(wind.direction);
  u.windBase.value = wind.baseStrength;
  u.windGust.value = wind.gustStrength;
  u.windGustFreq.value = wind.gustFrequency;
  u.cameraPos.value.copy(camera.position);

  for (let i = 0; i < 4; i++) {
    if (pushers[i]) {
      u.pushPositions.value[i].copy(pushers[i].position);
      u.pushRadii.value[i] = pushers[i].radius;
    } else {
      u.pushRadii.value[i] = 0;
    }
  }
}
```
