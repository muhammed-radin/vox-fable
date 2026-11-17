# VoxFable: Build Prompt for an AI Coding Agent

You are a senior TypeScript engineer. Build **VoxFable**, a browser-based platform for procedural 3D and 2D generation, from the specification below. Read the whole document before writing code. Build in the milestone order in section 17, and finish and test each milestone before starting the next.

Legend used in this document:

- **MUST** = a confirmed requirement from the product owner. Do not change it.
- **SHOULD** = a proposed design. Follow it unless you find a concrete problem, and then record the deviation in `docs/DECISIONS.md`.
- **Assumption** = something not yet confirmed. Implement the stated default and list it in `docs/OPEN_ITEMS.md`.

---

## 1. Product summary

VoxFable has two parts:

- **VoxFable Engine**: a pure TypeScript/JavaScript engine (the SDK). No React, no DOM-UI code. Built on three.js and other libraries. Optimized (culling, LOD, instancing). One unified API, split into **sectors**.
- **Fusion**: the VoxFable Editor. The UI and visual experience of the engine. Vite + React, dockview, shadcn UI, Monaco and Ace editors. Fusion talks to the engine only through its sectors.

Positioning (MUST): VoxFable is a procedural generator with the full power of scripting, not a node graph. Anything the engine can do, a script can do. It targets procedural 3D and 2D generation with editor assistance. AI-driven coding works externally for now (section 15).

### Vocabulary (use these words in code, UI and docs)

- **Sector**: one unified API of the engine.
- **Entity / Component**: the Core API building blocks.
- **Sheet**: one individual tool in Fusion (Assets Sheet, Inspector Sheet, and so on).
- **Panel**: one of four regions in Studio (Left, Center, Right, Bottom). A panel holds one or more sheets.
- **Studio**: the default full-screen tab in Fusion. It holds all sheets in one, arranged in the four panels.
- **Full-screen tab**: a sheet dragged out of Studio so it fills the window on its own.
- **Plugin library**: the global shared folder of available plugins. **Project plugins**: plugins imported into a project.

---

## 2. Tech stack and repository

- Package manager and monorepo: pnpm workspaces. Language: TypeScript, strict mode (Assumption: the engine is TypeScript; user scripts and plugins may be plain JS or TS).
- Packages:
  - `packages/engine`: the Engine. Zero React. Must run headless (tests run in Node with a mocked or headless three.js context where possible).
  - `packages/fusion`: the editor app. Vite + React + dockview + shadcn UI (Tailwind) + Monaco + Ace.
  - `packages/engine-types`: generated `.d.ts` for the SDK, used by Monaco for autocomplete and by the skill.md generator.
  - `examples/`: example scripts (Radial Repeat, Noise Terrain, Grass) and an example plugin.
  - `docs/`: `DECISIONS.md`, `OPEN_ITEMS.md`, and the generated `skill.md`.
- Libraries: three.js; Ammo.js (physics, built in); a noise library (for example `simplex-noise`); an IndexedDB wrapper for the browser-storage fs backend; zustand (or similar) for Fusion UI state; JSZip (or fflate) for zip packaging; GLTFExporter, OBJExporter, STLExporter from three.js examples, plus an FBX export path (SHOULD: evaluate available libraries and document the choice).
- Testing: Vitest for the engine and Fusion logic; Playwright for a small set of Fusion end-to-end tests.
- Quality bar: no `any` in the public SDK surface; every public API documented with TSDoc (the skill.md generator reads it).

---

## 3. Engine architecture

### 3.1 Sectors

The engine is a registry of sectors. Each sector implements `init()` and `dispose()` and is registered in a `SectorRegistry`. Plugins can register more sectors (MUST).

Built-in sectors:

