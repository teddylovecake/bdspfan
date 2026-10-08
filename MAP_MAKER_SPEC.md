# Map Maker rework spec (paste this into your local Claude session)

## Task

Rework the **Map Maker** editor window in this opendpr Unity project (Unity 2019.4.27f1). It's the window docked next to the Inspector, version v0.3. Its toolbar has Walking, Undo, Redo, Open and Save, plus a "Play a safe copy of the game" button. Find its scripts first. They sit next to the `Playtest` and `OpenDPR` menu code.

The goal is a map maker that works like the tool in the reference video ("procedurally generated pokemon town tools", a Gen 3-style "Pokemon Paint" app). The video is 2D, but here everything must be built from **the real BDSP assets already ripped into this project** (`Assets/Resources/...`): grass, tall grass, water, trees, flowers, sand, snow, rock, cliffs and mountains, buildings, bridges, signs and cave entrances. Anything you place must come out complete and finished. For example, a mountain gets its side walls, top surface, corners and height, not a single flat tile.

Before you write code:
1. Read the existing Map Maker scripts and summarise how they work now: play-mode editing, save format, tools, Undo/Redo.
2. Search `Assets/Resources` for the BDSP field prefabs, meshes and materials for each element in the **Asset catalogue** section below. Write down the paths you find in `MapMakerAssetCatalog` (see below). Don't guess at paths.
3. Fix the console spam `Material doesn't have a texture property '_MainTex'`. Something calls `material.mainTexture` on BDSP shaders that don't have `_MainTex`. Check `material.HasProperty("_MainTex")` and fall back to the shader's real base map property (look the property up on the shader). Don't just read `.mainTexture`.

