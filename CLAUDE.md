# Aethlands

Interactive 3D floating island viewer with Ghibli-inspired aesthetics. Procedurally generated terrain, vegetation, water, and atmosphere — all in a single HTML file.

## Architecture

Single-file app (`index.html`, ~2963 lines). HTML + CSS + JS in one file. No build step, no bundler, no external models.

### Tech Stack
- **Three.js** v0.163.0 via CDN import map (ES modules)
- **Three.js addons**: OrbitControls, EffectComposer, RenderPass, UnrealBloomPass, ShaderPass
- **DM Sans** font from Google Fonts
- Deployed on **Vercel** as a static site
- GitHub repo: `hunterbastian/aeth-lands`

### Code Map (line ranges approximate)
| Section | Lines | Description |
|---------|-------|-------------|
| CSS/HTML | 1-104 | Dark theme (#1a1a2e), glassmorphism buttons, tooltips, screenshot flash |
| `LANDS[]` biome configs | 110-316 | 7 biome presets with full color palettes + feature flags + `desc` |
| Three.js + post-processing setup | 318-400 | Renderer (ACES, PCFSoft, 4096 shadow map, preserveDrawingBuffer), camera (38 FOV), EffectComposer pipeline |
| Toon gradients | ~400-420 | `createToonGradient()` — 3/4/5/6 step gradients with configurable softness |
| Lighting rig | ~420-470 | 5 lights (sun, fill, rim, ambient, bounce) + hemisphere light |
| Wind system | ~470-520 | `updateWind()`, `getWindAt()` — traveling wind field with spatial coherence |
| Noise functions | ~520-570 | Seeded PRNG, 2D value noise, FBM (4 octaves), ridge noise/FBM |
| Terrain height | ~570-640 | `getTerrainHeight()` — 5 hills + ridge + valleys + cliff falloff at dist > 0.82 |
| Island creation | ~640-1100 | `createIsland()` — terrain mesh, vertex colors, cliff base, vegetation, water, FX |
| Water system (SoT-style) | ~750-950 | 128x128 grid, 12 Gerstner waves, Jacobian foam, custom GLSL (fresnel, subsurface, multi-lobe specular, FBM foam) |
| Vegetation | ~1100-1550 | Dispatcher + 5 tree types + bushes (all wind-responsive) |
| Clouds | ~1550-1600 | 12 clouds, each 4-8 deformed icosahedron puffs |
| Butterflies | ~1600-1650 | Dual-wing particles with circular orbit paths |
| Biome transition | ~1650-1720 | Animate-out (sink+spin) → rebuild → animate-in (ease-cubic rise) |
| Day/night cycle | ~1720-1830 | ~80s full cycle, 10 TOD phases, smoothstep interpolation |
| Stars | ~1830-1850 | 200 points in upper hemisphere, fade with nightFactor |
| Fireflies | ~1850-1880 | 25 glowing spheres, drift + blink at night |
| Satellite islands | ~1880-1960 | 4 mini-islands orbiting the main island, rebuilt on biome switch |
| God rays | ~1960-2050 | Volumetric light scattering via custom radial blur shader pass |
| Cinematic camera | ~2050-2180 | Idle auto-orbit (15s timeout), tour mode cycling all biomes |
| Screenshot | ~2180-2250 | Camera icon button, PNG capture with flash animation |
| UI | ~2250-2320 | Bottom-fixed button bar + tooltip system + tour button |
| Animation loop | ~2320-2963 | Wave vertex update, wind response, flower sway, glow pulse, post-processing |

## Biomes

| Biome | treeStyle | Unique Features | mountainHeight |
|-------|-----------|-----------------|----------------|
| Meadow | `round` | butterflies, flowers | 1.8 |
| Mushroom Grove | `mushroom` | glow spots, spotted caps | 1.2 |
| Crystal Peaks | `crystal` | snow, translucent spires | 2.8 |
| Autumn Vale | `round` | falling leaves | 1.8 |
| Coral Atoll | `coral` | butterflies, branching coral | 0.9 |
| Spirit Forest | `tall` | glow spots, S-curve trunks, vines | 2.0 |
| Volcanic | `round` | lava glow, no trees/flowers | 2.2 |

### Biome Config Shape
Every biome in `LANDS[]` has:
- `name`, `desc` (tooltip description), `treeStyle` (round/mushroom/crystal/tall/coral)
- `colors{}`: 28 color keys — grass(4), dirt(3), sand(2), rock(3), snow, water(4), trunk(3), foliage(3), flowers(3), bush(2), cliff(3)
- `sky[]` (3 gradient stops), `skyBottom`, `fog`, `outline`
- Feature flags: `hasTrees`, `hasFlowers`, `hasSnow`, `hasRocks`, `hasBushes`
- Optional: `hasButterflies`, `hasGlowSpots`, `hasLeaves`, `hasLavaGlow`
- `mountainHeight`, `treeCount`, `bushCount`
- `cloudColor`, `cloudShadow`, `ambientColor`, `sunColor`

## Rendering Systems

### Terrain
- 80x80 subdivided plane, displaced by `getTerrainHeight()`
- Height = base plateau + FBM texture + 5 positioned hills + ridge + valleys
- Cliff edge: sharp falloff at dist > 0.82 from center (RADIUS = 2.8)
- Vertex colors: height-banded (sand → grass variants → dirt → rock → snow) with noise variation
- Cliff base: deformed cylinder with horizontal striations, layered earth vertex colors

### Water (Sea of Thieves-style)
- 128x128 grid, clipped to circle (radius = RADIUS * 1.8)
- 12 overlapping Gerstner waves with varied direction, amplitude, frequency, speed, steepness
- Jacobian determinant computed per-vertex for physically-based whitecap foam
- Vertices updated every frame in JS — `origX/Y/Z` stored in `userData`
- Dampen factor near island (dist < 0.85 from center)
- Custom GLSL fragment shader:
  - Depth-based color blend (shallow → deep)
  - Subsurface scattering approximation
  - Schlick fresnel with multi-lobe specular (power 80 + power 20)
  - Jacobian-driven whitecap foam with FBM noise texture
  - Animated shore foam band
  - Caustic sparkle
- Shader uniforms updated per-frame: `uTime`, `uSunDir`, `uSunColor`

### Post-Processing Pipeline
- **EffectComposer** chain: RenderPass → UnrealBloomPass → GodRayPass → ColorGradePass
- **Bloom**: strength 0.15, radius 0.4, threshold 0.85 (dynamic per TOD)
- **God rays**: Custom radial blur shader — samples from sun screen position, density/weight/decay tuned per TOD, strongest at dawn/sunset
- **Color grading**: Saturation boost (1.15), contrast (1.08), vignette (0.35), all dynamic per time of day
- `preserveDrawingBuffer: true` on renderer for screenshot support

### Lighting Rig (5 lights + hemisphere)
- Ambient: soft fill
- Sun (directional): main light, 4096×4096 shadow map with normalBias, position orbits with day/night cycle
- Fill (directional): opposite side, subtle
- Rim (directional): backlit Ghibli glow
- Bounce (point light): warm uplight under island
- Hemisphere: sky/ground color blend, tracks TOD phases

### Day/Night Cycle
- `dayTime` advances by `CYCLE_SPEED * dt` (full cycle ~80s)
- 10 TOD phases (midnight → pre-dawn → dawn → morning → midday → golden → sunset → dusk → night → midnight)
- Each phase defines: sky gradient (3 colors), fog, sun color/intensity, ambient color/intensity, rim color/intensity, fill intensity
- Dynamic bloom strength/threshold and vignette per TOD
- God ray intensity peaks at dawn/sunset
- Smoothstep interpolation between phases via `getTOD()`
- Sky rendered to 1x512 canvas texture (equirectangular mapping)
- `nightFactor` (0-1) controls star opacity and firefly visibility

### Toon/Outline Aesthetic
- `MeshToonMaterial` everywhere with custom gradient maps (3-6 steps, configurable softness)
- Outlines: duplicate geometry with `MeshBasicMaterial({ side: BackSide })`, scaled 1.01-1.06x
- Terrain outline shifted down 0.03 on Y

## Wind System
- Traveling wind field with spatial coherence across the island
- `updateWind(dt)`: updates global wind direction and strength over time
- `getWindAt(x, z)`: returns wind vector at any world position (used by vegetation)
- All vegetation tagged with `userData` properties: `isTree`, `isGrass`, `isBush`, `isFlower`
- Each has `windPhase`, `windFlex`, `baseRotX`, `baseRotZ` for natural sway response
- Trees flex with larger amplitude, grass/flowers with smaller, faster response

## Vegetation System

5 tree style functions dispatched by `createVegetation()`:
- **`createGhibliTree()`** (round): Root-flared trunk + 9 deformed icosahedron canopy puffs
- **`createMushroomTree()`**: Bulging stem + 3-5 spotted dome caps + mini mushrooms at base
- **`createCrystalSpire()`**: 2-4 twisted prismatic shafts (translucent) + octahedron debris
- **`createTallTree()`**: S-curved trunk with root flare + sparse layered canopy + hanging vines
- **`createCoralFormation()`**: 3-5 branching cylinders with bumpy surface + spherical tips

All vegetation uses:
- Seeded PRNG (`seed` reset per category) for deterministic placement
- Noise-based organic deformation on all geometry
- Height/distance checks: skip placement if too low, too high, or too close to edge
- Size classes: large/medium/small with random distribution
- Wind response via `userData` tags

### Other Decorations
- **Bushes**: 2-4 deformed icosahedron puffs, similar to tree canopy but ground-level
- **Rocks**: Clustered 2-4 deformed icosahedrons, squat (Y * 0.8), mossy color variation
- **Flowers**: Clustered in patches (12-18 per patch), stem + icosahedron bloom, sway animation
- **Grass tufts**: 2-4 cone blades per tuft, 80-150 tufts depending on biome
- **Cliff roots**: 18 hanging vine/root cylinders with sine-wave bend on cliff face
- **Glow spots**: Pulsing transparent spheres (Mushroom Grove, Spirit Forest)
- **Floating leaves**: Plane geometry, drift + spin + fall (Autumn Vale)
- **Lava**: Animated opacity plane patches (Volcanic)

## Satellite Islands
- 4 mini floating islands orbiting the main island at different radii and speeds
- Each has simplified terrain (40x40 grid), a few trees, and cliff base
- Orbit at varying heights with gentle Y bob
- Rebuilt when biome switches (use current biome's color palette)
- Smaller scale (0.15-0.25x) with matching toon materials

## God Rays (Volumetric Light Scattering)
- Sun disk mesh positioned far along sun direction vector
- Custom ShaderPass with radial blur sampling (30 samples)
- Screen-space effect: projects sun position to screen coordinates
- Parameters: density 0.96, weight 0.25, decay 0.97, exposure 0.22
- Intensity peaks during dawn/sunset phases, fades at night/midday
- Blended additively with the scene

## Cinematic Camera System
- **Idle detection**: After 15s of no user interaction, enters cinematic mode
- **Auto-orbit**: Slow orbit around the island with gentle height variation
- **Tour mode**: Button to cycle through all 7 biomes with smooth camera transitions
- **Tour indicator**: "CINEMATIC" label shown during tour mode
- Any user interaction (mouse/touch/keyboard) exits cinematic mode immediately
- Camera smoothly interpolates back to OrbitControls target on exit

## Screenshot System
- Camera icon button (top-right corner)
- Captures current frame as PNG using `renderer.domElement.toDataURL()`
- Full-screen white flash animation on capture
- Auto-downloads as `aethlands-{timestamp}.png`
- Works because renderer has `preserveDrawingBuffer: true`

## Microinteractions
- **Water hover ripple**: Mouse position tracked via raycaster, creates ripple effect on water surface
- **Button entrance animation**: Biome buttons animate in with staggered delay on page load
- **Biome tooltips**: Hover over biome buttons shows description tooltip (safe DOM construction, no innerHTML)

## Animation Details

All animated elements use `userData` properties set at creation:
- `isWaveWater` + `origX/Y/Z` + `waves[]` + `uniforms` → Gerstner wave vertex update + Jacobian foam
- `isFlower` + `baseY` + `phase` → gentle Y sway (sin, period ~2s) + wind response
- `isGlow` + `baseY` + `phase` → opacity pulse + scale pulse
- `isLeaf` + `baseY/X` + `phase` + `speed` → drift + fall + spin, reset Y when below 0
- `isLava` + `phase` → opacity pulse (fast, period ~3.5s)
- `isButterfly` + `centerX/Z` + `radius` + `speed` + `phase` → circular orbit + wing flap (8Hz)
- `isFirefly` + `baseX/Y/Z` + `radius` + `speed` + `phase` → drift + blink, opacity scaled by nightFactor
- `isStars` → opacity = nightFactor * 0.8, slow Y rotation
- `isTree/isBush/isGrass` + `windPhase` + `windFlex` + `baseRotX/Z` → wind sway response
- Clouds: drift on X (wrap at x > 18 → -18), gentle Y bob, wind influence
- Island group: gentle Y breathing (sin, 0.35Hz) + slight Y rotation
- Satellite islands: orbital motion + Y bob

## Biome Transition
- `switchToLand(index)`: guards against re-selecting current or mid-transition
- Animate out: 0.026 step, sink Y (-2.5), spin, shrink to 0.8, fade cloud opacity
- At completion: destroy old groups, create new island/clouds/particles/satellites, update scene colors
- Animate in: 0.022 step, cubic ease-out rise from -2.5 to 0, scale 0.8 → 1.0
- Active button gets `.active` class (brighter glassmorphism)

## Noise & Randomness
- `seededRandom()`: LCG PRNG (multiplier 16807, modulus 2^31-1), `seed` reset per object category
- `noise2D()`: Hash-based 2D value noise with smoothstep interpolation
- `fbm()`: 4-octave fractional Brownian motion (amp * 0.5, freq * 2.1 per octave)
- `ridgeNoise()` / `ridgeFbm()`: Absolute-value ridge variant for mountain drama
- Seed values: terrain=42, trees=100, bushes=150, flowers=200, rocks=300, grass=400, clouds=500, glow=600, leaves=700, lava=800, cliff vegetation=900, butterflies=1000

## Development
- Open `index.html` in browser or any local server — zero setup
- Deploy: push to GitHub (Vercel auto-deploys) or `vercel --prod`
- All state is in module-scope variables (no classes, no framework)

## Style Guidelines
- Keep everything in the single `index.html` file
- Maintain the Ghibli-inspired soft color palette — no harsh/neon colors
- New biomes must follow the full `LANDS` config shape (all 28 color keys + all flags + `desc`)
- All new geometry should use `MeshToonMaterial` + outline meshes
- Glassmorphism UI pattern (rgba background, backdrop-filter blur, rounded borders) for any new controls
- Use seeded randomness for deterministic results — set `seed` before each object category
- Organic shapes: always apply noise deformation to geometry vertices, never use pristine primitives
- All vegetation must respond to the wind system (tag with appropriate `userData` properties)
- Post-processing: any new visual effects should integrate into the EffectComposer pipeline