1. **core**: entities, components, hierarchy, three.js mapping.
2. **assets**: the unified file system (section 3.2).
3. **plugins**: loading, map file, import, versions (section 11).
4. **events**: a typed event bus used by every sector and by scripts.
5. **settings**: preferences, configuration, performance mode.
6. **editor**: the bridge Fusion uses (selection, tools registry, command stack, wireframe/outline flags).
7. **scheduler** (MUST): runs all heavy work as async jobs (section 8).
8. **physics** (MUST): Ammo.js, built in (not a plugin). Emits contact events.
9. **ai** (MUST exists as a sector): the entry point for AI-driven coding. In v1 it is a thin placeholder that exposes the skill.md content and no live AI (section 15).

### 3.2 File system (assets sector)

One unified async fs API with three interchangeable backends (MUST):

1. **Local folder**: browser File System Access API. The user opens a real folder; reads and writes go to the user's disk.
2. **Browser storage**: a cache/IndexedDB-based fs. Works in most browsers.
3. **Server/Node fs**: NOT available in v1. Define the interface and a stub that throws a clear "not available" error. It is planned for v2 or as a plugin.

API sketch (SHOULD): `readFile`, `writeFile`, `readDir`, `mkdir`, `remove`, `rename`, `stat`, `watch`. Paths are POSIX-style. Dot-folders such as `/.exports`, `/.meshes`, `/.plugins` are real folders and MUST be visible in the Assets Sheet.

### 3.3 Events

A typed event bus. Required event families (SHOULD names): `entity:created|deleted|changed`, `component:added|removed|changed`, `script:attached|detached|enabled|disabled|error`, `selection:changed`, `job:started|progress|finished|cancelled`, `contact:begin|contact:end`, `project:opened|closed`, `plugin:loaded|updateAvailable`.

---

## 4. Core API

Core is an entity system with a hierarchy, built on three.js (MUST). It uses what three.js already provides instead of reinventing it.

### 4.1 Entities

Every entity has default properties with no component needed:

- `name`, `tags`, `keywords` (MUST).
- `id` (unique and stable), `enabled`, `locked` (blocks editing in Fusion), `description`, parent and children (SHOULD).

### 4.2 Components

Components are registered by type, with a schema, a serializer and lifecycle hooks. Plugins can register more (MUST).

Default components (physics and 2D are built in, not plugins: MUST; the exact list is a SHOULD):

- **3D basics**: Transform, Render (subtypes: Shapes such as cube and sphere, Mesh, more later), Script, Camera, Light (directional, point, spot, ambient), Particle System, Audio.
- **Physics** (built in, Ammo.js): RigidBody, Collider.
- **2D** (built in): Sprite, Shape 2D, Text 2D. 2D content is shown in the 3D world as planes.

### 4.3 Core entity to three.js mapping (MUST)

- Core entity -> one three.js `Object3D` root. Components attach their own three.js objects under it. One component can own many three.js objects (example: a Particle System entity owns many objects).
- Every three.js object created by a Core component gets an owner tag (`userData.vfOwner = entityId`) (SHOULD).
- **Hierarchy Sheet** shows Core entities only. **Threejs Hierarchy Sheet** shows the pure three.js scene, hidden and visible objects, for debugging.
- A script can use the three.js API directly through Core (fully customizable, but the editor gives no controls or tools for those objects), or it can use Core components (editor controls and tools apply).

---

## 5. Scripts

### 5.1 Model (MUST)

- A script is a file in the project fs (for example `scripts/RadialRepeat.js`). The Script component on an entity points to a script file.
- Several entities can share one script file. Each entity has its own attribute values (stored on the entity, not in the file).
- An entity can have **multiple scripts**, and scripts on an entity run in a **defined order** (see 5.4).

### 5.2 API

