# Agent / MCP Usage Notes

Practical notes for driving this extension's MCP server from an AI agent (or any MCP client). Learned in real use; add to this file whenever a new gotcha is discovered.

## Connecting

- The MCP server must be **running before the MCP client (e.g. Claude Code) starts its session** — clients wire up `.mcp.json` servers at session start. If the server comes up later, the tools will not be registered until the client **reconnects** (in Claude Code: `/mcp` → reconnect), or the session restarts.
- `GET http://127.0.0.1:3000/mcp` returns **HTTP 404** even when healthy — the endpoint speaks POST/SSE, not GET. Use the port being in `LISTENING` state as the up signal, not a GET 200.
- Start the server from the editor: **Extension menu → Cocos MCP Server → Start Server**. Confirm Status: Running, Port: 3000.

## Building scenes & prefabs

- After writing new component scripts to `assets/`, call `project_refresh_assets` so the editor **imports and registers the component classes** before you `component_attach_script` / `component_add_component` them. Without the import, the class is unknown and attaching fails.
- **Reference wiring** (`@property(Label)` / `@property(Sprite)` etc.): use `component_set_component_property` with `propertyType: "component"` and `value = the UUID of the NODE that holds the target component`. You pass the node UUID, not a component id — the server resolves the component on that node automatically.
- **`cc.Sprite` renders nothing without a `spriteFrame`.** Setting only `color` shows nothing. For solid-color backgrounds, assign a built-in white `spriteFrame` (find via `project_find_asset_by_name`) and then tint with `color`.
- Always pass `parentUuid` to `node_create_node` so nodes land where you expect (otherwise they go to scene root).
- Instantiate a prefab into the scene with `node_create_node` using `assetPath` (e.g. `db://assets/prefabs/LetterCell.prefab`).

## Panel / UI language (this fork)

- Panel UI templates are read from `static/template/...` **at runtime** (`readFileSync`), so editing them takes effect on panel reload — **no rebuild**.
- Panel *logic* strings (e.g. status text, category names) live in `source/panels/default/index.ts` → compiled to `dist/panels/default/index.js`; changing those requires `npm run build` (tsc).
- The editor's display language drives `i18n:` keys; this fork additionally translated the **hardcoded** template/panel strings to English and aliased `i18n/zh.js` to re-export `en.js`, so the panel is English regardless of editor language.

## Known broken / quirky tools (verified in real use)

- **[FIXED] `prefab_create_prefab` now creates working prefabs from nodes with custom-script components.**
  - Previous failure: the hand-rolled serializer wrote the script by class-name/cid string only, so instantiating yielded `cc.MissingScript` and dropped `spriteFrame`/size/`@property` wiring.
  - Fix: `createPrefab` now first routes through the **editor-native** serializer. The scene-script method `createPrefabFromNode` (in `source/scene.ts`) calls `cce.Prefab.createPrefabAssetFromNode(nodeUuid, url)` — the exact path the editor uses on drag-to-Assets — which serializes the script asset UUID, fileIds, and overrides correctly. The hand-rolled `createPrefabWithAssetDB` / `createStandardPrefabContent` path remains only as a fallback if the native call is unavailable.
  - Verify after reload: create a prefab from a wired node, instantiate it (`node_create_node` with `assetPath`), `component_get_components` on the instance — it must list the real view component (NOT `cc.MissingScript`), and the bg Sprite must retain its `spriteFrame`.
  - The `node_duplicate_node` workaround for grids still works and is still fine for identical cells.
- **[FIXED] `component_set_component_property` with `propertyType:"size"`** now accepts `{width,height}` (also `{x,y}`, numeric strings, and a JSON-string value). It no longer throws on valid input. UITransform `contentSize` is set via separate width/height set-property calls using the normalized values.
- **[FIXED] Components can now be addressed by class NAME or cid.** `component_set_component_property`, `component_remove_component`, and `component_get_component_info` accept either the compressed class id (cid, the `type` from `get_components`) OR the human script class name (e.g. `LetterCellView`). Name→cid resolution is done in the scene process via the new `resolveComponentCid` scene-script method (uses `cc.js._getClassId`). Error messages on these paths are now clear English (no Chinese), stating that either a cid or class name is expected and how to get the cid (`get_components`).
- **[NEW] `project_set_design_resolution`** sets `general.designResolution` via `Editor.Message.request('project', 'set-config', 'project', 'general.designResolution', {width,height,fitWidth,fitHeight})`. `fitHeight` defaults to `true` (portrait-friendly). Reads back the applied value for confirmation. Example: `{width:720, height:1280}`.
- **[NEW] `project_start_preview` (and `project_run_project` now reuses it)** opens the real editor preview (`Editor.Message.request('preview','open')`) and returns the preview URL (`preview` → `query-preview-url`). `run_project` no longer just opens the build panel.
  - **IMPORTANT — runtime logs are still NOT MCP-capturable.** The game's `onLoad`/runtime `console.log` lines (e.g. `[BrowserAdapter] init`, `[GameScene] loaded chain`) run in the preview's browser/WebView, not in the editor process MCP talks to. `debug_get_project_logs` / `debug_get_console_logs` return editor-process logs, not the running game's console. To observe runtime logs, open the returned `previewUrl` in a browser. Use `debug_validate_scene` + hierarchy/component inspection for static verification without a human.
- **Scene save path quirk:** editing often happens in the default *untitled* scene; `scene_create_scene` may leave an empty asset, and `scene_save_scene_as` can write to `db://assets/scene.scene` instead of the intended path. Verify the saved path, and `project_move_asset` to the target if needed; reopen + re-validate from disk.
