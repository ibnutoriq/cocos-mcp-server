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
