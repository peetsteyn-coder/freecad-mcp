# Inventor MCP — landscape research

**Research date:** 2026-11-17 · **Method:** web search (standard + extended, with deliberately disconfirming queries), GitHub search, cloning and reading source of ~60 repositories (nothing executed), direct download of Autodesk help pages, MCP Registry API queries. Full working notes with every URL and a credibility note per source are in [`research/`](research/).

**Limits that cap confidence:** no live Inventor was available, so *no server was run*; every "works on Inventor 20xx" statement below is a vendor or author claim. `forums.autodesk.com` and `www.autodesk.com` blog pages returned HTTP 403, so community sentiment and some Autodesk marketing were seen only as search snippets.

---

## 1. Headline findings

| # | Finding | Evidence strength |
|---|---|---|
| F1 | **Autodesk now ships an official Inventor MCP server** (Inventor **2027.2+**, released ≈2026-11-03), hosted by the *Autodesk Assistant* add-in, reached by `InvMcpStdio.exe` (stdio) → named pipes. **46 tools.** | **Strong** that it exists and is documented (Autodesk help pages fetched directly and re-verified independently). **Insufficient** for production readiness (≈2 weeks old, no independent field reports). |
| F2 | The official server is a **document/data/automation server, not a modeling server**: no sketch, feature, constraint or drawing-view tools. Geometry can only be created through `execute_ilogic_code` (VB.NET) or `execute_python_script`. | **Strong** (Autodesk tool reference, verified twice). |
| F3 | **≥25 community/third-party Inventor MCP projects** exist, all from 2026. Three are substantive and permissively licensed: **bimwright/ipt-mcp** (Apache-2.0, C#, 150 tools, Inventor 2022–2027), **cenitu/inventor-mcp-server** (MIT, Python COM, 65 tools), **tylerwarner33/autodesk-inventor-mcp** (MIT, C#, 35 tools). All are effectively **single-maintainer**. | **Moderate** (source read; tool counts re-counted by hand; no live test). |
| F4 | No option is **production-ready** in the sense of GA status + multiple maintainers + independent validation + security review. | **Moderate**. |
| F5 | Across all CAD hosts, mature MCP servers converge on: thin in-app add-in + external stdio server; one main-thread queue; hybrid tool surface (small typed core + gated code + observation + docs); verification data on every mutation; one undo step per call. | **Strong** for the threading pattern; **Moderate** for the rest (practitioner convergence, no controlled CAD study). |
| F6 | LLMs are **not reliable for complex parts or assemblies**; they are useful for **simple parts and edits to existing models** when given execution feedback and numeric verification. | **Strong** (negative) / **Moderate** (positive). |

**Bottom line for the build decision:** an Inventor MCP *exists*, so we should not build from zero without first evaluating what's there. But the gap that matters most for an LLM *designing* parts — typed, verifiable modeling operations with stable geometry references and visual/numeric feedback, on the Inventor versions we actually run — is **not covered by Autodesk's server** and is covered only by young single-maintainer community projects. See the recommendation in [`README.md`](README.md) and the specification in [`02-functional-specification.md`](02-functional-specification.md).

---

## 2. Autodesk's official Inventor MCP server

**Primary sources (fetched directly):**
- Overview — https://help.autodesk.com/cloudhelp/ENU/ADSKMCP-InventorMcp/files/ADSKMCP_InventorMcp_autodesk_inventor_mcp_html.html
- Client setup — https://help.autodesk.com/cloudhelp/ENU/ADSKMCP-InventorMcp/files/ADSKMCP_InventorMcp_ide_integration_html.html
- Tool reference — https://help.autodesk.com/cloudhelp/ENU/ADSKMCP-InventorMcp/files/inventormcp_tools.html
- Troubleshooting — https://help.autodesk.com/cloudhelp/ENU/ADSKMCP-InventorMcp/files/ADSKMCP_InventorMcp_troubleshooting_html.html
- Inventor 2027.2 What's New (AI) — https://help.autodesk.com/cloudhelp/2027/ENU/Inventor-WhatsNew/files/INV2027-2-AI-ENHANCEMENTS.htm

### 2.1 Facts (as Autodesk documents them)

| Aspect | Documented behavior |
|---|---|
| Availability | Inventor 2027.2 or later; Autodesk Assistant add-in (bundled) |
| Transport | `AI client ──stdio──▶ InvMcpStdio.exe ──named pipes──▶ Inventor (Autodesk Assistant add-in)`; exe at `C:\Program Files\Autodesk\Inventor 20XX\Bin\AutodeskAssistant\InvMcpStdio.exe /Inventor` |
| Preconditions | Inventor already running ("the proxy never starts Inventor") and **signed in to an Autodesk account**; server stops running tools if you sign out |
| Concurrency | **One external AI client per Inventor instance** |
| Transactions | "Most write tools run inside a single Inventor transaction" (one Undo entry). `execute_ilogic_code` runs **outside** a transaction — partial changes on failure |
| Code execution | `execute_python_script` (Python ≥3.13, 30 s default / 300 s max timeout; module screening that is "not a sandbox") and `execute_ilogic_code`; both run "with your Windows user permissions" |
| Units | Values "returned in the document's own units"; parameter edits take unit-bearing expressions (`"120 mm"`) |
| MCP features | Tools only — **no resources, no prompts**; read-only tools carry read-only annotations |
| Images | `export_document_as_image` writes a **file**; no image content returned to the model |
| Kill switch | Unload the Assistant add-in, or delete its manifest under `%ALLUSERSPROFILE%\Autodesk\Inventor 20XX\Addins\Assistant` |

### 2.2 The 46 tools

| Group | Tools |
|---|---|
| Files & documents (14) | `get_file_information`, `open_file`, `get_files`, `close_documents`, `get_all_open_documents`, `get_document_info`, `save_document`, `get_document_settings`, `get_document_selections`, `find_document_objects`, `update_document`, `export_document_as_pdf`, `export_document_as_image`, `edit_document_settings` |
| Properties (5) | `get_document_property_sets`, `get_document_properties`, `get_occurrence_properties`, `edit_properties`, `delete_properties` |
| Parameters (4) | `get_document_parameters`, `create_user_parameter`, `edit_parameter`, `delete_parameters` |
| Views (1) | `set_view` |
| Components (4) | `place_component`, `get_component_states`, `set_component_state`, `get_referenced_components` |
| Representations (4) | `get_representations`, `get_member_table`, `edit_member_table`, `edit_representation` |
| Projects & materials (2) | `get_project_info`, `find_materials` |
| Scripting guidance (1) | `get_script_documentation` |
| iLogic (8) | `execute_ilogic_code`, `list_ilogic_rules_and_forms`, `read_ilogic_rule`, `run_ilogic_rule`, `create_ilogic_rule`, `edit_ilogic_rule`, `delete_ilogic_rule`, `manage_external_rule_directories` |
| Python (1) | `execute_python_script` |
| Event triggers (2) | `get_event_triggers`, `edit_event_triggers` |

### 2.3 Release stage — unresolved

- The 2027.2 What's New page calls it the "Inventor Public MCP Server" and does **not** use the words beta/preview (verified).
- A search-engine snippet of Autodesk's MCP landing page labels the Inventor card **"Public Beta"**; our own render of that JS page did not show the card. **Treat as beta; not directly verified.**
- It is **not** in the official MCP Registry nor in `github.com/Autodesk/ai-integrations` (which carries only Fusion and Product Help); it ships only inside the Inventor installer.
- Unknowns: whether any tool traffic or telemetry routes via Autodesk cloud; which editions/subscriptions are entitled; roadmap to GA.

### 2.4 Assessment

| Strengths | Weaknesses (for an LLM that must *design*) |
|---|---|
| Vendor-built, in-process (fast, main-thread-correct) | **No modeling primitives** — every sketch/feature must be written as iLogic/Python by the model |
| Transactions on most writes; read-only annotations; admin kill switch | **2027.2+ only** — unusable on 2022–2027.1 |
| Same tools Autodesk dogfoods in Assistant | No image content back to the model; no resources/prompts |
| Tool list served live from the running version | One client per Inventor; sign-in required; proprietary — cannot be extended |

---

## 3. Community and commercial Inventor MCPs

Tool counts were re-counted from source (e.g. 150 `[McpServerTool(Name=…)]` attributes in ipt-mcp; 62 `@tool()` + 3 `@tool(com=False)` registrations in cenitu).

| Project | License | Architecture | Tools | Inventor | Escape hatch | Notable strengths | Notable risks | Evidence |
|---|---|---|---|---|---|---|---|---|
| [bimwright/ipt-mcp](https://github.com/bimwright/ipt-mcp) | Apache-2.0 | C# .NET 8 stdio server + per-year in-process add-in; named pipe (2025–27) / loopback TCP (2022–24); **per-session token** | 150 (39 read-only; 19 toolsets) | 2022–2027 | `send_code` (on by default; can be disabled) | HOST_BUSY vs TIMEOUT error semantics, export-root allowlist, checkpoints, mm/deg at the edge, 122 test files + CI | single maintainer (234 commits, 1 author), 10+ commits/day churn, prebuilt add-ins only for 2025 & 2027 | Moderate |
| [cenitu/inventor-mcp-server](https://github.com/cenitu/inventor-mcp-server) | MIT | Python, pywin32 late-bound COM out-of-process, dedicated COM thread, `SilentOperation` | 65 | self-reported 137/137 smoke tests on 2027.1 | `execute_python` (unrestricted) | volume/topology delta after every feature, modal-dialog detection, transaction tool, `capture_view` image, API field notes; on PyPI + MCP Registry | unrestricted code exec; single maintainer | Moderate |
| [tylerwarner33/autodesk-inventor-mcp](https://github.com/tylerwarner33/autodesk-inventor-mcp) | MIT | C# .NET 10 server (dotnet tool) + add-in, named pipe with owner verification | 35 | 2025–2027 | `inventor_eval_csharp` (Roslyn) | best code-exec safeguards (audit log w/ SHA-256, refuses unsaved docs, read-only tier, MCP elicitation), dialog policy file | vendors Autodesk interop DLLs (redistribution question); inspection-focused, no modeling tools | Moderate |
| [gustacassel/InventorBridge](https://github.com/gustacassel/InventorBridge) | MIT | C# out-of-process COM, Roslyn scripts | 19 | 2024–2027 | `run_csharp` (one undo step, rolled back on error) | generated API reference for 2024–2027 served to the agent; Vault tools | 3 weeks old | Weak–Mod. |
| [hsavas/inventor-mcp](https://github.com/hsavas/inventor-mcp) | **Non-commercial** | Python COM | 109 | unspecified | `inventor_run_script` | broad coverage (sketch, joints, drawings) | **license excludes commercial use** | Weak–Mod. |
| [NeonGlay/inventor-mcp](https://github.com/NeonGlay/inventor-mcp) | MIT | Python COM | 36 | 2026 | `execute_python` | most stars (53) | **archived; breaks on MCP SDK 2.x** | Weak |
| Autodesk official | Proprietary | add-in + stdio proxy | 46 | 2027.2+ | iLogic + Python | see §2 | see §2 | Mod.–Strong (exists) |
| ModernCAD "Inventor AI", AEC Pilot, ForgeMCP ($50/seat/mo), Pierski Tech | Commercial / unverifiable | closed | 57–470 claimed | various | various | — | cannot inspect | Insufficient |

Lesser and stub projects (≈20 more, including license-less and non-MCP-compliant ones) are catalogued in [`research/A_existing_inventor_mcps.md`](research/A_existing_inventor_mcps.md) §2.

**Two architectural camps.** (1) *Out-of-process COM* (cenitu, hsavas, InventorBridge): nothing to install inside Inventor, crash isolation; but cross-process marshalling cost, "first instance only" via the Running Object Table, and calls block behind modal dialogs. (2) *In-process add-in + local IPC* (Autodesk, ipt-mcp, tylerwarner33): speed, UI-thread access, events, transactions; but per-Inventor-version builds and an Inventor restart to update. **Autodesk itself chose camp 2.**

---

## 4. Lessons from comparable CAD MCP servers

Full analysis of ~30 projects (Fusion, Revit, AutoCAD, SolidWorks, Onshape, Rhino, Blender, KiCad, code-CAD, FreeCAD) with ~70 cited sources: [`research/C_comparable_cad_mcps.md`](research/C_comparable_cad_mcps.md).

### 4.1 Most relevant reference designs

| Project | Why it matters for Inventor |
|---|---|
| [bimwright/rvt-mcp](https://github.com/bimwright/rvt-mcp) (Revit, 135★) | Same author as ipt-mcp; Windows .NET single-threaded host; per-session token in ACL'd discovery file; `TransactionGroup.Assimilate()` → one undo step per batch; timeouts return an operation ID and say "do not retry". |
| [HarrierPigeon/Solidworks-MCP-Server](https://github.com/HarrierPigeon/Solidworks-MCP-Server) | Closest COM API analog. Stable IDs (`feat:`, `sketch:`, `entity:`), faces tagged with their creating feature, `find_face` by description, screenshots labelled F0, F1… matching the face list. Documents the silent wrong-face bug from view-dependent picking. |
| [ahujasid/blender-mcp](https://github.com/ahujasid/blender-mcp) (34.7k★) | Cut 41 → 10 tools around `execute_blender_code` + a bounded `look` tool (768 px default). Fixed Windows hangs by draining one main-thread queue. 3 published CVEs (e.g. [CVE-2026-10688](https://osv.dev/vulnerability/CVE-2026-10688), verified). |
| [faust-machines/fusion360-mcp-server](https://github.com/faust-machines/fusion360-mcp-server) (152★) | Cancels timed-out queued work; every mutation returns body-count/mass/bbox before→after; `isError` + `error_kind` + hints; mock mode → 363 tests in CI without Fusion. |
| Autodesk official Revit MCP (39 typed tools, **no code exec**, Read/Write/Experimental installs) vs Autodesk FusionMCPSample (code-first) | Autodesk itself has shipped both extremes. |

### 4.2 Patterns to adopt (P) and anti-patterns to avoid (A)

| Pattern | Strength | Anti-pattern seen in the wild | Strength |
|---|---|---|---|
| P1 Thin add-in + external stdio server | Strong | A1 Listener on all interfaces, no auth, plus code exec (mcp-servers-for-revit: `IPAddress.Any:8080`) | Strong (code read) |
| P2 One queue drained on the main thread | **Strong** | A3 Modal dialogs on the host thread (`TaskDialog.Show` ×23 in one Revit server) | Mod.–Strong |
| P3 Timeouts cancel queued work; never auto-retry a write | Mod.–Strong | A12 Auto-retry after timeout → duplicate features | Moderate |
| P4 Length-framed wire messages | Moderate | A5 Unframed TCP / parse-until-valid buffers | Moderate |
| P5 Hybrid tool surface | Moderate | A7 Hidden-schema `execute_tool` dispatcher (KiCad rolled it back: model hallucinated schemas) | Weak |
| P6 Small visible tool list; toolsets; read-only mode | Moderate | A8 Hundreds of always-on tools (KiCad 257; VS Code caps at 128) | Moderate |
| P7 Stable, human-readable geometry references; select by object, never by screen pick | Moderate | A10 View-dependent coordinate picking → silently wrong face | Moderate |
| P8 Verification data with every mutation | Moderate | A9 Reporting success without checking the effect | Moderate |
| P9 `isError` + actionable hints | Moderate | A6 Hidden directives in tool descriptions ("tool poisoning") | Moderate |
| P10 One undo step per call/batch | Moderate | A11 Ambiguous units (Fusion cm, SolidWorks m internal) | Moderate |
| P11 Units fixed at the boundary, stated in every schema | Moderate | A15 Unvalidated paths/URLs (FreeCAD traversal, Blender SSRF CVE) | Strong |
| P13 Loopback + per-session token + code exec off by default / host-side review | Strong need / Mod. efficacy | A17 Treating AST filters / "safe modes" as sandboxes | Moderate |
| P14 Bounded images (~768 px), at checkpoints, offscreen render, restore camera | Moderate | A16 Screen-grab screenshots that come back black when window not in front | Moderate |
| P16 Mock mode for CI + live-host integration suite + agent eval harness | Moderate | A13 Bootstrap gaps (no tool to create the state other tools require) | Weak |
| P17 Pin MCP SDK major; test new majors (SDK 2.x broke ≥3 servers, incl. NeonGlay) | Moderate | A18 Telemetry on by default | Weak |

---

## 5. What the research says about LLMs doing CAD

**Question:** can a frontier LLM, via tools, reliably produce correct parametric CAD?

| Supporting (it works) | Contradictory (it doesn't) | Uncertain / mixed | Gaps |
|---|---|---|---|
| CADPrompt (ICLR 2025, n=200): GPT-4 code **compiles** 92–96.5% ([arXiv 2410.05340](https://arxiv.org/abs/2410.05340)) | Same paper: **48% of failures are structural configuration errors** — compiling ≠ correct | Visual self-verification: modest gain in CADPrompt; **no gain** in CADTests (48.0% vs 48.5%) ([arXiv 2605.07807](https://arxiv.org/abs/2605.07807)) | Almost all benchmarks are **code-CAD** (CadQuery/FreeCAD); **none** test feature-based tool use in Inventor/SolidWorks/Fusion via MCP |
| BenchCAD **editing** accuracy is high: GPT-5.3 0.865, Claude Opus 4.7 0.853 ([arXiv 2605.10865](https://arxiv.org/abs/2605.10865), Table 3 — verified) | BenchCAD image→code: frontier IoU only **0.19–0.28** (Table 10 — verified); "a render specifies geometry but not numbers" | TraceCAD: removing visual feedback saves ~25% tokens but worsens geometry (IoU 0.364→0.317) | Mostly single runs, no confidence intervals |
| CAD-Assistant (ICCV 2025): tool-augmented planner beats plain VLLM baselines | Zero-to-CAD (Autodesk Research): **22.3%** first-attempt validation; failures: thin walls, self-intersections, misplaced holes, scale drift ([arXiv 2604.24479](https://arxiv.org/abs/2604.24479) §5 — verified) | LLM4CAD: GPT-4V did better text-only than text+image (summary only, not verified) | Unit and reference-selection errors rarely measured as separate categories |
| ReliCAD: 99.8% validity with structured spec + agent API + verification loop (abstract only) | MUSE: assemblies stay poor even when code executes (abstract only) | — | No independent replication of tool-vendor self-reports |

**Interpretation (calibrated):** evidence demonstrates that LLMs are **not** reliable for complex parts or assemblies (**Strong** — five independent benchmarks agree at different scales). Evidence suggests they **are** useful for simple parts and especially for **editing existing models** when execution feedback and numeric verification are provided (**Moderate**). Evidence on screenshots is **mixed**; numeric checks (volume, bounding box, mass, feature health) are cheaper and catch more (**Moderate**, inferred from benchmark failure analyses rather than tested head-to-head).

**Design implications:** (1) invest in verification data returned by tools more than in breadth of "create" tools; (2) give the model stable, describable geometry references; (3) favor edit-oriented workflows (parameters, features on existing models); (4) make units explicit everywhere; (5) images on demand, bounded.

**Typed tools vs code execution** — no controlled CAD-specific comparison exists (**Insufficient**). General-agent evidence moderately favors code actions (CodeAct, [arXiv 2402.01030](https://arxiv.org/abs/2402.01030): up to +20%), but a 2026 ablation found pass rates statistically tied. Practitioner convergence moderately favors a **hybrid** (**Moderate**). Steelman for pure-typed: Autodesk's own Revit server ships no code tool, because a typed surface is auditable, safe to auto-approve, and doesn't require the model to know the API. Steelman for code-first: Inventor's API is huge (thousands of members) and a code tool covers the long tail without us writing hundreds of wrappers — which is exactly why Autodesk's Inventor server leans on iLogic/Python.

---

## 6. Four-column evidence table — "Can we adopt instead of build?"

| Supporting (adopt) | Contradictory (build) | Uncertain / mixed | Gaps / unknowns |
|---|---|---|---|
| Autodesk ships a documented local MCP server in 2027.2 with transactions, read-only annotations, kill switch | Official server has **no modeling tools**; 2027.2+ only; one client per Inventor; sign-in required; not extensible | Release stage (beta?) not directly verified | Cloud/telemetry data flow; edition entitlement; GA roadmap |
| ipt-mcp: Apache-2.0, 150 tools, 2022–2027, token-auth IPC, tests + CI | Every community project is effectively single-maintainer (bus factor 1); 0.x APIs with high churn | Self-reported live test results (cenitu 137/137, ipt-mcp fixtures) not independently reproduced | No security audit of any project; no published agent-task success rates |
| cenitu: MIT, on PyPI + MCP Registry, active, modeling tools + image capture | Licensing blockers on several (non-commercial, no license, GPL/MIT conflict) | Vendored Autodesk interop DLLs in two repos — redistribution permitted? | Whether Autodesk terms restrict third-party add-ins exposing MCP — **not researched; needs legal review** |
| Common proven patterns lower risk of building *or* forking | CI never runs Inventor in any project | Community sentiment under-sampled (Autodesk forums 403) | Performance on large assemblies |

**Verdict:** evidence demonstrates adoptable building blocks exist (**Moderate–Strong**); preliminary evidence suggests none is production-ready (**Moderate**).

---

## 7. Self-audit (bias checklist)

| Check | Result |
|---|---|
| Searched for contradictory evidence? | Yes — "no MCP for Inventor" queries surfaced stale pre-2027.2 claims (now contradicted by Autodesk docs); disconfirming LLM-CAD benchmarks are given equal weight to positive ones. |
| Over-weighted recent/dramatic information? | Risk acknowledged: the official server is 2 weeks old and the headline finding. We deliberately rate its *readiness* Insufficient. |
| Anchoring on early sources? | The first agent report was cross-checked independently (Autodesk tool list re-fetched; tool counts re-counted; 3 paper numbers and 1 CVE checked in the primary text). |
| Framing effects? | Framed as "build vs adopt vs extend" rather than "build", because evidence changed the question. |
| Representative sources? | Skewed toward GitHub and English-language sources; Chinese/Vietnamese/Spanish/German repos were included where found; Autodesk forums under-sampled. |
| Certainty matched to evidence? | Every claim carries a strength label; vendor/competitor conflicts of interest are flagged in the source lists. |
| Hallucination-prone items | Star counts (change daily), tool counts (re-counted), API names (see `04-inventor-api-reference.md` verification marks), paper numbers (3 spot-checked, others marked "abstract only" or "summary only"). |
