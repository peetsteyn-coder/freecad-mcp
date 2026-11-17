# Lessons from FreeCAD MCP → design rules for an Inventor MCP

**Source:** this repository (`neka-nat/freecad-mcp` lineage, v0.1.22, 109 commits, ~15 contributors), read directly from source and `git log` on 2026-11-17.
**Verification tags** (`[V:doc]`, `[U]`, …) for Inventor API names follow [04-inventor-api-reference.md](04-inventor-api-reference.md).
**Evidence strength:** *Strong* for "what FreeCAD MCP does" (primary source: the code). *Moderate* for "this lesson transfers to Inventor" — the threading/transport lessons transfer directly because Inventor's API is also single-threaded COM bound to its UI thread; the FreeCAD-specific lessons (Wayland, OCCT) do not.

---

## 1. Architecture in one picture

```
 Claude / any MCP client
        │  stdio (MCP)
        ▼
 ┌──────────────────────────┐      XML-RPC over HTTP        ┌───────────────────────────────────────┐
 │ freecad-mcp (Python)     │ ───────────────────────────▶ │ FreeCAD add-on (runs INSIDE FreeCAD)  │
 │ src/freecad_mcp/server.py│   127.0.0.1:9875, 150 s       │ rpc_server.py  – SimpleXMLRPCServer   │
 │ FastMCP tools            │   socket timeout              │   on a background thread              │
 │ operations/core.py       │                               │ gui_dispatch.py – queue + Qt signal   │
 │ freecad_client.py        │ ◀─────────────────────────── │   marshals each call to the GUI thread │
 └──────────────────────────┘   dict {success, error, …}    │ view_manager.py – screenshots          │
                                 + base64 PNG               │ object_factory / property_mapper       │
                                                            └───────────────────────────────────────┘
```

Two processes, two roles:

| Layer | Runs where | Responsibility |
|---|---|---|
| **MCP server** (`src/freecad_mcp`) | Separate Python process launched by the MCP client (`uvx freecad-mcp`) | Tool schemas & docstrings (the "prompt surface"), argument validation, response shaping (text + optional PNG), connection management, CLI flags |
| **In-app bridge** (`addon/FreeCADMCP`) | Inside FreeCAD's own Python interpreter | Network listener, thread marshalling onto the GUI thread, all calls into the CAD API, serialization of CAD objects to JSON-safe dicts |

