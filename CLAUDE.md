# Aethlands

Interactive 3D floating island viewer with Ghibli-inspired aesthetics.

## Architecture

Single-file app (`index.html`, ~1960 lines). Everything — HTML, CSS, and JS — lives in one file. No build step, no bundler.

### Tech Stack
- **Three.js** (v0.163.0) via CDN import map
- **OrbitControls** for camera interaction
- **DM Sans** font from Google Fonts
- Deployed on **Vercel** as a static site

### Structure within index.html
1. **Styles** (lines 7-52): Dark theme, glassmorphism nav buttons, DM Sans typography
2. **Biome Presets** (~line 71): `LANDS` array — 7 biomes with full color palettes and feature flags
3. **Scene Setup**: Three.js renderer, camera, OrbitControls, lighting
4. **Terrain Generation**: Procedural floating island geometry per biome
5. **Decorations**: Trees (multiple styles), bushes, flowers, rocks, crystals, coral, mushrooms
6. **Particles/FX**: Butterflies, falling leaves, glow spots, lava, water foam
7. **Animation Loop** (~line 1870): Water waves, cloud drift, particle movement, butterfly flight
8. **UI**: Bottom-fixed biome selector buttons

### Biomes
Meadow, Mushroom Grove, Crystal Peaks, Autumn Vale, Coral Atoll, Spirit Forest, Volcanic

Each biome is a config object with: color palette, sky gradient, fog color, feature flags (`hasTrees`, `hasFlowers`, `hasSnow`, etc.), tree style, and counts.

## Key Patterns
- All 3D objects are procedurally generated (no external models)
- Colors are defined per-biome in the `LANDS` array — always use this pattern when adding biomes
- Animated elements use `userData` properties (`baseY`, `phase`, `speed`, etc.) set at creation time
- Water uses custom vertex shader animation for 3D wave effect
- Outline/toon aesthetic via `MeshToonMaterial` with outline pass

## Development
- Just open `index.html` in a browser or run any local server
- No dependencies to install, no build step
- Deploy: `vercel --prod` or push to GitHub (Vercel auto-deploys)

## Style Guidelines
- Keep everything in the single index.html file
- Maintain the Ghibli-inspired soft color palette aesthetic
- New biomes should follow the existing `LANDS` config object shape
- Glassmorphism UI pattern for any new controls
