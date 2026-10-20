# SLua script editor and IDE bridge: carry-over survey for vkstorm-aleph

Source tree surveyed: `C:\Dev\vkstorm-release-ref` at `origin/vkstorm-release` = `08c07b4cba` (read-only).
Feature commits: `c2b161aa8f` (port of LL Viewer 26.4 SLua work, tag `Second_Life_Release#e960123-26.4`) plus follow-ups `a550105f2c` (lifecycle/review fixes), `dfd42a720b` (notification resources), `073bfba6bc` (POSIX exclusive challenge writes), `8309d71896` (Boost/WebSocket++ include order), `92f93fd56d` (Lua export filenames).
Reference test plan: `doc/testplans/slua_editor.md`.

All path:line citations below are relative to `C:\Dev\vkstorm-release-ref` unless stated otherwise. Anything marked **[external]** was fetched from the `secondlife/sl-vscode-plugin` GitHub repository (develop branch) on 2026-10-20 and is not part of the release tree.

Two caveats about the diff basis:

- `git diff upstream/master -- <file>` compares the release branch against current Firestorm `master`, so for shared files it also shows unrelated Vulkanstorm/Firestorm drift (e.g. extra-parameter validation in `llviewerobject.cpp`, cached controls in `llselectmgr.cpp`, the FS "Copy Name" and `FSNewItemCtrl` features). I have cited only the SLua-related hunks.
- `llscripteditorws.cpp:6` refers to `doc/external-editor-json-rpc.md`; that file does not exist in the tree. The protocol description in section 2 is reconstructed from the code.

---

## 1. Feature description

### 1.1 Compile targets

Four target strings are used end to end; they are the literal values put into the `UpdateScriptTask` / `UpdateScriptAgentInventory` POST body `target` field (`indra/newview/llviewerassetupload.cpp:864,874`, `LLScriptAssetUpload` now carries a `std::string mCompileTarget` instead of the old `LSL2/MONO` enum, `indra/newview/llviewerassetupload.h` lines ~256-273):

| Value | Meaning | UI label (`panel_script_ed.xml:374-377`) |
|---|---|---|
| `lsl2` | LSL source on the legacy LSO VM | "LSL Legacy (LSO)" |
| `mono` | LSL source on Mono | "LSL on Mono" |
| `lsl-luau` | LSL source compiled for the Luau VM (compat mode) | "LSL on Luau" |
| `luau` | SLua (Lua) source on Luau | "Lua" |

The compile-target combo replaces the old Mono checkbox (`indra/newview/llpreviewscript.h` diff: `mMonoCheckbox` removed, `LLComboBox* mCompileTarget` added to `LLScriptEdCore`). The `luau` and `lsl-luau` entries are enabled only when the region advertises Lua (`llpreviewscript.cpp:3622-3627`, `:686-689` in the diff of `postBuild`).