```js
const RadialRepeat = new ScriptComponent('RadialRepeat');

// Full schema (editor-aware). The Inspector builds its fields from it.
RadialRepeat.attributes.add('angle', { type: 'number', default: 0, array: false, range: { min: 0, max: 360 } });
// Schema is optional.
RadialRepeat.attributes.add('sweep', { type: 'number' });

RadialRepeat.onInit = function () {};
RadialRepeat.onPreUpdate = function (dt) {};
RadialRepeat.onUpdate = function (dt) {};
RadialRepeat.onPostUpdate = function (dt) {};
RadialRepeat.onAttributeChange = function (name, value, oldValue) {};
RadialRepeat.onDestroy = function () {};

// Export hooks (called during an export, including whole-project export)
RadialRepeat.onPreExport = function (ctx) {};
RadialRepeat.onExport = function (ctx) {};
RadialRepeat.onPostExport = function (ctx) {};
```

Hook names are MUST. Signatures are SHOULD.

### 5.3 Reading other scripts

A script can read another script on the same entity (MUST):

```js
const terrain = this.entity.scripts.has('terrainMesh')
  ? this.entity.scripts.get('terrainMesh').attributes
  : null;
```

`has()` is false once that script is gone, so the reader gets `null` and must handle it.

### 5.4 Order (MUST: order exists; Assumption: UI)

Scripts on an entity run in order, top to bottom, in every phase (`onInit`, `onPreUpdate`, `onUpdate`, `onPostUpdate`, exports). Default order = the order they were added. Assumption: the Script component in the Inspector lets the user drag to reorder. Record this in `docs/OPEN_ITEMS.md`.

### 5.5 Lifecycle rules (MUST)

- **Delete and disable are the same**: both run `onDestroy`. The script must undo what it changed on the entity and delete what it made itself (hidden objects, an offscreen canvas, noise data, a generated mesh). Re-enabling runs `onInit` again and rebuilds from scratch.
- When a script is deleted or disabled, the entity is **reset** from its components. Nothing is really deleted: the entity keeps the same components, with or without hand edits (section 6.2). The effect is like deleting the entity and placing it again without the script (like removing a Blender geometry node), but implemented as a reset.
- When a script is deleted or disabled, the **other scripts on that entity automatically run `onInit` again**. Example: Radial Repeat copied Terrain's mesh, so it rebuilds after Terrain is removed. (Open item: all other scripts, or only dependent ones. Default: all other scripts, in order. Record in `docs/OPEN_ITEMS.md`.)
- Children of the entity are separate entities and are untouched, except children the removed script itself created (its `onDestroy` removes them).

### 5.6 Reload and errors (MUST)

- Saving a script file does NOT reload it. A **Reload/Restart script** button and a **keyboard shortcut** (editable in Key Binding Preferences) reload it. Attribute changes already sync live through `onAttributeChange`.
- Script errors, both runtime errors and syntax errors, MUST be caught, must not crash the engine, and must be shown in Fusion at the script (Script component in the Inspector and the Code Editor Sheet) with the stack trace. SHOULD: a failing script is stopped and marked as errored while the other scripts keep running.

### 5.7 Update mode

How changes to script attributes apply is decided by the script itself through `onAttributeChange` (MUST). Component update modes are in section 10.5.

---

## 6. Live procedural flow and non-destructive generation

### 6.1 Live flow (MUST)

Tools that belong to a component (modeling, sculpting, mesh editing) appear when that component is selected. Example: extrude a face, the scripts re-run, the scripts make their changes. Every change is received by components and can change them.

### 6.2 Generation model (MUST, like Blender geometry nodes)

This is an example flow, not a rule. Each script decides what to hide, show or build.

Example: an entity has a Render component (grid or cube) and a Terrain generation script.

1. The script hides the Core Render (opacity 0, hidden not deleted) and builds a mock mesh with the three.js API.
2. The user sees the mock mesh but cannot edit it directly. Changes happen through the script attributes, or by editing the hidden Core entity by hand.
3. The user can still edit the hidden Core entity: edit -> the script receives the change -> the three.js mesh updates in realtime while the Core SDK entity updates too.
4. Wireframe and outline are **Fusion options, not script or entity features**. They are a debugging aid and still show hidden objects. The user first selects the entity, then enables wireframe or outline for the selected entity in Fusion.
5. When the script is removed, the entity is reset from its Render component.

