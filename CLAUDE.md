# Aethlands

Interactive 3D floating island viewer with Ghibli-inspired aesthetics. Procedurally generated terrain, vegetation, water, and atmosphere — all in a single HTML file.

## Architecture

Single-file app (`index.html`, ~1958 lines). HTML + CSS + JS in one file. No build step, no bundler, no external models.

### Tech Stack
- **Three.js** v0.163.0 via CDN import map (ES modules)
- **OrbitControls** for camera (damped, clamped polar angle, 5-18 distance range)
- **DM Sans** font from Google Fonts
- Deployed on **Vercel** as a static site
- GitHub repo: `hunterbastian/aeth-lands`

### Code Map (line ranges approximate)
| Section | Lines | Description |
|---------|-------|-------------|
| CSS/HTML | 1-65 | Dark theme (#1a1a2e), glassmorphism buttons, DM Sans |
| `LANDS[]` biome configs | 71-253 | 7 biome presets with full color palettes + feature flags |
| Three.js setup | 255-321 | Renderer (ACES, PCFSoft shadows, 2048 shadow map), camera (38 FOV), 4-light rig |
| Toon gradients | 277-291 | `createToonGradient()` — 3/4/5/6 step gradients with configurable softness |
| Noise functions | 323-369 | Seeded PRNG, 2D value noise, FBM (4 octaves), ridge noise/FBM |
| Terrain height | 371-438 | `getTerrainHeight()` — 5 hills + ridge + valleys + cliff falloff at dist > 0.82 |
| Island creation | 440-1026 | `createIsland()` — terrain mesh, vertex colors, cliff base, vegetation, water, FX |
| Water system | 628-765 | Gerstner waves (4 wave sets), custom GLSL shader (fresnel, specular, foam, caustics) |
| Vegetation | 1028-1478 | Dispatcher + 5 tree types + bushes |
| Clouds | 1480-1529 | 12 clouds, each 4-8 deformed icosahedron puffs |
| Butterflies | 1531-1573 | Dual-wing particles with circular orbit paths |
| Biome transition | 1575-1634 | Animate-out (sink+spin) → rebuild → animate-in (ease-cubic rise) |
| Day/night cycle | 1636-1725 | ~80s full cycle, 10 TOD phases, smoothstep interpolation |
| Stars | 1727-1747 | 200 points in upper hemisphere, fade with nightFactor |
| Fireflies | 1749-1771 | 25 glowing spheres, drift + blink at night |
| Animation loop | 1809-1948 | Gerstner wave vertex update, flower sway, glow pulse, leaf fall, lava, clouds, butterflies |
| UI | 1790-1798 | Bottom-fixed button bar from LANDS array |

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
- `name`, `treeStyle` (round/mushroom/crystal/tall/coral)
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

### Water (Gerstner Wave System)
- 72x72 grid, clipped to circle (radius = RADIUS * 1.8)
- 4 overlapping Gerstner waves with direction, amplitude, frequency, speed, steepness
- Vertices updated every frame in JS (not GPU-only) — `origX/Y/Z` stored in `userData`
- Dampen factor near island (dist < 0.85 from center)
- Custom GLSL fragment shader: depth-based color, fresnel rim, sun specular (power 80), wave crest foam, animated shore foam band, caustic sparkle
- Shader uniforms updated per-frame: `uTime`, `uSunDir`, `uSunColor`

### Lighting Rig (4 lights + hemisphere)
- Ambient: soft fill
- Sun (directional): main light, 2048 shadow map, position orbits with day/night cycle
- Fill (directional): opposite side, subtle
- Rim (directional): backlit Ghibli glow
- Hemisphere: sky/ground color blend

### Day/Night Cycle
- `dayTime` advances by `CYCLE_SPEED * dt` (full cycle ~80s)
- 10 TOD phases (midnight → pre-dawn → dawn → morning → midday → golden → sunset → dusk → night → midnight)
- Each phase defines: sky gradient (3 colors), fog, sun color/intensity, ambient color/intensity, rim color/intensity, fill intensity
- Smoothstep interpolation between phases via `getTOD()`
- Sky rendered to 1x512 canvas texture (equirectangular mapping)
- `nightFactor` (0-1) controls star opacity and firefly visibility

### Toon/Outline Aesthetic
- `MeshToonMaterial` everywhere with custom gradient maps (3-6 steps, configurable softness)
- Outlines: duplicate geometry with `MeshBasicMaterial({ side: BackSide })`, scaled 1.01-1.06x
- Terrain outline shifted down 0.03 on Y

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

### Other Decorations
- **Bushes**: 2-4 deformed icosahedron puffs, similar to tree canopy but ground-level
- **Rocks**: Clustered 2-4 deformed icosahedrons, squat (Y * 0.8), mossy color variation
- **Flowers**: Clustered in patches (12-18 per patch), stem + icosahedron bloom, sway animation
- **Grass tufts**: 2-4 cone blades per tuft, 80-150 tufts depending on biome
- **Cliff roots**: 18 hanging vine/root cylinders with sine-wave bend on cliff face
- **Glow spots**: Pulsing transparent spheres (Mushroom Grove, Spirit Forest)
- **Floating leaves**: Plane geometry, drift + spin + fall (Autumn Vale)
- **Lava**: Animated opacity plane patches (Volcanic)

## Animation Details

All animated elements use `userData` properties set at creation:
- `isWaveWater` + `origX/Y/Z` + `waves[]` + `uniforms` → Gerstner wave vertex update
- `isFlower` + `baseY` + `phase` → gentle Y sway (sin, period ~2s)
- `isGlow` + `baseY` + `phase` → opacity pulse + scale pulse
- `isLeaf` + `baseY/X` + `phase` + `speed` → drift + fall + spin, reset Y when below 0
- `isLava` + `phase` → opacity pulse (fast, period ~3.5s)
- `isButterfly` + `centerX/Z` + `radius` + `speed` + `phase` → circular orbit + wing flap (8Hz)
- `isFirefly` + `baseX/Y/Z` + `radius` + `speed` + `phase` → drift + blink, opacity scaled by nightFactor
- `isStars` → opacity = nightFactor * 0.8, slow Y rotation
- Clouds: drift on X (wrap at x > 18 → -18), gentle Y bob
- Island group: gentle Y breathing (sin, 0.35Hz) + slight Y rotation

## Biome Transition
- `switchToLand(index)`: guards against re-selecting current or mid-transition
- Animate out: 0.026 step, sink Y (-2.5), spin, shrink to 0.8, fade cloud opacity
- At completion: destroy old groups, create new island/clouds/particles, update scene colors
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
- New biomes must follow the full `LANDS` config shape (all 28 color keys + all flags)
- All new geometry should use `MeshToonMaterial` + outline meshes
- Glassmorphism UI pattern (rgba background, backdrop-filter blur, rounded borders) for any new controls
- Use seeded randomness for deterministic results — set `seed` before each object category
- Organic shapes: always apply noise deformation to geometry vertices, never use pristine primitives