The compile queue (bulk recompile) accepts the same strings plus two pseudo-targets: `auto-luau` (choose `luau` or `lsl-luau` per item by subtype) and `auto` (use each item's recorded runtime; skip items with none) (`indra/newview/llcompilequeue.cpp:474-492`). It refuses to send a script whose source language does not match the target (`llcompilequeue.cpp:627-642`, message `SkippingIncompatibleScript` in `floater_script_queue.xml`). Menu actions `compile lua` / `compile lsl-luau` map to `luau` / `lsl-luau` (`indra/newview/llviewermenu.cpp:9940-9977`; menu items `menu_viewer.xml:2043-2049`, enabled by `LLEditableSelectedLua` which checks `UpdateScriptTask` cap + `LuaScriptsEnabled`, `llviewermenu.cpp:10638-10655`).

### 1.2 How the simulator advertises SLua

- **Simulator feature `LuaScriptsEnabled`** (boolean in the `SimulatorFeatures` LLSD). Read by: `have_lua_enabled()` (`llpreviewscript.cpp:173-195`), the Build floater Contents tab (`indra/newview/llpanelcontents.cpp:186`), `create_new_item` for agent inventory (`indra/newview/llviewerinventory.cpp:2069-2078`), and the menu enable check (`llviewermenu.cpp:10638-10649`). The region may not have features yet: `LLPanelContents` subscribes to `setSimulatorFeaturesReceivedCallback` and refreshes (`llpanelcontents.cpp` getState block after line 186).
- **Capabilities requested** (`indra/newview/llviewerregion.cpp:3494-3495`): `CreateTaskInventoryItem` (create script/notecard in a prim via HTTP) and `ScriptDefinitions` (combined LSL+Lua syntax definitions). Existing caps still used: `UpdateScriptTask`, `UpdateScriptAgentInventory`, `UpdateNotecardTaskInventory`, `LSLSyntax` (legacy, LSL only), `RequestTaskInventory`.
- **Simulator feature `LSLSyntaxId`** (UUID) still keys the syntax cache; when it changes the viewer fetches new definitions (`indra/newview/llsyntaxid.cpp:90-133`).
- **ScriptRunningReply** now arrives as an LLSD message with extra booleans `Mono`, `Luau`, `LuauLanguage` (`indra/llmessage/message_prehash.cpp` additions; the UDP template `scripts/messages/message_template.msg:5508-5518` marks it `UDPDeprecated` with `Mono` "Added to LLSD message"). The editor decodes them as: `Luau && LuauLanguage` -> `luau`; `Luau && !LuauLanguage` -> `lsl-luau`; `Mono` -> `mono`; else `lsl2` (`llpreviewscript.cpp:3576-3629`).

### 1.3 Script asset / inventory metadata

- **Subtype (source language)** lives in the low byte of the inventory item flags: `II_FLAGS_SUBTYPE_MASK = 0x0000ff` (`indra/llinventory/llinventorydefines.h:84`); `enum ScriptSubtype_t { SST_LSL = 0, SST_LUA = 1 }` (`indra/llinventory/llinventorytype.h:155`); accessors `getInventorySubType()` / `setInventorySubType()` (`indra/llinventory/llinventory.h:174,191`).
- **Runtime (VM)** is a new string member `LLInventoryObject::mRuntime` (`llinventory.h:77,94,121`), values `lsl2` / `mono` / `luau` or empty (never compiled). Serialisation:
  - AIS/LLSD: `sd["script"]["runtime"]` on export (`indra/llinventory/llinventory.cpp:1054`), accepted from `script.runtime` or `metadata.script.runtime` on import (`:1113`).
  - Legacy task-inventory stream: `metadata` line carries `{"script": {"runtime": ...}}` alongside thumbnail (`llinventory.cpp:261,846,981,1558`).
  - Cleared when the asset type changes away from `AT_LSL_TEXT` (`llinventory.cpp:207`); copied by `copyObject`/`copyItem`.
- **Task-inventory per-script state**: `LLViewerInventoryItem::mIsRunning` / `mIsFaulted` from the task-inventory LLSD fields `running` / `faulted` (`indra/newview/llviewerinventory.cpp:551-552`, header `llviewerinventory.h:151-152,171-172`).
- Tests: `indra/llinventory/tests/llinventory_runtime_test.cpp` (7 cases: SLua metadata round trip, LSL on Luau, AIS vs wrapped metadata, copy clears runtime on type change, unrelated flags preserved on subtype change, legacy serialisation).

### 1.4 Creation flow

- **In a prim** (Build floater > Contents): the "New Script" button is a flyout with `lsl` / `lua` values (`floater_tools.xml:3304-3307`); default follows the region (`llpanelcontents.cpp` getState). `onNewScriptFlyoutCommit` (`llpanelcontents.cpp:313`) builds `params = {enabled: true, vm: "luau"|"mono"}` and calls `createTaskInventoryItemHelper` (`:374`), which:
  1. applies RLVa guards (locked attachment / seated linkset);
  2. honours the Firestorm custom template (`FSBuildPrefs_UseCustomScript`) only if the template's subtype matches the requested language (`:398-402`);
  3. uses the `CreateTaskInventoryItem` cap when present via `LLViewerObject::createInventoryItem` (`indra/newview/llviewerobject.cpp:8204-8290`; body: `object_id, inventory_type, asset_type, sub_type, name, description, permissions{base,owner,everyone,group,next_owner}, params`; on success bumps `mExpectedInventorySerialNum` and re-requests inventory);
  4. otherwise falls back to the legacy `RezScript` UDP path with the subtype in the flags (`LLViewerObject::saveScript`, which now also supports an optional `NewScriptInfo.TemplateID` block, `llviewerobject.cpp:2977`).
  - New notecards also go through the cap (`onNewNotecardCommit`); the button is disabled without the cap.
  - The server supplies the default script body for the chosen subtype; the viewer-side `HELLO_LSL` template was removed (`llpreviewscript.cpp` diff lines 52-64, 2229, 2706).
- **In agent inventory**: `create_new_item` sets `subtype = SST_LUA` when the current region has `LuaScriptsEnabled` (`llviewerinventory.cpp:2062-2078`); there is a `*TODO*` noting the user cannot yet choose.
- **Script recovery** (Firestorm feature, kept): backups are `*.lslbackup` or `*.luabackup` by language (`llpreviewscript.cpp:2241-2251`); recovery creates the item with the matching subtype and uploads with target `luau` or `mono` (`indra/newview/llfloaterscriptrecover.cpp:162,248,307`).

### 1.5 Icons

`ICONNAME_SCRIPT_LUAU` (`llinventorytype.h:126`) -> texture `Inv_Script_Luau` (`indra/newview/llinventoryicon.cpp:105`, `textures.xml:1216`, PNG at `indra/newview/skins/default/textures/icons/Inv_Script_Luau.png`). Selection: `AT_LSL_TEXT` with `(flags & SUBTYPE_MASK) == SST_LUA` (`llinventoryicon.cpp:156-158`). Both the agent-inventory bridge and the task-inventory bridge override `getIcon()` to pass the flags (`indra/newview/llinventorybridge.cpp:8352`, `indra/newview/llpanelobjectinventory.cpp:1054`).

### 1.6 How the editor decides the language

Order of precedence, identical for agent-inventory (`LLPreviewLSL::onLoadComplete`, `llpreviewscript.cpp:2845-2925`) and task (`LLLiveLSLEditor::loadScriptText`, `:3244-3286`) editors:

1. `subtype == SST_LUA` -> Lua.
2. else if `runtime == "luau"` -> LSL with compile target `lsl-luau`; otherwise the compile target is the recorded runtime.
3. If no runtime recorded and not Lua, fall back to content analysis `is_lua_script()` (`llpreviewscript.cpp:198-215`): source containing a preprocessor directive at line start is LSL (Firestorm addition for `#include`d default states); source containing `\s*default\s*\{` is LSL; otherwise Lua. Default target when unknown: `luau` for Lua, else `mono` (task) or `FSSaveInventoryScriptsAsMono ? mono : lsl2` (agent inventory).
4. For live editors the later `ScriptRunningReply` overrides the combo and re-runs highlighting (`:3576-3629`).

Changing the combo flips `LLScriptEditor::setLuauLanguage` and reprocesses keywords (`:2836-2843`, `:3631-3638`), i.e. the language is "Lua iff target == luau".

Line/column display is 1-based for Lua and 0-based for LSL (`llpreviewscript.cpp` draw, diff lines 389-390; gutter `indra/newview/llscripteditor.cpp:320`), and compiler error rows are adjusted accordingly (`onErrorList`, diff line 566).

### 1.7 Lua lexical rules used for highlighting

`LLKeywords` (llui) got a language flag (`indra/llui/llkeywords.cpp:248`) and these additions in `processTokens` (`:255-280`):

- Both languages: `"` double-quoted strings (`TT_DOUBLE_QUOTATION_MARKS`).
- Lua only: `'` single-quoted strings, `` ` `` backtick (interpolated) strings as plain string tokens, `--` single-line comments (`TT_ONE_SIDED_DELIMITER`), `--[` long comments and `[` long strings as a new token type `TT_LONG_BRACKET` (`llkeywords.h:62,72`). The scanner validates `[=*[` after the head and finds the matching `]=*]` with the same level; an unterminated long bracket highlights to EOF; `--[abc` falls back to a normal `--` comment (`llkeywords.cpp:646-705`).
- LSL only: `@label`, `//`, `/* */`.
- Word scanning: in Lua a word may contain `.` followed by an alnum (`ll.Say`, `math.sin`); first try the whole dotted token, then the prefix (`ll`) and the suffix (`Say`) separately (`llkeywords.cpp:785-840`). LSL keeps the Firestorm `#` allowance for preprocessor macros.
- Colour groups: `constants-*` keys fall back to `SyntaxLslConstant` (`:236`). Function tooltips omit "Energy" when the definitions give a negative energy (`:394`).
- Highlighting is now computed off-thread: `collectSegmentOps` (`:529`) produces a vector of `SegmentOp{OP_LINE_BREAK|OP_TOKEN,start,end,token}`; `LLScriptEditorSyntaxWorker` (`llscripteditor.cpp:45-207`) runs it on an `LLWorkerThread`, and the editor applies ops in slices of 250 (`applySegmentOpsRange`). Setting `ScriptEditorDisableSyntaxHighlight` (`settings.xml:27667`) skips token scanning.
- The editor holds two `LLKeywords` instances (`mKeywordsLua`, `mKeywordsLSL`) and switches with `mLuauLanguage` (`llscripteditor.cpp:342-350,578-581`).

### 1.8 Definitions package (`lsl_definitions` 0.6.12)

Autobuild entry `autobuild.xml:3086-3126` (`https://github.com/secondlife/lsl-definitions/releases/download/v0.6.12/lsl_definitions-0.6.12-common-29281156609.tar.zst`, MIT). `indra/cmake/LSLDefinitions.cmake` copies `lsl_keywords.xml` and `lua_keywords.xml` from the package into `app_settings/keywords_lsl_default.xml` / `keywords_lua_default.xml`; both generated files are git-ignored (`.gitignore` diff). The package also ships (seen in `C:\Dev\vulkanstorm\build-vc170-64\packages\lsl_definitions\`):

- `lsl_definitions.yaml` (LSL: `llsd-lsl-syntax-version: 2`, enums, constants, functions, events) and `slua_definitions.yaml` (`version: 1.0.0`, base-classes, classes, globals, with typed methods/properties).
- `lsl_keywords.xml`, `lua_keywords.xml`: LLSD XML, top-level keys `controls`, `types`, `constants`, `events`, `functions`, `llsd-lsl-syntax-version` (=2). Each function entry has `arguments` (array of `{name: {type, tooltip}}`), optional `type-arguments`, `return`, `energy`, `sleep`, `tooltip`. This is exactly what the viewer parses (`llkeywords.cpp:309 processTokensGroup`).
- `syntax/lsl.tmLanguage(.json)`, `syntax/slua.tmLanguage(.json)` TextMate grammars.
- `secondlife.d.luau` (luau-lsp type definitions), `secondlife.docs.json` (luau-lsp docs), `builtins.txt`, `secondlife_selene.yml`, generator Python and C++ outputs.

Runtime handling (`LLSyntaxDefCache`, `indra/newview/llsyntaxid.cpp`): defaults are loaded from `app_settings` (`:348-364`); on region change, if `LSLSyntaxId` differs, it looks for `<cache>/syntax_<uuid>/` (`:428-455`) else fetches. With `ScriptDefinitions` it expects LLSD `{"files": {"lsl_keywords.xml": <llsd or binary>, "lua_keywords.xml": ...}}` and caches every file by name (`:257-342`); with legacy `LSLSyntax` only LSL is fetched and Lua keeps the default (`:185-255`). Any file missing from the region response keeps its previous content. A `boost::signals2` signal fires on change (`addSyntaxIDCallback`), used by both the editor and the bridge.

### 1.9 Syntax help

F1 / help button URL builder `build_script_help_url` (`llpreviewscript.cpp:127-142`): LSL and `ll.*` Lua symbols go to `LSLHelpURL` with `[LSL_STRING]` (for Lua, `ll.GetOwner` is rewritten to `llGetOwner`); Luau library namespaces (`bit32, buffer, coroutine, debug, math, os, string, table, utf8`) go to `https://luau.org/library`; everything else Lua goes to `https://wiki.secondlife.com/wiki/Lua_Alpha`. The dynamic help floater title/topic switches to "LUA REFERENCE"/`Lua_Alpha` (`llpreviewscript.cpp` diff 421-423). The "Insert..." function combo is populated from the active keyword set; the Firestorm legacy libraries (`scriptlibrary_preproc/ossl/aa.xml`) are only loaded for LSL (`:752-780`).

### 1.10 External-file edit workflow

Two modes, selected by `ExternalEditorTightIntegration` (`settings.xml:27645`):

- **Legacy external editor** (`LLScriptEdCore::openInExternalEditor`, `llpreviewscript.cpp:1673-1760`): writes `<tmpdir>/sl_script_<sanitised name>_<md5>.lsl|.luau` (`getTmpFileName`, `:2269-2282`; the hash is `md5(object_id + "_" + item_id)` from `LLScriptEditorWSServer::buildScriptSubscriptionId`, `:2284`), creates an `LLLiveLSLFile` (1 s poll `LLLiveFile`, deletes the file in its destructor, `:220-243`), subscribes the editor with the bridge if enabled (`startWebsocketServer`, `:2391-2401`), then launches the `ExternalEditor` command (`settings.xml:18580`, `"%s"` substituted; the Scripting Settings floater Browse button builds `"<exe>" "%s"`, `indra/newview/llfloaterscripting.cpp:125-132`). When the file changes, `onExternalChange` loads it and saves/compiles without re-syncing (`:2346-2362`). Compile results are also written to `<tmpfile>.log` with a timestamp (`logErrorsToFile`, `:2302-2345`) and that log file is tracked by a second `LLLiveLSLFile` so it is deleted with the editor. `sync()` rewrites the temp file after an in-viewer save, with `ignoreNextUpdate` to avoid loops.
- **Tight integration**: no temp file; the server is ensured running and VS Code is launched with `vscode://lindenlab.sl-vscode-plugin/connect?port=<port>&object=<root>&script=<item>` via `code --open-url` (`llscripteditorws.cpp:385-445`; Windows goes through `cmd.exe /c` because `code` is a `.cmd`). The extension opens the script as a virtual `sl://` document.
- Save/Load to file: picker filters `*.lsl;*.lua;*.luau` on all platforms (`indra/newview/llfilepicker.cpp:72,753,895`); Save-as appends `.luau` for Lua / `.lsl` for LSL (`llpreviewscript.cpp:1942-1965`).
- Script recovery and 60 s backup timer: see 1.4; `onBackupTimer` now renames the backup if the language changed (`:2222-2239`).

### 1.11 Preprocessor bypass for Lua

Every `_NACL_LSLPreprocessor` check in the editor is guarded by `!mEditor->getIsLuauLanguage()` (`llpreviewscript.cpp:643,752,819,1107,1125,1233,1644,1720,1727,1801`): no "Preprocessed" tab for Lua (`processKeywords` hides it, `:702-712` region), `setScriptText`/`getScriptText` pass raw text, `doSave` skips `preprocess_script`, external-edit files are written unprocessed. `LLPreviewLSL::saveIfNeeded` applies the `//mono`/`//lsl2` directive only for `mono`/`lsl2` targets (`:2787-2800`). The compile queue only sends LSL items to the preprocessor (`llcompilequeue.cpp:565-566`).

### 1.12 Templates

The LL default "Hello, Avatar" template was removed; new scripts come from the simulator per subtype (1.4). Firestorm's per-account custom template is retained but subtype-gated (`llpanelcontents.cpp:398-402`). The `luau.txt` file in `indra/newview/` is not a template; it is the Luau/Lua MIT licence text shipped for attribution.

---

## 2. IDE bridge protocol

### 2.1 Transport

- Library: WebSocket++ 0.8.2 header-only (`autobuild.xml:3130-3168`, `indra/cmake/websocketpp.cmake`), `asio_no_tls` config (`indra/llcorehttp/llwebsocketmgr.cpp:40`). **No TLS**; plain `ws://`.
- Bind: `ExternalWebsocketSyncLocal` (default true) binds `127.0.0.1:<port>`; false binds all interfaces (`llwebsocketmgr.cpp:218-245`). IPv6 loopback is not bound. Port `ExternalWebsocketSyncPort` default **9020** (`settings.xml:27623`; `LLScriptEditorWSServer::DEFAULT_SERVER_PORT`, `llscripteditorws.h:209`). Server name `script_editor_server`.
- Origin policy: WebSocket++ validate handler refuses (HTTP 403) any upgrade whose `Origin` header is non-empty (`llwebsocketmgr.cpp:326-344`, `acceptOrigin` default `origin.empty()` in `llwebsocketmgr.h:257`). This blocks browsers, including the viewer's own CEF; a native client (VS Code extension) sends no Origin. Tested in `indra/llcorehttp/tests/wsserver_test.cpp:232-411`.
- Threading: one `std::thread` per server running `io_service::run_for(100ms)` with a stop flag (`llwebsocketmgr.cpp:251-299`); websocketpp callbacks run on that thread; `WSServer::stop()` closes all connections with code 1001 "Server shutting down", stops ASIO, joins (warns after 5 s) (`:473-520`). `LLWebsocketMgr::update()` is called every frame from the main loop (`indra/newview/llappviewer.cpp:5772`) and removes any server whose `update()` returns false; `LLScriptEditorWSServer::update()` returns false when `ExternalWebsocketSyncEnable` is turned off, which stops the server (`llscripteditorws.cpp:459-462`).
- Messages: text frames only; fragmented frames are not reassembled by the wrapper (`llwebsocketmgr.cpp:395`). No explicit maximum message size is configured beyond websocketpp defaults. No rate limiting is implemented despite the `RATE_LIMITED` error code being defined.
- Connection limits: none; each connection is tracked by handle (`llwebsocketmgr.cpp:587-624`), tested with multiple simultaneous connections (`wsserver_test.cpp:409`).

### 2.2 JSON-RPC layer (`indra/llcorehttp/lljsonrpcws.*`)

- JSON-RPC 2.0 envelopes built by `makeEnvelope` (`lljsonrpcws.cpp:539-569`): `jsonrpc:"2.0"` always; notifications omit `id`; responses always carry `id` (null for parse errors). Outgoing request ids are strings `rpc_<n>` from a process-wide counter starting at 1000 (`:481-491`).
- Validation (`:350-426`): `jsonrpc` must be `"2.0"`; `method` string; `params` array or object if present; responses need `id` and exactly one of `result`/`error`; `error` must be an object with `code` and `message`.
- **Batch arrays are rejected** with `-32600 "Invalid Request: Batch requests are not supported"` (`:98-109`).
- Dispatch (`:155-316`): **unauthenticated connections get `-32002 Unauthorized` for every request and silence for notifications, without revealing whether the method exists** (`:166-176`); responses to the server's own calls are still processed (that is how the handshake answer gets in). Sync handlers run on the I/O thread; `registerAsyncMethod` handlers are posted to the main loop and run inside an `LLCoros` coroutine, and **cannot be invoked as notifications** (`-32600`) (`:199-270`). If the main loop is unavailable (shutdown) the request fails with `-32603 "Viewer is shutting down; cannot service <method>"`.
- Client-side (`call`) timeouts: default 120 s, swept once per second from the main thread; expired callbacks get `{code:-32001, message:"Request timed out"}`; on close every pending callback gets `-32000 "Connection closed"` exactly once (`:428-461`, `:55-75`).
- Built-in methods registered per connection (`:739-807`): `system.listMethods` -> array of method names; `system.ping` -> `{pong:"pong", timestamp?: <echo>, server_time: <ms since epoch>}` (`llscripteditorws.cpp:1312-1327`); `system.getVersion` -> `{client_name: <viewer channel>, client_version}` (`:1329-1337`); `system.status` -> `{status:"OK"}`; global `system.getStats` -> `{server_name, connection_count, is_running, global_method_count, total_requests_handled, total_notifications_sent, uptime}`.

Error codes (`lljsonrpcws.h:124-142`):

| Code | Name | Used by |
|---|---|---|
| -32700 | PARSE_ERROR | unparseable JSON |
| -32600 | INVALID_REQUEST | batches, async-as-notification, duplicate `object.item.create` |
| -32601 | METHOD_NOT_FOUND | unknown method (only once authenticated) |
| -32602 | INVALID_PARAMS | most argument/lookup failures (object/prim/item not found, wrong type, unknown command) |
| -32603 | INTERNAL_ERROR | cap missing, upload failed, queue could not open, asset fetch failed |
| -32000 | CONNECTION_CLOSED | pending call callbacks on close |
| -32001 | REQUEST_TIMEOUT | call timeouts; async handlers waiting on asset/upload/create |
| -32002 | UNAUTHORIZED | any request before handshake success |
| -32003 | FORBIDDEN | permission denials, RLVa restrictions, object not published, save-back unavailable |
| -32004..-32007 | RATE_LIMITED, SERVICE_UNAVAILABLE, MESSAGE_TOO_LARGE, INVALID_SESSION | defined, not emitted |

### 2.3 Authentication / handshake (`llscripteditorws.cpp:2831-3124`)

1. On WebSocket open the server (main thread) generates a 16-byte random secret via `RAND_bytes` and writes it as a UUID string into a **new** file `<tmpdir>/sl_script_challenge_<random uuid>.tmp` readable only by the current user: Windows explicit DACL + `CREATE_NEW` (`:3046-3120`); POSIX `open(O_WRONLY|O_CREAT|O_EXCL|O_CLOEXEC, 0600)` with full-write loop and unlink on failure (`indra/newview/llscripteditorposixfile.h:14-52`, test `indra/newview/tests/llscripteditorposixfile_test.cpp`). If the file cannot be written the connection is closed with `session.disconnect` reason `INTERNAL_ERROR`.
2. Server **calls** (request, expects a response) `session.handshake` with params:
   ```
   server_version:"1.0.0", protocol_version:"1.0",
   viewer_name, viewer_version, agent_id, agent_name,
   challenge: "<absolute path of challenge file>",
   languages: ["lsl","luau"], syntax_id: <uuid>,
   features: {live_sync:true, compilation:true, syntax_cache:true, commands:true, unified_diagnostics:true}
   ```
   with a 30 s timeout (`HANDSHAKE_TIMEOUT`, `:117`).
3. Client must respond with result `{challenge_response: "<file contents>", client_name, client_version, protocol_version:"1.0", languages:[...], features:{name:bool,...}, script_name?, script_language?}`. The server deletes the challenge file as soon as any answer/timeout arrives (`:2903-2906`). **[external]** The extension reads the path with `fs.readFile(path,'utf8')` and returns it as `challenge_response`; it sends `protocol_version "1.0"`, `client_name "sl-vscode-plugin"`, `languages ["lsl","luau"]`, features `{live_sync, error_reporting, unified_diagnostics, debugging:false, breakpoints:false, object_publish, commands}` and connects to `ws://localhost:<port>` (`synchservice.ts`).
4. Wrong/missing `challenge_response` -> `session.disconnect {reason:2 PROTOCOL_ERROR, message:"Invalid challenge response"}` + close 1000. Timeout -> reason 3 TIMEOUT. A protocol-version mismatch is only logged.
5. On success the connection is marked authenticated and the server sends notification `session.ok` (no params) (`:3009-3012`). Only then are requests served and broadcasts delivered (`notifyAll`/`notifyConnection` filter on `isAuthenticated()`, `:2411-2443`).

`DisconnectReason` values (`llscripteditorws.h:60-67`): 0 NORMAL, 1 EDITOR_CLOSED, 2 PROTOCOL_ERROR, 3 TIMEOUT, 4 INTERNAL_ERROR.

### 2.4 Methods the client may call (all registered per connection in `setupConnectionMethods`, `llscripteditorws.cpp:661-795`; all async/main-thread except `command.list`)

Common conventions: UUIDs are JSON strings; `prim_id` means the prim holding the item, `object_id`/`root_id` the linkset root; the object must be **published** for any item operation; permission masks are integers.

| Method | Params | Result / errors | Impl |
|---|---|---|---|
| `language.syntax.id` | none | `{id: <syntax uuid>}` | `:1339` |
| `language.syntax` | `{kind: "defs.lua"\|"defs.lsl"}` | `{id, defs: <keywords LLSD>, success}`; InvalidParams on unknown kind; InternalError if unavailable | `:1347` |
| `language.syntax.cache` | none | `{files:[names], success}` | `:1383` |
| `language.syntax.get` | `{filename, as_json?: bool}` | `{content: <string or LLSD>, success}` | `:1397` |
| `script.subscribe` | `{script_id, script_name?, script_language?}` | `{script_id, success, status, message, object_id?, root_id?, item_id?}`; status 0 OK, 1 invalid editor, 2 no subscription, 3 already subscribed, 4 internal | `:1445` |
| `script.unsubscribe` | `{script_id}` | empty | `:1507` |
| `script.list` | none | `{temp_dir, script_ids:[...], success}` | `:1519` |
| `object.request` | `{object_id}` | `{success}`; Forbidden if no modify perm; triggers `object.publish` notification later | `:1538` |
| `object.unpublish` | `{object_id}` | `{success, object_id}`; emits `object.unpublish` reason `manual` | `:1975` |
| `object.list` | none | `{objects:[<published object>]}` | `:797` |
| `object.content.get` | `{prim_id, item_id}` | `{success, prim_id, item_id, content, encoding:"utf-8"}`; scripts need copy+modify, notecards no perms; asset fetched via `getInvItemAsset` (30 s timeout); notecards unwrapped with `LLNotecard` | `:1621` |
| `object.content.save` | `{prim_id, item_id, content, vm?: "luau"\|"mono"\|"lsl2", running?: bool}` | scripts: `{success, prim_id, item_id, compiled, diagnostics?}` (60 s); notecards: `{success, prim_id, item_id}` (30 s). Empty content is InvalidParams. `vm:"luau"` on an LSL item becomes `lsl-luau`; without `vm`, the item's runtime decides, else content sniffing | `:1730,1748,1888` |
| `object.item.create` | `{prim_id, type:"script"\|"notecard", name, vm?: "luau"\|"mono"\|"lsl2" (scripts, required), text? (notecards)}` | `{item_id, name, description, type, prim_id, subtype?, vm?, permissions{owner,next_owner}, creator_id}`; one create in flight per prim (`-32600`); 30 s timeout; notecards require the `CreateTaskInventoryItem` cap, scripts fall back to UDP `RezScript` | `:1996` |
| `object.item.delete` | `{prim_id, item_id}` | `{success, prim_id, item_id}`; optimistic local remove + `object.update`, then authoritative refresh | `:1946` |
| `object.item.modify` | `{prim_id, item_id, name?, description?, permissions?:{next_owner}}` | `{success, prim_id, item_id}`; uses `UpdateTaskInventory` | `:1120` |
| `object.modify` | `{prim_id, name?, description?, permissions?:{next_owner}}` | `{success, prim_id}`; sends `ObjectName`/`ObjectDescription`/`ObjectPermissions` messages, then requests properties so an `object.update` follows | `:1034` |
| `object.script.set_running` | `{prim_id, item_id, running}` | `{success}`; `SetScriptRunning` message | `:804` |
| `object.script.reset` | `{prim_id, item_id}` | `{success}`; `ScriptReset` message | `:849` |
| `command.list` | none | `{commands:[{command, description, params?}]}` | `:1249` |
| `command.execute` | `{command, params}` | the command's result, or InvalidParams for unknown commands | `:1231` |

Registered viewer commands (`llscripteditorws.cpp:232-301`):

- `viewer.teleport {object_id}` -> `{success}` (teleports agent to the object's global position).
- `viewer.camera.focus {object_id}` -> `{success}` (same as context-menu Zoom In; `handle_zoom_to_object`).
- `viewer.object.save_back_to_contents {object_id}` -> `{success, result:{object_id}}`; requires `can_save_back` and a source task (`derez_objects(DRD_SAVE_INTO_TASK_INVENTORY)`, `llviewermenu.cpp:6263`) (`:1183`).
- `viewer.script.reset_all {object_id}` -> `{success, object_id, queued:true}` via a reset queue floater (`:892`).
- `viewer.script.recompile_all {object_id, target:"luau"|"lsl2"|"mono"|"auto"}` -> `{success, object_id, target, queued:true}`; `luau` is mapped to the `auto-luau` queue target so LSL items compile as `lsl-luau` (`:954`).

**[external]** The extension only registers one reverse command, `editor.show_message {message, level}`; the viewer can call `command.execute` on a client that advertised the `commands` feature (`sendCommandExecute`, `:1268-1293`; no viewer code currently invokes it).

### 2.5 Notifications and calls from the viewer to the client

| Method | Direction/kind | Params | Source |
|---|---|---|---|
| `session.handshake` | request (expects reply) | see 2.3 | `:2855` |
| `session.ok` | notification | none | `:3012` |
| `session.disconnect` | notification, then close(1000) | `{reason:int, message}` | `:2944` |
| `language.syntax.change` | broadcast | `{id: <new syntax uuid>}` (only when it actually changed) | `:1295` |
| `script.unsubscribe` | to the subscribed connection | `{script_id}`; sent when the viewer editor closes | `:2240` |
| `script.compiled` | to the subscribed connection | `{script_id, success, running, diagnostics?:[...]}` | `:2249` |
| `object.publish` | broadcast | `{object: <published object>}` | `:2536-2614` |
| `object.update` | broadcast | see 2.7 | several |
| `object.unpublish` | broadcast | `{object_id, reason?}` reasons seen: `manual`, `republish`, `deleted`, `linked`, `user` | `:2802` |
| `runtime.debug` / `runtime.error` | broadcast | see 2.8 | `:2349` |
| `command.execute` | request | `{command, params}` (only to clients with feature `commands`) | `:1268` |

### 2.6 Diagnostics structure

Produced in two places with the same shape (`saveScript` response `diagnostics`, `:1816-1868`; `script.compiled` notification, `:2265-2335`). Each entry: `{row:int, column:int, level:string, message:string, format?:"lsl"}`.

- Lua: compiler strings `<source>:<line>: <message>` (regex `^([^:]*):([0-9]+):\s*(.*)$`, `:123`) -> `row = line` (already 1-based), `column = 0` (Luau gives none), `level = "ERROR"`.
- LSL: `(<line>, <col>) : <SEVERITY> : <message>` (regex `:126`) -> `row = line+1`, `column = col+1`, `level = SEVERITY` (e.g. `ERROR`, `WARNING`), `format:"lsl"`.
- Unparseable error strings get `row:0, column:0`, the raw text as `message`.

Script association is the subscription `script_id` = `md5hex(<prim uuid>_<item uuid>)` (`buildScriptSubscriptionId`, `:373-383`) for editor-driven compiles, or `prim_id`/`item_id` in the `object.content.save` reply. The viewer's own compile results feed both the error list UI and the bridge (`LLPreviewLSL::finishedLSLUpload`/`failedLSLUpload`, `LLLiveLSLEditor::finishLSLUpload` which adds `is_running`; `llpreviewscript.cpp:2703-2754,3424-3449`).

**[external]** The extension's `handleCompilationResult` reads `success`, `running`, `diagnostics[].row/column/level/message`, matching the above.

### 2.7 Published-object model (`indra/newview/llpublishedobjectmgr.*`)

**Published object payload** (`buildPublishedObjectLLSD`, `llpublishedobjectmgr.cpp:791-831`; `buildObjectListLLSD`, `:833-891`):
```
{ object_id, object_name, object_description, owner_id, region?, can_save_back?,
  permissions?: {owner:int, next_owner?:int},
  inventory: [item...],
  linked_objects?: [ { link_id, link_number (2..n), link_name, link_description,
                       permissions?, inventory:[item...] } ] }
```
**Item entry** (`buildPrimInventoryLLSD`, `:673-736`, scripts and notecards only):
```
{ item_id, name, description, type:"script"|"notecard",
  subtype?:int (0 LSL,1 Lua), vm?:"lsl2"|"mono"|"luau", running?:bool, faulted?:bool,
  permissions:{owner,next_owner}, creator_id }
```
Permissions come from the selection node if an `ObjectProperties` reply exists, else from object flags with `next_owner` omitted (`:746-771`). `can_save_back` requires a valid select node with `mFromTaskID` and a non-attachment root (`:773-789`).

**Publish sequence** (`LLScriptEditorWSServer::publishObject`, `llscripteditorws.cpp:2470-2519`): requires modify permission; re-publishing unpublishes first (`republish`); collects root+children; requests `ObjectPropertiesFamily` for the root; creates a `PendingPublish` with one `LLVOInventoryListener` per prim (`beginPendingPublish`, `llpublishedobjectmgr.cpp:529-541`); calls `requestInventory()` on each prim; when every prim reported inventory (`handlePrimInventoryReadyEvent`) it builds the payload, finalises `PublishedObjectInfo` (merging any names/descriptions that arrived while pending, `:1366-1409`), broadcasts `object.publish`, and re-requests properties so the first property reply yields an `object.update` with real names/`can_save_back`.

**Update rules** (all broadcast as `object.update` with `object_id` of the root):

- Root inventory change -> `{object_id, inventory:[...]}`; child inventory change -> `{object_id, changes:{linked_objects:{modified:[{link_id, inventory:[...]}]}}}` (`reconcileInventoryChanged`, `:1062-1113`). Listeners are the per-prim `LLPublishedPrimListener` (`:103-141`). A pending `object.item.create` is completed from the same event (`:1048-1057`).
- Property change (from `ObjectProperties`/`ObjectPropertiesFamily`/NV pairs; hooks `llselectmgr.cpp:6392-6396,6485-6489`, `llviewerobject.cpp:792-803`) -> root: `{object_name?, object_description?, can_save_back?}`; child: `changes.linked_objects.modified[{link_id, link_name?, link_description?}]`; only emitted when something differs (`applyPropertyChange`, `:1115-1195`). A changed inventory serial on a prim triggers a `dirtyInventory()+requestInventory()` unless one is already in flight (`markPrimInventorySerialAndDetectChange`, `:1305-1338`; `llscripteditorws.cpp:2765-2781`).
- Permissions received -> root: `{permissions, owner_id?}`; child: `modified[{link_id, permissions}]` (`applyPermissionsChange`, `:1197-1250`).
- Linkset child added (`LLViewerObject::addChild`, `llviewerobject.cpp:1005-1021`): if the child was itself published it is unpublished with reason `linked`; the child is appended with a listener and marked as a "new child"; its inventory is requested; a 5 s safety timer is scheduled (`LINKSET_ADD_FLUSH_DELAY`). When the inventory arrives the child is considered ready; when the last pending child is ready (or the timer fires) a full `{object_id, linked_objects:[...]}` update is broadcast (`flushLinksetUpdate`, `llscripteditorws.cpp:2686-2699`).
- Child removed (`removeChild`, `llviewerobject.cpp:1067-1077`): removed from the prim list and listeners, link numbers renumbered, update flushed after 0.2 s (`LINKSET_REMOVE_FLUSH_DELAY`). Only one flush timer per root exists at a time.
- Root deleted (`markDead`, `llviewerobject.cpp:441-446`) -> `object.unpublish` reason `deleted`.
- `object.unpublish` cleanup (`cleanupObjectStateForUnpublish`, `:1411-1423`): drops pending publish, listeners, record, flush timer, and pending children.

**No duplicate subscriptions / stale state rules**: one `EditorSubscription` per `script_id`, bound to at most one connection (`updateScriptSubscription` returns ALREADY_SUBSCRIBED if the existing connection is alive, `llscripteditorws.cpp:599-635`); a closed connection clears `mConnectionID` on all its subscriptions (`unsubscribeConnection`, `:583-597`) so a reconnecting client can re-subscribe; dead editor handles are pruned on the next subscribe. Only one `object.item.create` per prim at a time (`reservePendingItemCreate`, `llpublishedobjectmgr.cpp:643-671`), released by an RAII guard (`llscripteditorws.cpp:2114-2117`). Inventory request start times (`mInventoryRequestStartSec`) prevent duplicate `requestInventory` calls. A published object is re-published (not duplicated) if requested again.

### 2.8 Runtime messages (owner say / debug / run-time errors)

Hook: `LLFloaterIMNearbyChatHandler::processChat` forwards `CHAT_TYPE_DEBUG_MSG` and `CHAT_TYPE_OWNER` chat when the server is enabled and `ExternalWebsocketForwardDebug` (`settings.xml:27656`, default on) is set (`indra/newview/llfloaterimnearbychathandler.cpp:620-635`). `LLPublishedObjectMgr::ingestRuntimeChat` (`llpublishedobjectmgr.cpp:171-242`):

- A plain message with no pending burst is emitted immediately as `runtime.debug`.
- A message whose first line matches `^(.+?)\s+\[script:([^\]]+)\]\s+Script run-time error` starts a burst keyed by (from_id, from_name, VM, channel); subsequent fragments from the same source within 1 s (`FRAGMENT_TIMEOUT`, flushed by a 0.25 s timer) are concatenated with `\n` (`RuntimeEventAggregator`, `:353-517`). The VM is Luau if the named script's inventory item has runtime `luau`.
- `buildRuntimeChatEvent` (`:249-351`) resolves prim/root, item id by script name, and parses the error: Luau `source:line: message`; LSL `(line, col) : SEV : message` (1-based conversion) or, failing that, the first non-header line (e.g. `Math Error`). Remaining lines after the header become `stack` (array of raw strings).
- Sent only if the root is published or the script has an editor subscription (`sendRuntimeEvent`, `llscripteditorws.cpp:2349-2409`). Payload:
  ```
  { script_id?, object_id (root), prim_id, item_id, object_name, message,
    item:{root_id, prim_id, item_id, name, language:"luau"|"lsl"},
    channel:"debug"|"owner_say",
    // runtime.error only:
    error, line, column, stack:[...] }
  ```

### 2.9 Lifecycle, reconnection, shutdown

- Server creation is lazy (`ensureServerRunning`, `llscripteditorws.cpp:327-371`): triggered by Edit-in-external-editor, "Explore in IDE" (`llpanelcontents.cpp:474-508`: toggles publish if a client is connected, otherwise launches VS Code), the Build > Scripts > "Script Editor Server" toggle (`llviewermenu.cpp:6999-7026`), or `launchVSCode`. User notifications `ExternalEditorServerStarted/Failed/Stopped` (`notifications.xml:15100-15112`).
- `onStarted` subscribes to syntax-id changes; `onStopped` (after the I/O thread joined) clears published objects, subscriptions and connection maps silently (`:464-498`).
- Connection open/close events from the I/O thread are marshalled to the main loop (`LL::WorkQueue::postToMainLoop`, `indra/llcommon/workqueue.h:267`) before touching `mActiveConnections` (`:500-530`). All handler state is main-thread-owned.
- Editor close: `LLScriptEdContainer::~LLScriptEdContainer` sends `script.unsubscribe` to the client and removes the subscription (`llpreviewscript.cpp:2404-2413`).
- Pending async operations at shutdown: coroutine handlers check `isConnected()` before replying; the handshake callback and async launches no-op when the main loop is gone; `LLWebsocketMgr::cleanupSingleton` stops all servers (`llwebsocketmgr.cpp:66-69`).
- Reconnection is client-driven: the viewer never dials out. **[external]** the extension exposes connect/disconnect commands and `disconnectDelayMs`/`disposeDelayMs` settings.

### 2.10 Preference toggles (`indra/newview/app_settings/settings.xml`)

| Key | Line | Default | Effect |
|---|---|---|---|
| `ExternalWebsocketSyncEnable` | 27612 | 1 | master switch; server refuses to start/run without it |
| `ExternalWebsocketSyncPort` | 27623 | 9020 | listen port |
| `ExternalWebsocketSyncLocal` | 27634 | 1 | bind loopback only |
| `ExternalEditorTightIntegration` | 27645 | 0 | VS Code URI launch instead of temp file + external editor; forces sync enable (`llfloaterscripting.cpp:101-113`) |
| `ExternalWebsocketForwardDebug` | 27656 | 1 | forward owner/debug chat and runtime errors |
| `ScriptEditorDisableSyntaxHighlight` | 27667 | 0 | perf escape hatch |
| `ExternalEditor` | 18580 | "" | legacy external editor command line |
| `_NACL_LSLPreprocessor`, `FSSaveInventoryScriptsAsMono` | 21318, 1053 | FS | preprocessor on/off; default agent-inventory target |

UI: floater `scripting_settings` (`indra/newview/skins/default/xui/en/floater_scripting_settings.xml`, registered `llviewerfloaterreg.cpp:596`, menu `menu_viewer.xml:1755`).

### 2.11 RLVa guards on IDE writes

`ensureScriptObjectEditable()` (`llscripteditorws.cpp:90-103`) throws `-32003 Forbidden` if the root is a locked attachment (`gRlvAttachmentLocks.isLockedAttachment`) or the agent is seated on it under `@unsit`/`@sittp`. It is applied in `validatePublishedItem` for any `PERM_MODIFY` operation (content save, item delete, item modify; `:1605-1616`) and directly in `set_running`, `reset`, `reset_all`, `recompile_all`, `object.modify`, `object.item.create`, and `save_back_to_contents` (`:814,858,901,974,1051,2011,1206`). Reads (`object.content.get`, publish) are not restricted. This mirrors the native editor guards (`llpreviewscript.cpp:3302,3332,3469`) and `LLPanelContents` creation guards.

### 2.12 Cross-check with the VS Code extension **[external]**

- `installation.md` / `USER_GUIDE.md` / `features-overview.md`: port setting `slVscodeEdit.network.websocketPort` default 9020; "Explore in IDE" and "Build > Scripts > Script Editor Server" match the viewer; menu commands (Start/Stop, Restart, Select VM, Rename, Delete, Save Back to Contents, Teleport To, Zoom In, New File, Unpublish) map one-to-one onto `object.script.set_running`, `object.script.reset`, `object.content.save{vm}`, `object.item.modify`, `object.item.delete`, `viewer.object.save_back_to_contents`, `viewer.teleport`, `viewer.camera.focus`, `object.item.create`, `object.unpublish`. "Teleport To / Zoom In / Save Back" appear only when `command.list` advertises them.
- `src/synchservice.ts`: literal `script.subscribe` with `script_id, script_name, script_language`, reading `success, root_id, object_id, item_id, message`; handshake fields `viewer_name, viewer_version, agent_id, agent_name, languages, syntax_id, features, challenge`; notifications `session.disconnect{reason,message}`, `script.unsubscribe{script_id}`, `language.syntax.change{id}`, `script.compiled{script_id,...}`, `runtime.debug`, `runtime.error`, `object.publish{object}`, `object.unpublish{object_id}`, `object.update{object_id,...}`, `command.execute`, `command.list`. Other wire names are hidden inside the `#sl-ide-ws-client` package, so they could not be verified character-for-character; the viewer code is authoritative.

---

## 3. UI-dependent versus UI-independent

**UI-independent (portable as-is in spirit)**

- Transport + JSON-RPC (`indra/llcorehttp/llwebsocketmgr.*`, `lljsonrpcws.*`): depends only on LLSD, boost::json, OpenSSL `RAND_bytes`, threads. No viewer UI.
- Bridge protocol semantics (`llscripteditorws.*` minus the floater lookups): handshake, challenge file, auth gate, method table, diagnostics parsing, runtime-message aggregation, commands.
- Published-object model (`llpublishedobjectmgr.*`): pure state machine over "object/prim/item" records plus the inventory-change listener. Its inputs are world-state events (inventory ready/changed, properties, permissions, link add/remove, object death, chat).
- Inventory metadata: subtype flag, `runtime` string, `running`/`faulted` bits, their LLSD/legacy serialisation, icon selection rule, creation payload for `CreateTaskInventoryItem`, `RezScript` fallback with subtype, `NewScriptInfo.TemplateID`.
- Compile flow: `UpdateScriptTask`/`UpdateScriptAgentInventory` body with `target`, `is_script_running`, `experience`; result `compiled`/`errors[]`; error regexes; compile-queue target resolution and language-compatibility rule; `ScriptRunningReply` decoding.
- Syntax definitions cache: sim feature / cap discovery, cache directory per syntax id, `files` map format, version check.
- Language detection heuristics (`is_lua_script`), help URL mapping, temp-file naming (`sl_script_<name>_<md5>.<ext>`), backup naming, exclusive challenge-file writes.

**Firestorm/LL UI-dependent (must be re-created in sl-ui)**

- `LLScriptEdCore` / `LLPreviewLSL` / `LLLiveLSLEditor` floaters (XUI `panel_script_ed.xml`, `floater_live_lsleditor.xml`, `floater_script_preview.xml`): compile-target combo, running checkbox, reset button, experience panel, error list with click-to-line, Insert function combo, help floater (CEF browser), font selection, F1/Home behaviour, preprocessor tab.
- `LLScriptEditor` / `LLKeywords` highlighting: tied to `LLTextEditor` segments, `LLStyle`, `LLUIColorTable` colour names (`SyntaxLsl*`), and the worker-thread slicing designed around LLTextBase reflow. The *lexical rules* are reusable; the segment machinery is not.
- Build floater Contents tab (`floater_tools.xml`, `llpanelcontents.cpp`): New Script flyout, New Notecard, "Explore in IDE" toggle, Reset Scripts.
- Scripting Settings floater, menu entries, notifications/toasts, `LLFloaterReg` lookups used by the bridge to push saved text into an open editor (`llscripteditorws.cpp:1870-1883,1928-1941`) and to read `isLuauLanguage` for diagnostics format (`:2252-2263`).
- Compile/reset queue floaters (`LLFloaterCompileQueue`/`LLFloaterScriptQueue`) used by `recompile_all`/`reset_all`.
- External-editor launch (`LLExternalEditor`, `LLProcess`), file pickers, chat handler hook point.

The seam to draw: **a `ScriptingService` that owns the bridge server, published-object model, syntax cache, compile submission, and runtime-chat aggregation, exposing events; the editor UI and build-tools UI are consumers of that service.** In the C++ code the bridge reaches into UI (`LLFloaterReg::findTypedInstance`, `LLScriptEdContainer::getScriptEdCore`) in exactly three places (save-back into open editor for scripts and notecards; language lookup for diagnostics); in Rust those become events/queries on the service.

---

## 4. Rust port recommendation

### 4.1 Crate placement

- `sl-inventory`: `ScriptSubtype {Lsl=0, Lua=1}` in the flags low byte, `runtime: Option<ScriptRuntime>` (`Lsl2|Mono|Luau`), `running`/`faulted` on task items, LLSD/legacy (de)serialisation (`script.runtime`, `metadata.script.runtime`), icon enum variant. Port the 7 `llinventory_runtime_test` cases.
- `sl-comms` (region/caps layer): cap names `CreateTaskInventoryItem`, `ScriptDefinitions`, `UpdateScriptTask`, `UpdateScriptAgentInventory`, `UpdateNotecardTaskInventory`; sim features `LuaScriptsEnabled`, `LSLSyntaxId`; `ScriptRunningReply` LLSD fields `Mono/Luau/LuauLanguage`; `SetScriptRunning`, `ScriptReset`, `RezScript` (+`NewScriptInfo`), `ObjectName/Description/Permissions`, `UpdateTaskInventory` senders; the two-stage asset upload with `target`.
- New `sl-scripting` crate (UI-free), modules:
  - `compile`: target enum (`Lsl2|Mono|LslLuau|Luau` serialising to the four strings), upload request/response, diagnostics parser (both regexes), language detection, compile-queue policy.
  - `syntax`: definitions cache (per-syntax-id directory, `files` map, version 2 check), typed model of `*_keywords.xml` (`controls/types/constants/events/functions`), Lua/LSL lexer (see 4.3).
  - `published`: the published-object state machine (port `LLPublishedObjectMgr` nearly 1:1; it is already decoupled), runtime-chat aggregator with the 1 s burst window.
  - `bridge`: WebSocket server, JSON-RPC, handshake/challenge, method table, commands.
  - `extedit`: temp-file naming, watcher, `.log` writer, VS Code URI launch, backup/recovery naming.
- `sl-ui`: script editor view, build-tools Contents tab, scripting settings, notifications; consumes `sl-scripting` events.

### 4.2 Crates

- WebSocket: `tokio-tungstenite` (server side via `tokio::net::TcpListener` bound to `127.0.0.1:port`, `accept_hdr_async` to inspect and reject `Origin`). Plain TCP, no TLS, matching the viewer.
- JSON: `serde`/`serde_json`; model the envelope as an enum (`Request{id,method,params}`, `Notification{method,params}`, `Response{id,result|error}`). Hand-roll the dispatcher (about 300 lines) rather than pulling `jsonrpsee`: batches must be rejected, unauthenticated requests must get `-32002` without method discovery, and async handlers must hop to the main/world thread. If a crate is preferred, `jsonrpc-core`/`jsonrpsee` both support custom transports but need a wrapper for the auth gate.
- Challenge file: `rand::rngs::OsRng` 16 bytes -> UUID string; `std::fs::OpenOptions::new().write(true).create_new(true)` plus `std::os::unix::fs::OpenOptionsExt::mode(0o600)` on Unix and the `windows` crate (`CreateFileW` with explicit DACL via `SetEntriesInAclW`, as the C++ does) on Windows. Delete the file on first answer/timeout.
- Timers: tokio for handshake (30 s), request (120 s), asset/upload timeouts (30/60/30/30 s), linkset flush (5 s / 0.2 s), runtime flush (0.25 s / 1 s).
- Main-thread hop: every handler that touches world state should send a message to the world/sim actor and await a oneshot, equivalent to `postToMainLoop` + coroutine.

### 4.3 Definitions and highlighting

- Consume the same `lsl_definitions` package. Options: (a) vendor the 0.6.12 tarball as a build dependency and parse `lsl_keywords.xml`/`lua_keywords.xml` at build time into a compact table (`build.rs` + `quick-xml`), keeping the runtime loader for region-supplied `ScriptDefinitions` overrides; (b) parse at runtime only. The viewer needs the runtime loader anyway (regions push new definitions keyed by syntax id and the bridge serves them verbatim to the IDE via `language.syntax*`), so (b) with a cached parsed form is simplest; `lsl_definitions.yaml`/`slua_definitions.yaml` are richer (typed signatures) and could feed completion later.
- Highlighting: a small hand-written lexer reproducing section 1.7 is about 400 lines and gives exact parity (long brackets, dotted namespaces, interpolated strings). `tree-sitter-luau` would give structural highlighting and folding but adds a C dependency and does not know SL's `ll.*`/`constants` classes; it could be layered later. Keyword classes map to the definition groups (`controls`, `types`, `constants`, `events`, `functions`), with tooltips built as in `processTokensGroup`.
- Keep the Luau 1-based / LSL 0-based line-number convention in the UI only; store 0-based internally.

### 4.4 Estimated size

- C++ reference: bridge 3.1k + model 1.4k + transport/RPC 2.5k + syntax cache 0.6k + keyword lexer changes 0.6k + editor integration (~1.2k of diff in `llpreviewscript.cpp`/`llscripteditor.cpp`) + inventory/compile/queue plumbing (~0.6k).
- Rust estimate: `sl-scripting` 4.5-6k lines (bridge 1.5k, published model 1k, JSON-RPC/ws 0.8k, diagnostics/compile 0.5k, syntax cache + lexer 1k, extedit 0.4k) plus tests; `sl-inventory`/`sl-comms` additions ~0.5k; `sl-ui` editor and build-tab work depends on the toolkit (the compile-target combo, running/reset controls, error list, help, external-edit menu are the minimum). Roughly 3-5 engineer-weeks for the service crate with tests, excluding the editor widget.

### 4.5 Security considerations for a local WebSocket server

- Bind loopback only by default (`127.0.0.1`; consider also `::1`); expose the all-interfaces option only as an explicit advanced setting, and if enabled still require the challenge.
- Reject any upgrade with a non-empty `Origin` (blocks browser/CEF/prim-media pages); also consider requiring `Sec-WebSocket-Protocol: sl-ide-1.0` or similar as a second cheap filter.
- Challenge: random 128-bit secret in a `create_new`, user-only file in the temp directory; path (not value) sent in `session.handshake`; delete after use; 30 s deadline; no method dispatch (not even discovery) before success; answers to the server's own calls are the only inbound traffic processed pre-auth. The threat model is "another local user or a web page reaches the port"; a same-user local process is trusted by design.
- Do not log challenge values; log paths only at debug level.
- Limit frame size (tungstenite `max_message_size`, e.g. 4 MiB; scripts are at most 64 KiB and notecards similar) and per-connection pending async operations; the C++ code has no such cap.
- Enforce server-side permission checks exactly as the viewer does (copy+modify to read script source; modify on item *and* prim to write; RLVa locks) rather than trusting client state; treat `object.request` on no-modify objects as forbidden.
- Keep `ExternalWebsocketSyncEnable` default **off** in the Rust viewer (the ported settings default it on at `settings.xml:27612`, but the test plan asks for it to start disabled); start the server lazily on user action.
- VS Code launch: pass the URI through `std::process::Command` with separate args (never a shell string), mirroring the care taken for `&` on Windows (`llscripteditorws.cpp:419-432`).

---

## 5. File index (release tree)

| Area | Files |
|---|---|
| Bridge | `indra/newview/llscripteditorws.{h,cpp}`, `llpublishedobjectmgr.{h,cpp}`, `llscripteditorposixfile.h`, `tests/llscripteditorposixfile_test.cpp` |
| Transport/RPC | `indra/llcorehttp/llwebsocketmgr.{h,cpp}`, `lljsonrpcws.{h,cpp}`, `tests/wsserver_test.cpp`, `tests/jsonrpcws_test.cpp`, `tests/testwsclient.h`, `indra/cmake/websocketpp.cmake` |
| Editor | `indra/newview/llpreviewscript.{h,cpp}`, `llscripteditor.{h,cpp}`, `indra/llui/llkeywords.{h,cpp}`, `llfloatergotoline.cpp`, XUI `panel_script_ed.xml`, `floater_live_lsleditor.xml`, `floater_script_preview.xml` |
| Syntax defs | `indra/newview/llsyntaxid.{h,cpp}`, `indra/cmake/LSLDefinitions.cmake`, `autobuild.xml` (`lsl_definitions`, `websocketpp`) |
| Inventory | `indra/llinventory/llinventory.{h,cpp}`, `llinventorytype.h`, `tests/llinventory_runtime_test.cpp`, `indra/newview/llviewerinventory.{h,cpp}`, `llinventoryicon.cpp`, `llinventorybridge.cpp`, `llpanelobjectinventory.cpp`, `textures.xml`, `icons/Inv_Script_Luau.png` |
| Creation / objects | `indra/newview/llviewerobject.{h,cpp}`, `llpanelcontents.{h,cpp}`, `llselectmgr.cpp`, `llviewerregion.cpp`, `floater_tools.xml` |
| Compile | `indra/newview/llviewerassetupload.{h,cpp}`, `llcompilequeue.{h,cpp}`, `llviewermenu.cpp`, `floater_script_queue.xml`, `menu_viewer.xml`, `fslslbridge.cpp` |
| Settings/UI | `indra/newview/llfloaterscripting.{h,cpp}`, `floater_scripting_settings.xml`, `app_settings/settings.xml`, `notifications.xml`, `strings.xml`, `llviewerfloaterreg.cpp`, `llfloaterimnearbychathandler.cpp`, `llappviewer.cpp` |
| Misc | `indra/llcommon/workqueue.{h,cpp}` (`postToMainLoop`), `indra/llmessage/message_prehash.{h,cpp}`, `indra/newview/llfilepicker.cpp`, `llfloaterscriptrecover.cpp`, `luau.txt` (licence), `scripts/tests/test_slua_review.py`, `test_slua_ui_resources.py`, `doc/testplans/slua_editor.md` |