### 6.3 Hand-made mesh edits and stored meshes (MUST)

- If the user edited a shape by hand, the model is no longer a plain shape. The user's changes are stored as a variable and also saved in Assets, in the `/.meshes` folder.
- The Render component refers to the stored mesh by the name `[shape or mesh name]#[variable id]`, for example `cube#m_2736` or `mesh#m_36`.
- When the script is deleted afterwards, the entity is recovered from the stored mesh, so the hand edits are kept.
- Stored meshes are reusable by other entities. The file name can be renamed, but the id (`m_2736`) can never change, so references keep working.
- An optional "bake" step has not been discussed. Do not build it.

### 6.4 Acceptance scenario: physics-driven grass

Build this as an example script and a demo project (SHOULD implementation):

- A surface entity (grass script + rigid body mesh) and a cube entity (rigid body). Grass fills the surface with noise.
- The user moves the cube with a gizmo; when it touches the surface, grass instantly covers the cube. Moving the cube away makes the grass leave, like a magnetic fluid.
- Design: the physics sector emits `contact:begin` and `contact:end`; the grass script keeps blades in chunks (a spatial grid) and only touches chunks near a contact; affected blades get a new target on the cube surface (seeded random sampling, stable results) and return home when contact ends; blade movement is animated on the GPU with per-instance attributes; blades live in an instanced mesh; culling and LOD apply.

---

## 7. Entity references (SHOULD)

Add an attribute type `entity` (a reference to another entity), shown in the Inspector as an entity picker, so scripts can depend on other entities. Cross-entity events must be available to scripts (SHOULD).

---

## 8. Async-first execution and the scheduler (MUST principle)

- Tasks run as functions and asynchronous processes, not long synchronous `for`/`while` loops.
- Work is placed one by one or group by group with small delays (about 0 to 100+ ms, for example 1 ms between groups) so the browser is not stressed.
- Each running process shows its progress in the **status bar**.
- A sudden large process can kill the tab (Aw Snap, WebGL or canvas errors). Treat this as a core design rule.

Scheduler sector (the sector is a MUST, the mechanism is a SHOULD):

- A job queue with a time budget per frame (default 2 to 4 ms) and optional delays.
- Priorities, cancellation (moving the cube again drops the old job), and progress reporting.
- Guards: caps on instance counts, chunked buffers that get disposed, web workers for heavy sampling, handling of WebGL context loss.
- Job API (SHOULD): `scheduler.run({ name, priority, budgetMs, delayMs, chunks, onProgress, signal })`.
- Every first-party tool, exporter and generator must use the scheduler for anything that can take longer than one frame.

---

## 9. Fusion layout (MUST)

From top to bottom and left to right:

- **Top menu** with dropdowns (File, Edit, View, and others as needed).
- **Full-screen tabs row**. The default tab is **Studio**.
- Inside Studio: an icons-only **Tools strip** on the far left, then the **Left panel**, the **Center panel**, the **Bottom panel** (under the center only) and the **Right panel** (full height). Panels are resizable (dockview).
- A thin **status bar** under the Bottom panel, spanning the whole window like VS Code. It shows running processes with progress. Clicking a job shows details or cancels it.

Sheet and tab behavior (MUST):

- By default every sheet lives in one of the four panels.
- A sheet that is hard to use in a narrow panel (Code Editor, Assets) can be dragged into the full-screen tabs row. It **moves** there (it is not copied).
- Closing any sheet (in Studio or full-screen) **hides** it and stores its position and state in localStorage. It does not return to its origin panel.
- A closed sheet is shown again from **View -> [sheet name]**, where each sheet is a checkbox.

