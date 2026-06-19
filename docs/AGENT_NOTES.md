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

- **`prefab_create_prefab` is broken for nodes with custom-script components.** It serializes the script by **class-name string**, not by asset UUID, so instantiating the resulting prefab yields `cc.MissingScript` and drops the `spriteFrame`, size, and `@property` wiring. The prefab file on disk is non-functional.
  - **Workaround for a grid of identical nodes:** build ONE fully-wired master node in the scene, then `node_duplicate_node` it — duplication preserves the real component and re-targets each copy's `@property` refs to its own children correctly.
  - **To actually produce a prefab asset:** drag the wired scene node into the Assets panel **in the editor UI** (that path serializes the script asset UUID correctly). The MCP path cannot.
- **`component_set_component_property` with `propertyType:"size"` fails** ("Size value must be an object…"). Use `node_set_node_property` with `contentSize` instead.
- **Custom script components must be addressed by their compressed class UUID** (e.g. `71d55rWvj9B8aSIwxmckg8R`), not the class name, in `component_set_component_property`. (`get_components` returns the usable id.)
- **`project_start_preview_server` is not supported** ("Preview server control is not supported through MCP API"). Component `onLoad`/runtime logs only fire under a real preview — start it from the editor (**Project ▸ Preview / the Play button**); the editor console is empty until then. Use `debug_validate_scene` + hierarchy/component inspection for static verification.
- **Design resolution is not editable via MCP** — it lives in Project Settings (`settings/v2/packages/project.json` → `general.designResolution`). Set portrait there or via the Project Settings UI.
- **Scene save path quirk:** editing often happens in the default *untitled* scene; `scene_create_scene` may leave an empty asset, and `scene_save_scene_as` can write to `db://assets/scene.scene` instead of the intended path. Verify the saved path, and `project_move_asset` to the target if needed; reopen + re-validate from disk.