Keep the current safe-copy play mode (the user's real save is never touched), and keep Walking, Undo/Redo and Open/Save working throughout.

---

## What the reference video shows, in order

The window layout: a menu bar (File, Edit, View, Layer, Help), a tool column on the left, the map canvas in the middle, a tile palette along the bottom, and on the right a **Town Map** preview, a **Layers** panel and a context panel that changes with the tool (**Route Generator** or **House**). A one-line hint at the bottom of the right panel tells you what to do with the current tool.

1. **Start full, then carve.** A new map is completely filled with trees. Hint: *"Fill the canvas, then carve your game into it."* A round **carve brush** with a dashed circle outline (drag to resize) cuts open grass areas. The tree border around each cut fixes itself automatically, with proper edge trees and no half trees.
2. **Edge exits.** Each side of the canvas has a small control:
   - the direction letter (N/E/S/W)
   - a **connection type label** you can click to cycle it: *Land*, then *Sand*, and so on
   - an **offset number with + and −** that slides the exit along that edge
   - a **↕ button** that turns an exit on or off on that side

   When you turn on N, a path is carved from the north edge into the clearing. Hint: *"Drag the arrow; the path follows."* After that: *"Drag an edge arrow; click to change flow."* You can drag an exit arrow along its edge and the path re-routes live. Clicking an arrow changes which way the route flows.
3. **Route Generator panel.**
   - Buttons: **Fill, Flow, Terrain, Colors, Trees, Path**.
   - Settings: **Palette** dropdown (e.g. "Calm"), **Width** (path width, e.g. 2), **Center** dropdown ("First side", N, E, S, W: what the path network bends around), **Curviness** slider (50%) and **Width wobble** slider (0%).
   - **Terrain** makes new ground detail while keeping the route and palette. Hint: *"New terrain; route and palette kept."* It scatters flowers, small bushes, tall grass patches and a trainer or item.
   - **Colors** cycles seasonal or biome looks and shows the name: *"Pale winter"* (snowy blue-white trees and ground), *"Quiet woodland"* (normal green), and so on.
   - **Trees** re-rolls the mix of tree types.
   - **Path** redraws the path with the current Width, Curviness and Wobble.
   - **Fill** refills the map, and **Flow** re-runs exit routing.
4. **House / building tool** (house icon).
   - Drag a rectangle on the map (yellow dashed box) and a full building is generated to fit it: roof, walls, windows, base and a door.
   - When a building is selected it shows **four arrows (up, down, left, right)**. Dragging them resizes it: up adds floors (a tall apartment block), and left or right changes the width. The building re-generates correctly at every size.
   - The **House panel** has one row per part: **Family, Roof, Wall, Base, Columns, Door, Door frame, Door header** (and more if you scroll). Each row has a *style index* field, a *colour swatch* and a *colour index*.
   - Changing the values swaps parts live: flat blue roof versus red pitched roof, white panel walls versus tan brick versus purple brick, different doors.
   - Two buttons at the bottom: **randomise** the style, and **apply to all / copy**.
   - Dragging a building moves it, and the trees around it re-flow.
5. **Path material by edge type.** Setting an edge label to **Sand** turns that route's path into sand or light ground. The label on the opposite edge updates to match.
6. **Fill and tint.** The paint bucket plus the colour swatches in the left column (brown, green, gold, rust, pink, purple, light blue, pale) re-tint an area of ground.
7. **Cliffs, ledges and rivers.** You choose the cliff/rock tile and drag a diagonal band. It builds stepped **cliff faces with tops, sides and corners**, and a **river** follows along the bottom of the cliff on its own, with proper banks.
8. **Bridge tool** (bridge icon). You drag across the river and a wooden plank bridge is placed, with posts and ends. The path joins up on both sides.
9. **Left tool column:** rectangle select, brush/pencil, fill bucket, eyedropper, house, bridge, stairs/ledge, zoom, three Poké Ball-style buttons (quick-generate presets: small, medium and large town or route), a second select/marquee and a "!" button (sign/event or help). Below those are the colour swatches, two brush sizes and a brush-shape button.
10. **Bottom tile palette.** The big swatch on the left shows the current tile (foreground and background, like MS Paint). Then, left to right: rock/mountain top, grass, snow, sand, water, deep water, still pond water, tall (encounter) grass, tree, flower patch, sign, rock in water, rocky cliff, ledge cliff with dirt base, small bush (cuttable tree).
11. **Right panels.**
    - **Town Map:** an overview of the region showing where this map sits.
    - **Layers:** "Layer 1: Paint" with an eye toggle, a thumbnail, and add and delete buttons.

---

## What to build (BDSP version)

### Core
- A **grid-based 3D map** with a 1-unit cell size matching BDSP field grids. Each cell stores: ground type, height level, an object (tree, flower, sign and so on), and a BDSP `AttributeData.MapAttribute` value so walking and encounters work. Examples: `MATTR_ZIMEN_01` for ground, `MATTR_E_KUSA_*` for tall grass, `MATTR_UMI_01`/`MATTR_POOL_*` for water, `MATTR_SUNA_01` for sand, `MATTR_ICE_01` for ice, `MATTR_JUMP_*` for ledges.
- **Auto-tiling everywhere.** Each placement checks its 8 neighbours and picks the correct BDSP piece:
  - water: edge, inner corner and outer corner shore pieces
  - grass to sand or path borders
  - forest edge trees
  - cliff tops, walls and corners
- **Undo/Redo** for every action, as single steps (one brush stroke is one undo).
- **Layers** with show/hide, add and delete: Ground, Objects, Buildings, Events/Warps.
- **Live in play mode** on the existing safe copy, with Tab switching to Walking to test.

### Tools (all of them)
1. **Carve / Fill brush:** a round brush with an adjustable radius (drag or scroll) and a dashed circle preview. New Map = all trees, and carve opens space.
2. **Paint brush:** two brush sizes, a cross/shape option and a rectangle fill.
3. **Fill bucket and tint:** flood fill a ground type, plus colour swatches for tint.
4. **Eyedropper:** picks the tile or object under the cursor.
5. **Select / move / copy / paste / delete** with rectangle select.
6. **Tile palette (bottom bar):** every element in the catalogue below, each with a thumbnail taken from the real asset.
7. **Edge exits:** N, E, S and W controls with on/off, connection type (Land / Sand / Water / Cave / Gate), an offset with +/− and a draggable arrow. Paths re-route live. Each exit **links to a target map and warp**.
8. **Route Generator:** Fill, Flow, Terrain, Colors, Trees and Path, with Palette, Width, Center, Curviness and Width wobble, exactly as in the video. Colors offers BDSP season and biome sets, such as Sinnoh route green, Snowpoint winter, Pastoria swamp, beach sand and Mt. Coronet rock. Terrain keeps the route and palette.
9. **Mountain / cliff tool:**
   - Drag out an area and pick a **height** (number of levels). It builds the full mountain: top surface, side walls on every side, corners and stepped levels, using BDSP cliff and rock pieces.
   - Drag the edges to make it **bigger**. Use +/− or drag up to make it **taller**.
   - Options: add **stairs**, add **ledges** (one-way `MATTR_JUMP_*` in the chosen direction) and add a **river along the base**.
10. **Water tool:** sea, pond and river with shores built on their own, plus shallow water. Rivers can be drawn as a path with width.
11. **Bridge tool:** drag across water or a gap and get a full bridge with ends, joined to the path on both sides and walkable.
12. **Building tool:**
    - Drag a footprint, then use the 4 resize arrows. Up adds floors.
    - House panel rows for Family (Pokémon Center, Mart, house, gym, lab, apartment), Roof, Wall, Base, Columns, Door, Door frame and Door header. Each row has a style index plus a colour index or swatch.
    - Randomise and Copy-to-all buttons. Buildings are made from BDSP building prefabs or parts.
    - **Every door is a warp** (see below).
13. **Cave entrance tool:** place a BDSP cave mouth on a cliff or mountain wall. It snaps to the wall face and cuts the wall to fit. It creates a warp (see below).
14. **Warp / connection tool:**
    - Click any door, cave entrance, stairs or edge exit to open a warp editor: target **ZoneID**, target **warp index** (a list of the target map's warps), **arrival facing / InputDir**, optional flag, `CallLabel` script and `ExitLabel`.
    - Store these using the same fields as `XLSXContent.MapWarp.SheetData`: Position, GruopID, Size, WarpZone, WarpIndex, InputDir, FlagIndex, ScriptLabel, ExitLabel and ConnectionName.
    - A "link both ways" button also creates the return warp in the target map.
    - Lines or arrows on the map show where each warp goes, and you can click to jump to the target map.
15. **Objects:** trees (several types, re-rollable), flowers, bushes / cuttable trees, signs (editable text), boulders (Strength), smashable rocks, rocks in water, items, NPC/trainer placeholders, fences and lamp posts. Each sets the correct attribute.
16. **Tall grass and encounter tag:** paint tall grass, and assign an encounter table per area.
17. **Zoom / pan** and a **Town Map preview** panel that shows this map's position in the region.
18. **Quick-generate presets:** the three Poké Ball buttons for a small town, a route and a big city.
19. **Hint line:** a one-sentence instruction for the current tool, like in the video.
20. **Save / Open** to the existing Map Maker save format, plus export of the walkable attributes and warps.

### Asset catalogue
Create a `MapMakerAssetCatalog` ScriptableObject that maps each element to the real BDSP asset paths found in `Assets/Resources`:
- ground: grass, dirt path, sand, snow, ice, mud
- tall grass
- water: sea, pond, river, shallow, waterfall
- trees (all types and season variants)
- flowers, bushes, cut trees
- rocks, boulders, smash rocks
- cliffs and mountains: top, wall, inner and outer corners, stairs, ledges
- cave entrances
- bridges
- buildings and building parts
- signs, fences, lamps

The tools only read assets through this catalogue, so a missing or renamed asset is fixed in one place and shown in the window as a warning, not an exception.

### Done when
- Every tool above works in the Map Maker window during the safe-copy play session.
- Placed pieces use real BDSP textures and meshes and look finished: edges, corners and walls are all filled in.
- Mountains can be built at any size and height, with walls and tops.
- Cave entrances and doors warp the player to the linked spot when walked into.
- Undo/Redo, Save/Open and Walking still work.
- There are no new console errors, and the `_MainTex` errors are gone.