Default arrangement in Studio (SHOULD): Left = Hierarchy, Assets; Center = Canvas (with Code Editor and Image Editor as other tabs); Right = Inspector, Tool Properties; Bottom = Previewer, Threejs Hierarchy, Plugin Manager. Projects, Settings, Key Binding Preferences and Export open as sheets the user can place anywhere.

---

## 10. Fusion sheets (MUST unless marked)

### 10.1 Sheet list and specs

- **Projects Sheet**: Fusion's home page. Lists all projects (not scenes). Create, delete, edit title and description. Opening a project loads it into Studio. Only one project can be open at a time, and opening another requires closing the current one.
- **Code Editor Sheet**: multi-tab code editor. The editor engine is a global setting: **Monaco** (recommended on desktop with a hardware keyboard: best suggestions and linting) or **Ace** (recommended on tablets and phones with a virtual keyboard). No device auto-detection; the user switches in Settings. Both sit behind one `EditorAdapter` interface so tabs, files and SDK type hints are the same. Monaco gets the SDK `.d.ts` from `engine-types`. Assets can be dropped in to insert a path. Includes the Reload script button.
- **Assets Sheet**: folder-based file system with CRUD over the unified fs API. Dropping files in to import is allowed. Right-click gives **Copy Path**. Assets can be dropped on Inspector fields and into the Code Editor. Dropping an asset onto the Canvas to create an entity is NOT allowed.
- **Canvas Sheet**: the 3D editor (three.js viewport, camera controls, gizmos, selection, wireframe/outline overlay).
- **Image Editor Sheet**: a simple editor built from separate tools, not modes: paint, draw shapes, paint with shapes, stamp, plus view and edit. Main use: paint a grayscale image, export a PNG into Assets, and let a script use it as a heightmap. Advanced editing is out of scope.
- **Inspector Sheet**: components of the selected entity and their properties (section 10.5).
- **Hierarchy Sheet**: the Core entity tree.
- **Threejs Hierarchy Sheet**: the pure three.js scene, hidden and visible, for debugging.
- **Previewer Sheet**: previews 3D files, assets and images. Opens on **double-click** of an asset.
- **Plugin Manager Sheet**: list, import, remove, enable/disable, update plugins (section 11).
- **Export Sheet**: section 12.
- **Settings Sheet**: user preferences, stored globally in localStorage, exportable to and importable from JSON. Includes the performance mode and the editor engine choice.
- **Key Binding Preferences Sheet**: same storage and JSON export/import as Settings.
- **Tool Properties Sheet**: one shared sheet that shows the properties of whichever tool is active (tool-based, not mode-based). Image Editor tools and modeling tools both use it.

### 10.2 Tools strip

An icons-only strip, grouped, with no Object/Edit mode switch (MUST):

1. **Global controls**: always shown, not tied to an entity. Move (moves the camera), select (selects entities), delete, and so on.
2. **Transform tools** (gizmos): shown when the selected entity has a Transform component.
3. **Modeling tools**: shown for Render subtypes such as Shapes and Mesh. Sub-groups: face-based tools (extrude, move, select, gizmos) and vector-based tools.
4. **More groups** from other components. Plugins can add tools and override existing ones.

Each tool declares the components or subtypes it applies to; the strip is rebuilt from the selection. With several entities selected, only tools for components common to all selected entities are shown.

### 10.3 Selection

Selecting an entity (Hierarchy or Canvas) drives the Canvas, Inspector, Tool Properties and Tools strip. Multi-select is supported (Ctrl/Shift click).

### 10.4 Multi-select behavior

Gizmos move the group by a position delta applied to each entity. In the Inspector, a value shared by all selected entities is shown; differing values show "..." and typing over "..." applies the value to all selected entities.

### 10.5 When Inspector changes apply