**Transfer to Inventor:** keep the same split. The in-app half becomes an **Inventor add-in** (C#/.NET, loaded by Inventor via an `.addin` manifest) instead of a Python workbench, because Inventor does not embed a Python interpreter. The MCP half can stay Python (reusing this repo's patterns) or move to C#. See `02-functional-specification.md` §5 for the trade-off.

## 2. Tool surface (16 tools)

| Category | Tools |
|---|---|
| Documents | `create_document`, `list_documents`, `reload_document` |
| Objects (generic, property-bag) | `create_object`, `edit_object`, `delete_object`, `get_object`, `get_objects` |
| Escape hatch | `execute_code` (GUI thread, 90 s), `execute_code_async` (background thread, no GUI/document access) |
| Visual feedback | `get_view` (+ optional screenshot on every mutating tool) |
| Library | `get_parts_list`, `insert_part_from_library` |
| Analysis | `run_fem_analysis` (CalculiX) |
| Health | `get_rpc_status` (does **not** touch the GUI thread) |
| Prompt | `asset_creation_strategy` (an MCP *prompt*, not a tool) |

Observation: the generic `create_object(obj_type, obj_properties)` works for FreeCAD because FreeCAD objects are property bags (`Part::Box` with `Length/Width/Height`). **Inventor is not like this** — features are built from *definitions* that reference sketches, profiles, faces and edges (`ExtrudeFeatures.CreateExtrudeDefinition(profile, operation)` → `SetDistanceExtent` → `Add`). A generic property-bag tool would not map cleanly, so the Inventor MCP needs **typed feature tools** plus an escape hatch.

## 3. Lessons learned, each tied to the commit that paid for it

| # | Problem observed in FreeCAD MCP | Fix (commit) | Design rule for Inventor MCP |
|---|---|---|---|
| L1 | CAD API must run on the app's main/UI thread; the RPC listener runs on another thread | Queue + Qt-signal wake + 500 ms heartbeat (`gui_dispatch.py`) | Every Inventor API call goes through one **main-thread dispatcher** in the add-in (WinForms `Control.Invoke`/`SynchronizationContext`). No API calls from worker threads. |
| L2 | Client retried after a timeout and the original queued task *still ran* → duplicate geometry | `2cd4fd8` cancel-on-timeout: a queued-but-not-started task is dropped | Dispatcher must mark each request **cancelled on timeout** and never start it late. Make mutating tools **idempotent where possible** (client-supplied `request_id`). |
| L3 | A task that already started can't be interrupted; later calls queued behind it and hung | `DispatchHealth` → `GUI_DISPATCH_STUCK` fail-fast + `get_rpc_status` that never touches the GUI thread | Same: a **health/status tool served off the UI thread**, structured error code `INVENTOR_BUSY`/`DISPATCH_STUCK`, fail fast instead of piling up. |
| L4 | MCP client hung 4+ minutes on a frozen GUI | `_TimeoutTransport` with socket timeout (`freecad_client.py`) | Client-side **timeouts on every call**; per-tool overrides for long operations (exports, interference analysis, Design Automation jobs). |
| L5 | Modal dialogs / popups / mouse-drag interrupted by automation | `process_gui_tasks` defers while `activeModalWidget()` / `activePopupWidget()` / mouse buttons are active | Inventor equivalent: check for active command/modal state before running; set `Application.SilentOperation = True` `[V:doc]` around calls that could raise dialogs (it does not help saves during sketch edit — 04 §3); report `INVENTOR_MODAL_DIALOG` instead of hanging. |
| L6 | `localhost` resolved to `::1` on one side and `127.0.0.1` on the other → connection refused | `0282121` bind to `127.0.0.1` | Bind explicitly to `127.0.0.1` (or a named pipe, which sidesteps this). |
| L7 | App silently renamed objects ("My Doc"→"My_Doc", "Box"→"Box001"); tools echoed the *requested* name; every follow-up call broke | `71b7db2` return the *actual* name | Every create tool returns the **actual** Inventor name *and* a stable handle (reference key / internal name). Never echo input. |
| L8 | Feature "created" but failed to compute; LLM kept building on a broken object | `object_validation.py` checks state after recompute | After every mutation, check feature **health** (Inventor: `PartFeature.HealthStatus` — enum values `[U]`, read from the type library — plus `Application.ErrorManager` text; see 04 §3) and return it. A sick feature is an error, not a success. |
| L9 | Each screenshot was sent **twice** (image block + base64 inside `structuredContent`) — measured ~51k extra tokens per call | `951809d` `structured_output=False` | When returning images, disable FastMCP auto structured output (or return images via a separate content block only). Test token cost explicitly. |
| L10 | Full-window screenshots (~1932×1246) cost ~2.1k tokens and get downscaled by hosts anyway | `d4ed610` cap at 1024 px long edge | Default screenshot ≤ 1024 px long edge; `include_screenshot` and `view` per call; global `--text-only` flag. |
| L11 | Blank screenshots on occluded/minimized windows (macOS) and Wayland | `6882cf1`, `db1b8bd` | Inventor: use `Camera.SaveAsBitmap` `[V:doc]` after `View.Update()` and verify behavior when Inventor is minimized; add an automated test that the PNG is not uniform. |
| L12 | LLM-callable path argument allowed reading any file (`../`, absolute paths) | `a5926dd` realpath + `commonpath` containment | Every path argument (open, save, export, insert component, template) is resolved and checked against an **allow-listed root set** (active project workspace + configured export dir). |
| L13 | Unhandled exception in one task killed the dispatch loop | `e98461a` | Dispatcher catches per task; converts COM `HRESULT`s to structured errors; loop never dies. |
| L14 | Long solver runs exceeded the default socket timeout | per-call proxy with `timeout + 30` (`freecad_client.py`) | Long operations get explicit timeouts or become **async jobs** (`start_*` → `get_job_status`). |
| L15 | Serialization crashed on invalid shapes / non-geometric objects | `bd27057`, `4c52065` | Serializers are defensive: every property read wrapped; return `{"error": …}` per field rather than failing the whole call. Cap output size. |
| L16 | MCP SDK moved `FastMCP` → `MCPServer` in v2 | try/except import; CI matrix over `mcp<2` and `mcp>=2` | Pin and CI-test the MCP SDK major versions you claim to support. |
| L17 | `execute_code` with `exec(code, globals())` = arbitrary code execution as the CAD user | Accepted risk; documented | For Inventor, make the code/iLogic escape hatch **off by default**, gated by a config flag, logged, and wrapped in a transaction so it can be undone. |
| L18 | Remote access needed for some users | IP allow-list + opt-in `0.0.0.0` bind (`ip_filter.py`) | Default **local-only**; if remote is ever enabled, require a token, not just IP filtering (IP filtering has no authentication). |

## 4. What FreeCAD MCP does *not* have (gaps to close in the Inventor design)

- **No undo grouping.** A multi-step tool call can leave a half-built model. → Inventor has `TransactionManager.StartTransaction` `[V:doc]` — wrap each tool call in one transaction so one Ctrl-Z (or an `undo` tool) reverts it.
- **No stable geometry references.** FreeCAD uses `Face1`/`Edge3` names that change after topology edits (the "topological naming problem"). → Inventor has reference keys (`ReferenceKeyManager`, `[V:doc]`) that survive many edits (a split face binds to a collection); expose them as handles — see 04 §8.
- **No units contract.** FreeCAD is mm-native so it rarely bites. → Inventor's API is **cm/radians internally** `[V:doc]`; this is the single most likely source of 10× errors. Every length/angle parameter in the Inventor MCP must carry units or be documented as mm/deg and converted.
- **No authentication** on the local socket (any local process can drive FreeCAD). → Use a per-session token or a named pipe with an ACL.
- **No structured error codes** beyond `GUI_DISPATCH_STUCK`. → Define an error-code enum up front.
- **No tests that touch the real CAD app in CI** (tests are pure-Python with mocks). → Same constraint for Inventor (no Windows+Inventor CI runner available off the shelf); plan a manual/self-hosted integration suite.
