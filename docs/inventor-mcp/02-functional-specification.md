# Functional Specification — Inventor Modeling MCP

| | |
|---|---|
| **Status** | Draft v0.1 for review — 2026-11-17 |
| **Working name** | `inventor-mcp` (an MCP server + Inventor add-in) |
| **Inputs** | [01 Landscape research](01-landscape-research.md) · [03 Tool catalog](03-tool-catalog.md) · [04 Inventor API reference](04-inventor-api-reference.md) · [05 Lessons from FreeCAD MCP](05-lessons-from-freecad-mcp.md) |
| **Conventions** | "shall" = mandatory for the phase listed; "should" = recommended. Requirement IDs are stable; cite them in issues and PRs. Evidence tags (`[V:doc]`, `[U]`, …) are defined in 04. |

---

## 1. Purpose

Let an LLM client (Claude Desktop, Claude Code, Cursor, or any MCP client) **inspect, create and modify Autodesk Inventor parts, assemblies and drawings** through a safe, typed, verifiable tool interface, running against the user's own Inventor session on their own Windows workstation.

## 2. Why build this, given what exists

Research ([01 §1](01-landscape-research.md#1-headline-findings)) found:

1. **Autodesk's official Inventor MCP server** (2027.2+, 46 tools) covers documents, iProperties, parameters, representations, iLogic rules and exports — but has **no sketch/feature/constraint/drawing-view tools**; modeling is only possible by having the LLM write iLogic VB.NET or Python against the raw API. It also requires 2027.2+, sign-in, and allows one client per Inventor.
2. **Community servers** exist (ipt-mcp, cenitu, tylerwarner33) and are useful references or fork bases, but all are single-maintainer, 0.x, and unaudited.
3. Benchmarks show LLMs fail most often on **structure, geometric references and units**, not on syntax ([01 §5](01-landscape-research.md#5-what-the-research-says-about-llms-doing-cad)). An interface that gives the model typed operations, stable references, explicit units and numeric verification attacks those failure modes directly; a raw code tool does not.

**Positioning:** this product is a **modeling-first** MCP. It shall be usable stand-alone on Inventor 2025–2027, and shall **coexist** with Autodesk's server on 2027.2+ (users may connect both; we do not duplicate Autodesk's iLogic-rule-management and event-trigger tools).

> **Gate:** Phase 0 (§13) evaluates Autodesk's server, ipt-mcp and cenitu hands-on against the acceptance task set in §12.3 before committing to build. If one of them meets ≥80% of the Phase 2 acceptance tasks with acceptable security posture, adopt or fork it instead, and use this document as the gap list.

## 3. Scope

### 3.1 In scope (v1)
- Part modeling: sketches, extrude, revolve, hole, fillet, chamfer, shell, rectangular/circular/mirror patterns, work features; parameters; iProperties; material/appearance.
- Model inspection: feature tree, health, topology description with stable handles, measurements, mass properties.
- Assemblies: place components (incl. Content Center members), constraints/joints, grounding, interference, BOM.
- Drawings: sheets, base/projected/section/detail views, retrieved model dimensions, parts list, balloons, title block fields.
- Sheet metal: face, flange, cut, flat pattern, flat-pattern DXF.
- Export: STEP, STL, IGES, SAT, Parasolid, JT, OBJ, PDF, DWG, DXF.
- Visual feedback: bounded screenshot returned as MCP image content.
- Undo: one Inventor transaction per mutating call; batch-as-one-transaction; undo tool.
- Escape hatch: gated C# / iLogic code execution (off by default).
- API documentation lookup against the **installed** Inventor's type library.

### 3.2 Out of scope (v1)
- Cloud/headless operation (Design Automation for Inventor) — possible future backend for batch jobs (04 §1.9).
- Multi-user / server-hosted Inventor ("Inventor as a service") — conflicts with Autodesk's single-user terms (04 §12); requires legal review.
- Stress analysis / Nastran, Frame Generator, Tube & Pipe, Cable & Harness — no public API (04 §1.10).
- Vault check-in/out.
- Inventor LT (API availability unverified).
- Training any ML model on tool outputs (prohibited by Autodesk AUP item 8 — 04 §12).

## 4. Users and use cases

| Persona | Goal |
|---|---|
| **Design engineer** (primary) | Describes a part or change in words/sketch image; reviews and accepts the result in Inventor |
| **Automation engineer** | Scripts repetitive configuration/export/drawing work through an agent; uses batch and escape hatch |
| **CAD admin / IT** | Deploys, configures permissions (read-only, toolsets, code exec), audits logs |

| ID | Use case | Primary tools | Phase |
|---|---|---|---|
| UC-01 | "What is this part? List its features, parameters and mass." | `get_status`, `get_model_tree`, `get_parameters`, `get_mass_properties` | 1 |
| UC-02 | "Change `Length` to 120 mm and show me." | `set_parameters`, `get_view` | 2 |
| UC-03 | "Make a 100×60×10 mm plate with four Ø6.6 through holes 10 mm from the edges." | `new_document`, `create_sketch`, `add_sketch_geometry`, `extrude`, `hole` | 2 |
| UC-04 | "Fillet the four vertical edges R5." (or "fillet the edges I selected") | `describe_geometry`, `fillet` | 2 |
| UC-05 | Model a part from a dimensioned 2D drawing image | sketch + feature tools, `get_view` | 2 |
| UC-06 | "Export STEP AP242 and an STL in mm to the output folder." | `export` | 2 |
| UC-07 | "Place two of these brackets and an M8 bolt from Content Center, mate them." | assembly toolset | 3 |
| UC-08 | "Check for interference and give me the BOM." | `check_interference`, `get_bom` | 3 |
| UC-09 | "Create an A3 drawing with front/top/iso views, dimensions and a parts list." | drawing toolset | 3 |
| UC-10 | "Turn this into sheet metal, add a 90° flange, export the flat pattern DXF." | sheet-metal toolset, `export` | 3 |
| UC-11 | "Set Part Number, Description and material on all parts in this assembly." | `set_properties`, `batch` | 2 |
| UC-12 | Anything the typed tools can't do (long-tail API) | `api_lookup`, `run_code` | 4 |

## 5. Architecture

### 5.1 Components

```
 MCP client (Claude Desktop / Claude Code / Cursor)
        │ stdio (MCP, JSON-RPC)
        ▼
 ┌─────────────────────────────┐   named pipe \\.\pipe\inventor-mcp-<pid>     ┌───────────────────────────────────────────────┐
 │ inventor-mcp server         │   length-prefixed JSON frames                │ InventorMcp add-in (inside Inventor.exe)      │
 │  • tool schemas & docs      │ ───────────────────────────────────────────▶ │  • pipe listener (background thread, no API)  │
 │  • arg validation, units    │   per-session token (discovery file,         │  • STA dispatcher → Inventor main thread      │
 │  • response shaping, images │   user-only ACL)                             │  • command handlers (typed operations)        │
 │  • toolsets / read-only     │ ◀─────────────────────────────────────────── │  • transactions, verification, handles        │
 │  • timeouts, health         │   {ok, result, verify, error}                │  • capability probe (type library)            │
 └─────────────────────────────┘                                              │  • code-exec host + review dialog (opt-in)    │
                                                                               └───────────────────────────────────────────────┘
```

| Component | Responsibility | Mirrors in FreeCAD MCP |
|---|---|---|
| **MCP server** (separate process launched by the client) | MCP protocol, tool schemas/descriptions, argument validation and unit normalization, response shaping (text + image), toolset gating, client-side timeouts, health tool that works when Inventor is busy | `src/freecad_mcp/` |
| **Add-in** (`ApplicationAddInServer`, loaded by Inventor via `.addin` manifest) | Owns **every** Inventor API call, on Inventor's main thread; transactions; verification data; handle management; capability detection; screenshot capture | `addon/FreeCADMCP/rpc_server/` |
| **Contract** (`contract/*.schema.json`) | One JSON Schema per command, shared by both sides; CI fails if server and add-in disagree | — (new; prevents anti-pattern A14) |
| **Mock backend** | Implements the contract without Inventor, for CI on Linux | — (new; FreeCAD MCP tests are pure-Python mocks) |

### 5.2 Key architecture decisions

| # | Decision | Choice | Rationale / evidence |
|---|---|---|---|
| AD-1 | Where Inventor API calls execute | **In-process add-in** | 100k trivial calls: 74.8 s out-of-process vs 0.24 s in-process; out-of-process calls are only served when the UI is idle (Ekins, verified) [V:adn]. Add-in can detect modal dialogs and own transactions/events. Autodesk's own server chose this. |
| AD-2 | Thread marshalling | Hidden WinForms `Control` created in `Activate`; `BeginInvoke` per request; **single queue** | Inventor has no `ExternalEvent` equivalent; pattern proven in ipt-mcp [V:code]; one-queue drain is the strongest cross-host pattern (Strong) |
| AD-3 | Server↔add-in transport | **Named pipe** with per-session random token in a user-ACL'd discovery file; length-prefixed frames | Avoids firewall prompts and the `localhost`→`::1` bug (FreeCAD L6); unframed TCP wedged other servers (A5) |
| AD-4 | MCP server language | **Recommended: Python** (FastMCP, as in this repo) — **alternative: C#** (official C# SDK) | Python reuses this repo's server/response/screenshot code and mock-test approach and lets CI run on Linux; C# gives one toolchain and no Python on CAD workstations. **Open decision D-2.** |
| AD-5 | Add-in builds | One thin shell per .NET generation from shared source: **net8.0-windows** (2025–2026), **net10.0-windows** (2027); optional **net48** (2022–2024) | Inventor 2025 moved add-ins to .NET 8; 2027 recommends .NET 10 [V:doc] |
| AD-6 | Tool surface | **Hybrid**: ~30 typed core tools + opt-in toolsets + gated code tool + docs lookup + one image tool | Practitioner convergence (Moderate); typed tools target the observed LLM failure modes |
| AD-7 | API name/enum source | **Read the installed type library at add-in start**, build capability map | Methods are renamed/hidden between releases (e.g. `ShellFeatures.CreateDefinition` new in 2026 — verified) [V:doc] |
| AD-8 | Units at the boundary | **mm, degrees, kg, seconds** for all numeric inputs/outputs; strings with explicit units also accepted for parameter expressions | Inventor database units are cm/rad [V:doc]; unit drift is a top LLM failure |

### 5.3 Request lifecycle

1. Client calls a tool → server validates args against the contract schema, normalizes units (mm→cm, deg→rad) only where the add-in expects database units, attaches a `request_id`.
2. Server sends the frame on the pipe with a **per-command timeout**.
3. Add-in listener checks token, enqueues; dispatcher **refuses immediately** with `HOST_BUSY` if a modal dialog is open or a previous task is stuck (FreeCAD L3/L5).
4. On the main thread: capture `verify.before` → `StartTransaction(doc, "MCP: <tool> <name>")` → execute handler → `doc.Update()` → check feature health → on failure `Abort()`, on success `End()` → capture `verify.after` → re-bind tracked handles.
5. Add-in returns `{ok, result, verify, handles, warnings}` or `{ok:false, error}`.
6. Server shapes the MCP result: text JSON + optional image block, `isError` on failure, **no duplicated structured content** (FreeCAD L9).
7. If the server-side timeout fires first: return `TIMEOUT_OUTCOME_UNKNOWN` (never auto-retry); the add-in marks the request cancelled so a not-yet-started task never runs late (FreeCAD L2).

## 6. Functional requirements

Priority: **P1** = Phase 1 walking skeleton, **P2** = Phase 2 part-modeling MVP, **P3** = Phase 3 assemblies/drawings/sheet metal, **P4** = Phase 4 hardening/power features.

### 6.1 Connection, session and health

| ID | Requirement | Pri | Acceptance |
|---|---|---|---|
| FR-SES-01 | The server shall discover running Inventor instances via discovery files written by the add-in (`%LOCALAPPDATA%\inventor-mcp\inventor-<year>-<pid>.json`, user-only ACL) and connect to one. | P1 | With one Inventor running, `get_status` reports version, pid and active document within 2 s. |
| FR-SES-02 | With multiple instances, the server shall require an explicit target when ambiguous and expose `list_targets` / `select_target`. | P3 | Two Inventors running → write tools return `AMBIGUOUS_TARGET` until one is selected. |
| FR-SES-03 | Every pipe request shall carry the session token; requests without it shall be rejected and logged. | P1 | Raw pipe write without token → `UNAUTHORIZED`; add-in log entry. |
| FR-SES-04 | `get_status` shall be served **without touching Inventor's main thread** and report dispatcher state (`idle`/`busy`/`stuck`), current operation, elapsed time, modal-dialog presence and title. | P1 | While a 60 s operation runs, `get_status` returns in <500 ms showing `busy` and the operation name. |
| FR-SES-05 | The add-in shall not process requests until `Application.Ready` is true. | P1 | Requests during Inventor start-up return `HOST_STARTING`. |
| FR-SES-06 | The server shall never start Inventor itself unless `--allow-start` is configured. | P1 | Default config + Inventor closed → `INVENTOR_NOT_RUNNING` with setup hint. |
| FR-SES-07 | The add-in shall report a **capability map** (Inventor version, .NET runtime, available members such as `ShellFeatures.CreateDefinition` vs `CreateShellDefinition`, Content Center availability, iLogic availability). | P1 | `get_status(detail=true)` lists capabilities; a tool needing an absent capability returns `API_UNSUPPORTED_IN_VERSION`. |

### 6.2 Documents and files

| ID | Requirement | Pri | Acceptance |
|---|---|---|---|
| FR-DOC-01 | Create part, sheet-metal part, assembly and drawing documents from the default or a named template; return the **actual** document name and a document handle. | P1 | `new_document(type="part")` → handle + name; `GetTemplateFile` empty-string case falls back to `FileOptions.TemplatesPath` or fails with `TEMPLATE_NOT_FOUND`. |
| FR-DOC-02 | Open, save (in place / save-as / save-copy-as) and close documents, using `SilentOperation` around the call and restoring it afterwards. | P1 | Save of a document in sketch edit exits the edit first or returns `IN_EDIT_MODE`; no dialog appears. |
| FR-DOC-03 | Every path argument shall be resolved (`realpath`) and checked for containment in the allow-listed roots: active project workspace, configured output dir, configured extra roots. | P1 | `../../Windows/x.ipt`, absolute paths outside roots, UNC paths not allow-listed → `PATH_NOT_ALLOWED`. |
| FR-DOC-04 | Save shall not silently overwrite existing files unless `overwrite=true`; saving assemblies shall default to `Save2(False)` (document only) to avoid rewriting library/Content Center files. | P2 | Existing target without `overwrite` → `FILE_EXISTS`. |
| FR-DOC-05 | `list_documents` shall return open documents with type, full path, dirty flag, active flag, and handle. | P1 | — |

### 6.3 Inspection and geometric references

| ID | Requirement | Pri | Acceptance |
|---|---|---|---|
| FR-INS-01 | `get_model_tree` shall return sketches, features (type, name, suppressed, **health**), work features, bodies, parameters summary, and (assemblies) occurrences, with handles; output capped (default 32 KB) with paging. | P1 | Feature in error shows `health:"error"` with Inventor's message. |
| FR-INS-02 | `describe_geometry` shall list faces/edges/vertices of a body with **descriptors** — surface/curve type, normal or axis (unit vector), centroid/midpoint (mm), radius (mm), area (mm²) or length (mm), owning feature, adjacent faces — filterable by type, owning feature, normal direction, nearest point, and size rank. | P2 | "The four vertical edges of the plate" can be selected with `filter={type:"line", direction:[0,0,1]}` and returns exactly 4 handles. |
| FR-INS-03 | Entity handles shall be **stable across model changes** where Inventor can re-bind them: reference keys under a per-document key context kept by the add-in; on re-bind, a split entity returns `HANDLE_AMBIGUOUS` with the candidate handles, a deleted one returns `HANDLE_STALE`. | P2 | Handle obtained before adding a fillet still resolves after the fillet if the face survives. |
| FR-INS-04 | `measure` shall return distance/angle/min-distance between two entities, in mm/deg. | P2 | Two parallel faces 10 mm apart → `10.000`. |
| FR-INS-05 | `get_mass_properties` shall return mass (kg), volume (mm³), area (mm²), centre of mass (mm), bounding box (mm) and the material used, forcing an update first. | P1 | Plate 100×60×10 steel → volume 60 000 mm³. |
| FR-INS-06 | `describe_geometry(source="selection")` shall return the user's current selection (`Document.SelectSet`) as handles + descriptors, so users can point at geometry instead of describing it. | P2 | User selects an edge in Inventor; tool returns its handle. |

### 6.4 Sketching

| ID | Requirement | Pri | Acceptance |
|---|---|---|---|
| FR-SKT-01 | Create a planar sketch on an origin plane (XY/XZ/YZ), a named work plane, an offset from either, or a planar face handle. | P2 | Sketch on a face handle lands on that face. |
| FR-SKT-02 | Add sketch geometry in **one call** (list of entities): line, polyline, rectangle (two-point / center), circle, arc (3-point, center-start-end), polygon, slot, point (hole centre), spline; coordinates in mm in **sketch space**; construction flag. | P2 | A 10-entity profile is created by a single call inside one transaction; `Sketch.DeferUpdates` used for batches. |
| FR-SKT-03 | Report sketch → model space mapping (via `SketchToModelSpace`) so the model can reason about where sketch X/Y point in the world. | P2 | Response includes origin and axis vectors of the sketch in model space (mm). |
| FR-SKT-04 | Add geometric and dimensional constraints by entity reference; report remaining degrees of freedom where available. | P3 | Fully constrained rectangle reports DOF 0 (or "unknown" if API doesn't expose it — `[U]`). |
| FR-SKT-05 | `get_sketch` shall report profiles found and whether each is closed, to avoid the bare `E_INVALIDARG` from `Profiles.AddForSolid` on open sketches. | P2 | Open polyline → `PROFILE_NOT_CLOSED` with the gap location. |

### 6.5 Part features

| ID | Requirement | Pri | Acceptance |
|---|---|---|---|
| FR-FEA-01 | `extrude`: profile (sketch or profile index), extent (distance / two distances / symmetric / through-all / to-face), direction, operation (new body, join, cut, intersect), optional taper; distance may be a number (mm) or a parameter expression. | P2 | UC-03 plate volume = 60 000 mm³ ± 0.1%. |
| FR-FEA-02 | `revolve`: profile, axis (sketch line, work axis, edge handle), full or angle, operation. | P2 | Ø20×30 cylinder by revolve has volume 9 424.8 mm³ ± 0.1%. |
| FR-FEA-03 | `hole`: placement by sketch points or face + two edge offsets; type drilled / counterbore / countersink; diameter **or** thread spec validated against the **installed thread table**; depth or through-all; drill point angle. | P2 | Invalid thread designation → `INVALID_ARGUMENT` with `did_you_mean` from the thread table. |
| FR-FEA-04 | `fillet` (constant radius on edge handles) and `chamfer` (distance / distance+angle / two distances). | P2 | UC-04: four R5 fillets on the 10 mm vertical edges remove 4·(5² − π·5²/4)·10 = 214.6 mm³ (task T4). |
| FR-FEA-05 | `shell` with removed-face handles, thickness, direction; uses `CreateDefinition` (2026+) or `CreateShellDefinition` (≤2025) per capability map. | P2 | Same call succeeds on 2025 and 2026. |
| FR-FEA-06 | `pattern`: rectangular, circular, mirror of features or bodies. | P2 | 6× circular pattern of a hole → 6 holes, volume decreases accordingly. |
| FR-FEA-07 | `create_work_feature`: work plane (offset / angle / through entities), work axis, work point. | P2 | — |
| FR-FEA-08 | `edit_feature`: rename, suppress/unsuppress, delete, change a feature's driving parameter. | P2 | Delete reports dependents removed. |
| FR-FEA-09 | Advanced features toolset: sweep, loft, coil, thread (cosmetic), emboss, split, combine. | P3 | — |
| FR-FEA-10 | Every feature tool shall return the created feature's name and handle, its health, and the verification block (§7.3). A feature that computes with errors shall be rolled back and reported as `FEATURE_FAILED` with Inventor's error text (`Application.ErrorManager`). | P2 | Fillet radius too large → `FEATURE_FAILED`, model unchanged (volume identical to before). |

### 6.6 Parameters, properties, materials

| ID | Requirement | Pri | Acceptance |
|---|---|---|---|
| FR-PAR-01 | `get_parameters` / `set_parameters` (batch) for model, user and reference parameters; set by expression with units (`"120 mm"`, `"Length/2"`); create user parameters; result includes evaluated value in mm/deg. | P1 (read) / P2 (write) | Locale with decimal comma still accepts `"12.5 mm"` (via `GetLocaleCorrectedExpression`). |
| FR-PAR-02 | `get_properties` / `set_properties` for iProperties using **internal set names (GUIDs)**, including custom properties. | P2 | Setting `Part Number` and a custom `Supplier` property round-trips. |
| FR-PAR-03 | Set material and appearance by name, searching asset libraries; return `did_you_mean` on misses. | P2 | `"Steel, Mild"` resolves; `"steel"` returns suggestions. |

### 6.7 Assemblies (toolset `assembly`)

| ID | Requirement | Pri |
|---|---|---|
| FR-ASM-01 | Place a component from an allow-listed path with a position/rotation (mm/deg); optionally ground it. Unsaved new parts are saved first (Add requires a file on disk). | P3 |
| FR-ASM-02 | Place a Content Center member by family path + row/size (e.g. ISO 4762 M8×25) via `ContentFamily.CreateMember`, with cached family navigation. | P3 |
| FR-ASM-03 | Add constraints (mate, flush, angle, insert, tangent) and joints (rigid, rotational, slider, cylindrical, planar, ball) between entity handles; the add-in creates **geometry proxies** automatically. Prefer origin planes/axes when the model names them. | P3 |
| FR-ASM-04 | Move/rotate occurrences (`TransformOccurrences`), set grounded/visible/suppressed, delete. | P3 |
| FR-ASM-05 | `check_interference` across all or selected occurrences; report pairs and interference volume (mm³). | P3 |
| FR-ASM-06 | `get_bom` (structured / parts-only), enabling the view if needed and selecting views by type, not localized name. | P3 |

### 6.8 Drawings (toolset `drawing`)

| ID | Requirement | Pri |
|---|---|---|
| FR-DRW-01 | Create a drawing from template; add sheets with standard or custom size; fill title block prompted fields. `SilentOperation` on to avoid the prompted-text dialog. | P3 |
| FR-DRW-02 | Add a base view (orientation, style, scale, flat-pattern option) and projected views placed relative to it. | P3 |
| FR-DRW-03 | Add section (from a line in view-sheet coordinates) and detail views (circular/rectangular fence). Activate the sheet first. | P3 |
| FR-DRW-04 | Retrieve model dimensions into a view using the version-appropriate API (`GeneralDimensions.Retrieve` ≤2024, `Sheet.RetrieveAnnotations` 2025, `RetrieveAnnotations2` 2026.1+). | P3 |
| FR-DRW-05 | Add a parts list and auto-balloons for assembly views. | P3 |
| FR-DRW-06 | Capture a sheet image (`get_view` on a drawing). | P3 |

### 6.9 Sheet metal (toolset `sheet_metal`)

| ID | Requirement | Pri |
|---|---|---|
| FR-SHM-01 | Convert/create sheet-metal parts (subtype GUID), set thickness/style by **activating** a style (not assigning `ActiveSheetMetalStyle`). | P3 |
| FR-SHM-02 | Face, flange (angle, height via `SetDistanceHeightExtent`), cut (sketch on sheet face), corner round. Version-specific `CreateDefinition` vs `CreateFlangeDefinition`. | P3 |
| FR-SHM-03 | Create/refresh flat pattern; report flat extents (mm) and bend list; exit flat-pattern edit afterwards. | P3 |

### 6.10 Export and import

| ID | Requirement | Pri | Acceptance |
|---|---|---|---|
| FR-EXP-01 | `export` to STEP (AP203/214/242), IGES, SAT, Parasolid, JT, STL, OBJ, PDF, DWG, DXF via translator add-ins by GUID; output path allow-listed; options passed as typed fields, not raw option maps. | P2 | STL with `units="mm"` sets `ExportUnits=5` (not 4 — verified trap); STEP header `FILE_SCHEMA` matches the requested AP (check in acceptance harness — Autodesk pages disagree on AP numbering). |
| FR-EXP-02 | Flat-pattern DXF/DWG via `DataIO.WriteDataToFile("FLAT PATTERN DXF?...")` with layer options. | P3 | DXF opens; outer profile on configured layer. |
| FR-EXP-03 | Import STEP/IGES/SAT as a new document (allow-listed path); report created files. | P3 | — |
| FR-EXP-04 | Every export shall return the absolute output path, file size and a freshness check (mtime after call start). | P2 | — |

### 6.11 Visual feedback

| ID | Requirement | Pri | Acceptance |
|---|---|---|---|
| FR-VIS-01 | `get_view` shall render the active document via `Camera.SaveAsBitmap` to a temp PNG and return it as an MCP **image content block**; default long edge **768 px**, clamp 256–1568. | P1 | A 768×768 image costs ≈790 tokens (Anthropic's w×h/750 approximation); image sent only as an image block, never as base64 text. |
| FR-VIS-02 | Orientation presets (iso/front/top/right/back/left/bottom), fit-all or fit-to-entities, optional face/edge **labels** matching `describe_geometry` handles. | P2 (labels P3) | Labelled view shows `F3` on the face reported as `F3`. |
| FR-VIS-03 | Restore the user's camera after capture. | P1 | Camera eye/target unchanged after call. |
| FR-VIS-04 | Mutating tools shall **not** return images by default (numeric verification first); `include_view=true` opts in. A global `--no-images` flag shall override. | P2 | — |
| FR-VIS-05 | Detect a blank/uniform image (e.g. minimized window) and return `VIEW_UNAVAILABLE` rather than a useless picture. | P2 | Minimized Inventor → error, not a black PNG. |

### 6.12 Transactions, undo and batch

| ID | Requirement | Pri | Acceptance |
|---|---|---|---|
| FR-TRX-01 | Each mutating tool call shall run in exactly one Inventor transaction named `MCP: <tool> <target>`; on exception or failed health, `Abort()`. | P2 | One Ctrl-Z in Inventor reverts one tool call. |
| FR-TRX-02 | `batch` shall run up to N (default 25) operations in one transaction; stop at first failure and abort the whole batch; return per-op results. Code execution and rule execution are not allowed inside a batch. | P2 | A 5-op batch with op 4 failing leaves the model unchanged. |
| FR-TRX-03 | `undo` shall undo the last N MCP transactions (only MCP-named ones unless `include_user=true`). | P2 | — |
| FR-TRX-04 | The add-in shall subscribe to `TransactionEvents`/`ApplicationEvents` and tell the client (in the next response's `warnings`) when the user changed the model or active document between calls; cached handles are re-validated. | P3 | User deletes a feature manually → next call warns and affected handles return `HANDLE_STALE`. |

### 6.13 Escape hatch and documentation (toolset `code`, off by default)

| ID | Requirement | Pri | Acceptance |
|---|---|---|---|
| FR-COD-01 | `run_code` shall execute C# (Roslyn script) or iLogic VB snippet in the add-in **only if** enabled by the user in config **and** (default) approved in an **in-Inventor review dialog showing the code**. | P4 | Disabled → `CODE_EXEC_DISABLED`; enabled → dialog appears; reject → `USER_REJECTED`. |
| FR-COD-02 | Code runs inside a transaction (abort on exception), with output capped (60 KB) and a timeout; refused on documents with unsaved changes unless `allow_unsaved=true`. | P4 | — |
| FR-COD-03 | Every executed snippet shall be appended to an audit log with timestamp, SHA-256, document, and outcome. | P4 | — |
| FR-COD-04 | Generated iLogic shall never be stored as a rule in the user's document unless explicitly requested (`AddRule` runs and persists — 04 §10). | P4 | — |
| FR-DOCS-01 | `api_lookup(query)` shall search the **installed** type library (objects, members, signatures, enum values) and return matches with a "hidden/deprecated" flag. | P2 | `api_lookup("ShellFeatures")` on 2026 shows `CreateDefinition` and flags `CreateShellDefinition` hidden. |
| FR-DOCS-02 | The server shall ship MCP **prompts** (`modeling_workflow`, `drawing_workflow`) and resources (`inventor://capabilities`, `inventor://thread-table`) with modeling best practices (units, sketch-then-feature, verify-after-every-step, prefer editing parameters), free of hidden directives. | P2 | — |

### 6.14 Configuration and toolsets

| ID | Requirement | Pri |
|---|---|---|
| FR-CFG-01 | Toolsets: `core` (default), `assembly`, `drawing`, `sheet_metal`, `advanced_features`, `code`. Disabled toolsets are **absent from `tools/list`** (not hidden behind a dispatcher). | P2 |
| FR-CFG-02 | `--read-only` exposes only read-only tools. | P1 |
| FR-CFG-03 | Tool annotations: `readOnlyHint`, `destructiveHint`, `idempotentHint` set accurately on every tool. | P1 |
| FR-CFG-04 | Settings via CLI flags / env vars for the server and a JSON settings file for the add-in (allowed roots, code-exec enablement, review dialog, max image size, timeouts). The add-in settings are authoritative for security-relevant options (the server cannot enable code exec on its own). | P2 |

## 7. Data contracts

### 7.1 Units policy
- Inputs/outputs: **mm, deg, kg, mm², mm³, s**. Field names carry the unit where ambiguous (`radius_mm`, `angle_deg`).
- Parameter **expressions** are passed through verbatim (`"120 mm"`, `"d0 * 2"`) and evaluated by Inventor; responses add `value_mm`/`value_deg`.
- The add-in converts mm→cm and deg→rad at the API boundary (one place, unit-tested).
- Locale: expressions are normalized with `UnitsOfMeasure.GetLocaleCorrectedExpression`.

### 7.2 Handles
Opaque but readable strings, e.g. `doc:Bracket.ipt`, `feat:Extrusion1`, `sk:Sketch2`, `face:F7@k3a9…`, `edge:E12@k3a9…`, `occ:Bolt:1`.
- Feature/sketch/occurrence handles use Inventor names (always the **actual** name returned by Inventor — FreeCAD L7).
- Face/edge/vertex handles wrap a reference key bound in the add-in's per-document key context; the short label (`F7`) matches image labels.
- Handles are document-scoped; using one with another document → `INVALID_ARGUMENT`.

### 7.3 Response envelope (mutating tools)
```json
{
  "ok": true,
  "result": { "feature": "feat:Extrusion1", "name": "Extrusion1", "health": "ok" },
  "verify": {
    "volume_mm3": { "before": 0, "after": 60000.0 },
    "bodies": { "before": 0, "after": 1 },
    "bbox_mm": { "min": [0, 0, 0], "max": [100, 60, 10] },
    "faces": { "before": 0, "after": 6 }
  },
  "undo": { "transaction": "MCP: extrude Extrusion1" },
  "warnings": []
}
```

### 7.4 Error envelope
MCP result with `isError: true` and:
```json
{ "ok": false, "error": { "code": "FEATURE_FAILED", "message": "…Inventor error text…",
  "hint": "Radius 12 mm exceeds the 10 mm plate thickness; try ≤ 4.9 mm.",
  "retry_safe": false, "did_you_mean": [] } }
```

| Code | Meaning | `retry_safe` |
|---|---|---|
| `INVENTOR_NOT_RUNNING`, `HOST_STARTING` | No ready Inventor | yes (after user action) |
| `UNAUTHORIZED` | Bad/missing pipe token | no |
| `HOST_BUSY` | Not started: modal dialog / in-progress command / dispatcher busy (includes dialog title) | **yes** |
| `IN_EDIT_MODE` | Sketch / flat pattern / in-place edit active | yes after exit |
| `TIMEOUT_OUTCOME_UNKNOWN` | Started; result unknown — inspect before retrying | **no** |
| `DISPATCH_STUCK` | A previous task exceeded timeout and is still running | no |
| `INVALID_ARGUMENT` | Schema/semantic error; `did_you_mean` where possible | after fix |
| `PATH_NOT_ALLOWED`, `FILE_EXISTS`, `TEMPLATE_NOT_FOUND` | File policy | after fix |
| `ENTITY_NOT_FOUND`, `HANDLE_STALE`, `HANDLE_AMBIGUOUS` | Reference problems (ambiguous returns candidates) | after fix |
| `PROFILE_NOT_CLOSED` | Sketch has no closed profile | after fix |
| `FEATURE_FAILED` | Feature computed with errors; rolled back | after fix |
| `API_UNSUPPORTED_IN_VERSION` | Capability missing in this Inventor | no |
| `VIEW_UNAVAILABLE` | Screenshot impossible/blank | yes |
| `CODE_EXEC_DISABLED`, `USER_REJECTED` | Escape-hatch policy | no |
| `COM_ERROR` | Unmapped HRESULT; includes HRESULT name and `ErrorManager` text | no |

## 8. Non-functional requirements

| ID | Category | Requirement |
|---|---|---|
| NFR-PERF-01 | Latency | Read tools on a 200-feature part: p95 < 1 s (excluding Inventor compute). Status tool: p95 < 200 ms. |
| NFR-PERF-02 | Bulk | Sketch batches use `DeferUpdates`; assembly batches use `AssemblyOptions.DeferUpdate` (Ekins: 699 s → 54 s for 500 constrained occurrences) [V:adn]. |
| NFR-REL-01 | No hangs | No tool call blocks the client longer than its timeout (default 60 s; export/interference 300 s; configurable). |
| NFR-REL-02 | No late execution | A request that timed out before starting never executes (FreeCAD L2). |
| NFR-REL-03 | Crash containment | Exceptions in handlers never kill the dispatcher; COM objects are re-resolved per call (no cached B-Rep objects across calls). |
| NFR-REL-04 | User coexistence | Automation does not run while the user is mid-command or dragging; `UserInteractionDisabled` (on `UserInterfaceManager`) is always reset in `finally` if used. |
| NFR-TOK-01 | Context economy | Default tool list ≤ 32 tools (the `core` toolset in 03 has 31); each tool description ≤ 1 200 chars; responses capped (default 32 KB text) with paging; images opt-in. |
| NFR-SEC-01 | Local only | Named pipe only; no TCP listener in v1. Pipe ACL = current user. |
| NFR-SEC-02 | Least privilege | Code execution off by default; read-only mode available; security settings live in the add-in config, not settable by the client. |
| NFR-SEC-03 | Untrusted content | Names, iProperties, file names and rule text from models are returned as **data fields**, never interpolated into tool descriptions or instructions (prompt-injection hygiene). |
| NFR-PRIV-01 | Privacy | No telemetry by default. Logs stay local. Tool outputs are never used for model training (Autodesk AUP). |
| NFR-COMP-01 | Compatibility | Inventor 2025, 2026, 2027 (P2); 2022–2024 optional (net48 shell). Windows 10/11 x64. MCP SDK major pinned and CI-tested (FreeCAD L16). |
| NFR-OBS-01 | Observability | Structured JSONL logs in both components (request id, tool, duration, outcome, HRESULT), rotation, secrets/paths masked in client-facing errors. |
| NFR-I18N-01 | Localization | Never look up objects by English display name (origin planes, BOM views, property sets): use indices, internal names or GUIDs. |
| NFR-MAINT-01 | Contract | JSON Schemas are the single source; CI checks server and add-in against them. |

## 9. Security and threat model

| Threat | Vector | Mitigation (requirement) |
|---|---|---|
| Local process drives Inventor | Another process connects to the pipe | Token + user-only ACL (FR-SES-03, NFR-SEC-01) |
| Remote attacker | Network listener | No TCP listener in v1 (NFR-SEC-01) |
| Arbitrary code via LLM | `run_code` | Off by default, host-side review dialog, audit log, transaction (FR-COD-01..03) |
| File exfiltration / overwrite | Path args on open/save/export/place | Allow-listed roots + `realpath` containment + no-overwrite default (FR-DOC-03/04) |
| Prompt injection via model data | Malicious part names / iProperties / iLogic rule text | Return as data; never in descriptions; code exec requires human review (NFR-SEC-03) |
| Persistence via iLogic | Generated rule saved into a document that runs on open | Never `AddRule` unless asked (FR-COD-04) |
| Tool poisoning | Hidden directives in descriptions | Descriptions reviewed in PR; lint for imperative "always/never tell the user" phrases |
| Supply chain | Unpinned MCP SDK / NuGet | Pinned versions, lock files, dependency audit in CI |
| Licence/terms | Multi-user hosting; training on outputs | Local single-user only; no training (§3.2) — **legal review pending** |

## 10. Configuration surface (initial)

| Setting | Where | Default |
|---|---|---|
| `toolsets` | server `--toolsets core,assembly` | `core` |
| `read_only` | server `--read-only` | false |
| `images` | server `--no-images` | images allowed (opt-in per call) |
| `max_image_edge` | server | 768 |
| `timeouts` | server | 60 s default; per-tool overrides |
| `allowed_roots` | add-in settings | active project workspace + `%USERPROFILE%\Documents\inventor-mcp-output` |
| `code_exec.enabled` / `.review_dialog` | add-in settings | false / true |
| `log_dir` | add-in settings | `%LOCALAPPDATA%\inventor-mcp\logs` |

## 11. Packaging and installation

- **Add-in:** signed DLLs + `.addin` manifest per supported year under `%APPDATA%\Autodesk\Inventor 20xx\Addins\` (per-user) or `%PROGRAMDATA%\Autodesk\Inventor 20xx\Addins\` (machine). Registry-free (required since 2024). Installer with `-WhatIf` preview; never `ExecutionPolicy Bypass` one-liners.
- **Server:** if Python — PyPI package runnable with `uvx inventor-mcp` (absolute path documented for GUI clients); optionally a PyInstaller `.exe`. If C# — self-contained `.exe`.
- **Client config:** documented snippets for Claude Desktop, Claude Code (`claude mcp add inventor -- …`), Cursor; installer can back up and patch config.
- **Do not redistribute** `Autodesk.Inventor.Interop.dll`; build against the SDK on the build machine (redistribution permission unverified).

## 12. Verification and acceptance

### 12.1 Test layers
| Layer | Runs on | What |
|---|---|---|
| Unit (server) | Linux CI | Schemas, unit conversion, path policy, response shaping, error mapping — against the mock backend |
| Contract | Linux CI | Server and add-in schemas identical; every tool has ≥1 example payload validated |
| Unit (add-in) | Windows CI (no Inventor) | Pure logic: unit conversion, handle encoding, dispatcher cancel/stuck semantics (port of FreeCAD's `test_gui_dispatch.py`) |
| Integration (live) | Self-hosted Windows + licensed Inventor, per supported year | Each tool against disposable fixtures; type-library dump diffed against 04 |
| Agent evaluation | Same machine, real MCP client | Task suite §12.3, ≥3 runs per task, success rate and token cost tracked per release |

### 12.2 Phase 0 verification harness (closes `[U]` items in 04)
1. Dump type library (interfaces, members, hidden flags, enum values) for each installed Inventor.
2. Verify: reference-key round-trip and split behavior; `View`/`Camera.SaveAsBitmap` when minimized/occluded; `HealthStatusEnum` values; `DisplayModeEnum`; PDF 2D export option names; whether `ControlDefinition.Execute` of modeling commands blocks; STEP AP numbering via exported headers.

### 12.3 Acceptance task suite (numeric oracles)
| # | Prompt (abridged) | Oracle |
|---|---|---|
| T1 | Plate 100×60×10 mm | volume 60 000 mm³; bbox 100×60×10 |
| T2 | T1 + 4× Ø6.6 through holes, 10 mm from edges | volume 58 631.5 mm³ (±0.1%); 4 cylindrical faces Ø6.6 |
| T3 | Cylinder Ø20×30 by revolve | volume 9 424.8 mm³ |
| T4 | T2 + R5 fillets on the 4 vertical edges | volume decreases by 4·(5² − π·5²/4)·10 = 214.6 mm³ → 58 416.9 mm³ |
| T5 | Shell a 50×50×50 box, 2 mm wall, top face open | volume 50³ − 46·46·48 = 23 432 mm³ |
| T6 | Change parameter `d0` (length) to 120 mm | bbox X = 120 |
| T7 | Export T2 as STEP AP242 and STL (mm) | files exist; STEP header schema AP242; STL extents 100×60×10 |
| T8 | Set Part Number, Description, material "Steel" | iProperties round-trip; mass = volume × density |
| T9 | Assembly: two T2 plates stacked & flush, 4 M6 bolts (Content Center) inserted | 0 interferences; BOM qty plates 2, bolts 4 |
| T10 | Drawing of T2: front/top/iso, A3, parts list n/a, retrieved dims | sheet has 3 views; ≥4 dimensions |
| T11 | Sheet metal L-bracket 2 mm, flange 90° × 30 mm; flat DXF | flat extents within 0.5 mm of hand-computed (with K-factor from style) |
| T12 | Model a part from a dimensioned drawing image (provided fixture) | volume and bbox within 1% of reference |

Release gate: Phase 2 tools pass T1–T8 in ≥ 90% of runs; Phase 3 adds T9–T11; T12 is tracked, not gated (research shows this is hard — 01 §5).

## 13. Delivery phases

| Phase | Content | Exit criteria |
|---|---|---|
| **0 — Evaluate & de-risk** | On a Windows machine with Inventor: run Autodesk's server, ipt-mcp and cenitu against T1–T8; run the verification harness (§12.2); legal review of terms (single-user local use, add-in distribution, interop DLLs). | Build/fork/adopt decision recorded; `[U]` items resolved or re-scoped. |
| **1 — Walking skeleton** | Add-in shell (2026 or 2027 first), pipe + token + discovery, STA dispatcher with cancel/stuck semantics, `get_status`, documents, `get_model_tree`, `get_parameters`, `get_mass_properties`, `get_view`; server with mock backend; CI. | UC-01 works end-to-end; dispatcher tests green; no hang under modal dialog. |
| **2 — Part-modeling MVP** | Sketch, extrude, revolve, hole, fillet, chamfer, shell, pattern, work features, edit feature, parameters/properties/material, `describe_geometry` + stable handles, verification block, transactions, batch, undo, export, `api_lookup`, guidance prompt. Second Inventor year. | T1–T8 ≥ 90%. |
| **3 — Assemblies, drawings, sheet metal** | Toolsets `assembly`, `drawing`, `sheet_metal`, `advanced_features`; image labels; change notifications. | T9–T11 ≥ 80%. |
| **4 — Hardening & power features** | `run_code` with review dialog + audit; installer, signing, docs; optional net48 shell; security review; performance pass. | Security review closed; installer tested on clean machines. |

## 14. Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Autodesk adds modeling tools to its official server, making ours redundant | Medium | High | Phase 0 gate; keep scope modeling-first; architecture lets us retire overlapping tools |
| API names drift per release | High (observed) | Medium | Capability map from type library (AD-7); per-year integration suite |
| LLM picks wrong geometry | High (benchmarks) | High | Descriptors + stable handles + labelled images + `get_selection`; numeric verification |
| Modal dialogs hang automation | High (all hosts) | High | Dialog detection, `SilentOperation`, `HOST_BUSY`, never auto-retry |
| Per-version add-in builds/maintenance | Medium | Medium | Shared source, thin shells, CI matrix |
| Terms-of-use / redistribution issues | Unknown | High | Legal review in Phase 0; no interop redistribution; local single-user |
| Bus factor (like every community project) | Medium | Medium | Contract-first design, tests, docs; avoid unique tribal knowledge |
| Security incident via code exec | Low–Medium | High | Off by default, review dialog, audit, transaction |

## 15. Open decisions for the product owner

| ID | Decision | Options | Recommendation |
|---|---|---|---|
| D-1 | Build vs fork vs adopt | (a) build per this spec; (b) fork ipt-mcp (Apache-2.0) and trim; (c) adopt Autodesk + cenitu | Decide after Phase 0 with data |
| D-2 | MCP server language | Python (reuse this repo) / C# (single toolchain) | Python unless CAD workstations can't have Python/uv |
| D-3 | Inventor versions to support | 2025–2027 / also 2022–2024 | Match what your users run; each extra .NET generation adds a build |
| D-4 | Code-exec escape hatch in v1? | none / C# / iLogic / both | C# + iLogic, off by default, Phase 4 |
| D-5 | Distribution | internal only / public OSS / commercial | Internal first (terms: "internal business use") until legal review |
| D-6 | Coexistence with Autodesk server on 2027.2+ | duplicate doc/param tools / defer to Autodesk | Keep minimal doc/param tools for bootstrap on all versions; defer iLogic-rule management to Autodesk |

## 16. Traceability to research

| Requirement(s) | Source |
|---|---|
| AD-1, NFR-PERF-02 | Ekins performance post (verified numbers), 04 §1.7 |
| AD-2, FR-SES-04, NFR-REL-01/02 | FreeCAD MCP L1–L4 (05); ipt-mcp STA dispatcher; pattern P2/P3 (01 §4.2) |
| AD-3, FR-SES-03, NFR-SEC-01 | FreeCAD L6/L18; bimwright token design; anti-pattern A1/A2 |
| AD-7, FR-SES-07, FR-FEA-05, FR-DRW-04 | 04 §0.3 version drift (verified `ShellFeatures.CreateDefinition` 2026) |
| AD-8, §7.1, FR-EXP-01 | 04 §3 units; STL `ExportUnits` 4=cm/5=mm (verified); anti-pattern A11 |
| FR-INS-02/03, FR-VIS-02 | 04 §4.11 reference keys; HarrierPigeon labels; DIP-CAD/GERD research (01 §4.2 P7) |
| FR-FEA-10, §7.3 | FreeCAD L8; faust-machines verification deltas (P8); CADPrompt structural-error finding |
| FR-TRX-01..03 | 04 §2 TransactionManager; bimwright `TransactionGroup` pattern; FreeCAD gap §4 |
| FR-VIS-01..05 | FreeCAD L9–L11; blender-mcp `look` sizing; CADTests/TraceCAD mixed evidence (01 §5) |
| FR-DOC-03 | FreeCAD L12 (commit a5926dd); Blender SSRF CVE |
| FR-COD-01..04 | tylerwarner33 audit log; spkane/revit-python review dialog; 04 §10 iLogic `AddRule` |
| FR-CFG-01 | KiCad dispatcher rollback (A7); tool-count evidence (A8) |