- The user's performance setting decides: low-performance (fast) mode applies on leave or Enter; high-performance mode applies live.
- A component can declare its own live or on-change behavior, but the user's performance setting overrides it by default. A component wins only if it sets `blockUserPreferenceOverride: true`.
- The Script component is outside these rules; the script decides through `onAttributeChange`.

---

## 11. Project layout and plugins

### 11.1 Project file system (SHOULD names)

```
<project>/
  scripts/        user scripts
  assets/         textures, models, images
  plugins/        project plugins (imported from the library)
  .meshes/        stored hand-edited meshes (visible in Assets), ids like m_2736
  .exports/       export output (visible in Assets), zipped by Fusion
  project.json    title, description, scenes/entities, settings
```

Dot-folders are real folders and are shown in the Assets Sheet.

### 11.2 Plugins (MUST)

- A plugin is a TS/JS file that extends the engine or Fusion. It exports a **header config**: `name`, `version`, `title`, `description` and the target (`engine` or `fusion`). Then code that runs once and uses the SDK. It can keep updating live through SDK events.
- **Plugin library**: the global shared folder `/.plugins` (local now; cloud libraries are planned but do not exist yet, so only define the interface). It contains a **plugins map JSON**: an array of objects, each with an extension name and a path. Fusion and the SDK read the map and load only the mapped files.
- **Project plugins**: each project imports the plugins it wants from the library into its own `./plugins` folder. Only those run for that project. Project exports include them.
- Plugins run when Fusion starts, or when a plugin is installed while Fusion is running.
- **Versions**: a project stays on its imported version until the user updates it in the Plugin Manager Sheet. Fusion tells the user when a newer version is found in the library (notify only, never auto-update).
- Plugins can register sectors, components, tools, sheets and menu items. SHOULD: security notes in `docs/OPEN_ITEMS.md` (plugins and scripts run arbitrary code in the user's browser; consider a trust prompt on import).

---

## 12. Export (MUST)

- 3D formats: glb/gltf, fbx (with textures), obj, stl.
- If an export produces more than one file, deliver a zip.
- During an export, including a whole-project export, every Script component receives `onPreExport`, `onExport`, `onPostExport` (in script order). Scripts write what they need (used textures, offscreen-generated noise, and so on) into `/.exports`. Fusion zips that folder.
- The Export Sheet handles 3D export, plugin export and project export.
- Exports run as scheduler jobs with progress in the status bar.

---

## 13. 2D (MUST)

2D has its own built-in components (Sprite, Shape 2D, Text 2D) and its own tools. 2D content is shown in the 3D world as planes. The Tools strip logic is the same as for 3D: tools follow the selected entity's components.

---

## 14. Undo and redo (MUST: supported; SHOULD: mechanism)

All edits made by tools and the Inspector go through one **command stack** (command pattern, each command has `do` and `undo`). Script-generated results are rebuilt from their inputs and do not need their own history. How undo treats script file edits is not decided: default to leaving file edits to the Code Editor's own undo, and record it in `docs/OPEN_ITEMS.md`.

---

## 15. AI-driven coding (MUST)

- v1: AI works **externally**. Generate and maintain a `skill.md` that contains the full VoxFable documentation and knowledge (sectors, Core API, components, scripts and hooks, plugin format, export, folders, conventions, plus the example scripts). The user gives `skill.md` to an AI agent, which creates scripts and other files; the user imports those into the Assets Sheet and uses them.
- Build a `pnpm docs:skill` command that generates `skill.md` from the TSDoc and a hand-written knowledge file. Keep it in sync as the SDK changes.
- v2 (do NOT build now): internal AI with an AI Sheet inside Fusion, through the `ai` sector.

---

## 16. Non-goals for v1

- Server/Node fs backend (interface and stub only).
- Cloud plugin libraries (interface only).
- Internal AI and the AI Sheet.
- Opening more than one project at a time.
- Creating an entity by dropping an asset onto the Canvas.
- A bake step for generated meshes.
- Advanced image editing (layers, filters, and so on).

---

## 17. Milestones and acceptance

Do them in order. At the end of each, run tests, update `docs/DECISIONS.md`, and summarize what is done and what is open.

1. **M0, tooling**: monorepo, TS strict, lint, Vitest, Vite app that boots an empty Studio.
2. **M1, engine core**: sector registry, events, scheduler, entities with default properties, component registry, Transform and Render (Shapes), three.js mapping with owner tags, hierarchy vs three.js hierarchy. *Accept:* create and delete entities headless; tests pass; scheduler time budget respected in a test.
3. **M2, assets fs**: unified API, File System Access backend, IndexedDB backend, server stub. *Accept:* the same test suite passes on both real backends.
4. **M3, scripts runtime**: ScriptComponent, attributes with optional schema, all hooks, script order, shared script files with per-entity values, reading other scripts, delete/disable semantics with `onDestroy`, automatic `onInit` re-run of other scripts, error capture with stack, reload. *Accept:* the Radial Repeat and Terrain-plus-Radial-Repeat scenarios behave as in section 5.
5. **M4, Fusion shell**: dockview Studio, full-screen tabs row, drag-out, close-and-hide with localStorage, View menu checkboxes, status bar, Projects Sheet, Settings and Key Binding Preferences with JSON export/import. *Accept:* the tab behavior in section 9 works and survives a reload.
6. **M5, editing core**: Canvas, Hierarchy, Inspector (multi-select, "...", update modes), Tools strip with context groups, Tool Properties, gizmos, command stack with undo/redo. *Accept:* multi-select edit and undo work across three entities.
7. **M6, code and assets**: Code Editor with Monaco and the Ace adapter, SDK type hints, Reload button and shortcut, Assets Sheet with drag and drop rules and Copy Path, Previewer on double-click.
8. **M7, physics**: Ammo.js sector, RigidBody and Collider, contact events. *Accept:* the grass demo from section 6.4 runs smoothly with the status bar showing job progress.
9. **M8, generation and modeling**: hidden Core entity, mock mesh flow, `/.meshes` with stable ids, basic face-based tools (extrude), wireframe and outline in Fusion. *Accept:* the Noise Terrain Cube scenario, including hand edits surviving script removal.
10. **M9, plugins**: header config, library map, per-project import, versions and update notice, Plugin Manager Sheet. *Accept:* an example plugin adds a component and a tool.
11. **M10, export**: the four 3D formats, export hooks, `/.exports`, zip. *Accept:* a project export zip reopens with its plugins.
12. **M11, 2D and Image Editor**: 2D components and tools, Image Editor tools, the grayscale-to-heightmap workflow.
13. **M12, polish**: `skill.md` generator, performance pass (culling, LOD, instancing), crash guards, touch and mobile pass for Ace and the sheets.

---

## 18. Working rules for you, the agent

- Prefer small, tested commits. Keep the engine free of UI code.
- Never block the main thread with long synchronous loops; use the scheduler.
- Do not invent product behavior. If something is not in this document, choose the simplest default, implement it, and list it in `docs/OPEN_ITEMS.md`.
- Keep `docs/DECISIONS.md` current: every deviation from a SHOULD, with the reason.
- At the end of every milestone, report: what works, what is tested, what is still open, and the next milestone.

## 19. Open items (record these in `docs/OPEN_ITEMS.md` and ask the owner)

1. Script order UI (default: drag to reorder in the Script component).
2. Whether all other scripts or only dependent scripts re-run `onInit` when a script is removed (default: all).
3. The exact default component list for v1.
4. How undo and redo treat script file edits.
5. Entity reference attribute type and cross-entity events.
6. Security model for plugins and scripts (sandboxing or trust prompt).
7. Project file format and versioning.
8. Touch and mobile behavior of Studio, the Canvas gizmos and the sheets.
9. How far an internal AI may go in v2.
