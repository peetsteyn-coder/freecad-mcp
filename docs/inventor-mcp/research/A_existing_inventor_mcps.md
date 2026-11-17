> **Appendix — raw research notes.** Produced on 2026-11-17 by a research sub-agent; synthesized and spot-checked in [`../01-landscape-research.md`](../01-landscape-research.md). Local clone paths refer to a temporary research workspace that no longer exists; use the GitHub URLs instead.

# A. Existing Autodesk Inventor MCP servers and Inventor-AI bridges: landscape survey

Survey date: 2026-11-17. Scope: Autodesk **Inventor** (Windows desktop mechanical CAD). Fusion, AutoCAD and Revit are mentioned only where needed for contrast.

---

## 0. Method, access limits and how to read this report

**How evidence was gathered**
- GitHub repository search (`gh api search/repositories`) with 13 query variants (`inventor mcp`, `autodesk inventor mcp`, `inventor-mcp`, `inventor model context protocol`, `ilogic mcp`, `inventor ai agent`, `inventor claude`, `inventor llm`, `inventor copilot`, `inventor chatgpt`, `inventor com api mcp`, `ipt mcp`, `inventor mcp in:readme`). The search endpoint worked. The `repos/{owner}/{repo}` endpoint was blocked in this session, so per-repo metadata (stars, forks, open-issue counts, license, push date) comes from the search API using `repo:` qualifiers.
- Every candidate with real code was **git-cloned (no checkout of binaries, nothing executed)** into a scratch folder. I read the README and source. Tool lists come from scanning the source for `@mcp.tool` / `@tool` decorators (Python) or `[McpServerTool(Name=...)]` attributes (C#), plus each repo's own README. Commit counts, first and last commit dates and author counts come from `git log`.
- Issue and PR counts come from the GitHub issue-search API.
- Official MCP Registry: queried `registry.modelcontextprotocol.io/v0/servers` with `search=inventor|autodesk|ilogic|ipt-mcp|bimwright|aecpilot|...`, paginated.
- Awesome lists: grepped the raw READMEs of punkpeye, appcypher and wong2 `awesome-mcp-servers`, plus `mlightcad/awesome-cad`.
- Autodesk official: downloaded the raw help-topic HTML from `help.autodesk.com/cloudhelp/...`. I located these pages by reading the help site's table of contents (`/view/ADSKMCP/ENU/data/toctree.json` and the Inventor 2027 equivalent).
- I ran web searches (standard and extended), including disconfirming queries.

**Access failures (affect confidence)**
- `forums.autodesk.com` (all threads) and `www.autodesk.com` (blogs, the MCP-servers solution page) returned **HTTP 403** to both WebFetch and curl. Content from those pages comes **only from search-engine snippets** and is flagged as such.
- The Autodesk MCP help *landing page* is JavaScript-rendered. I could not obtain the Inventor card's release-stage label directly (see §3).
- I could not run any server. **No live Inventor testing was done.** All "works with Inventor 20xx" statements are the authors' own claims, CI evidence, or third-party reports.

**Evidence-strength scale used for "works with current Inventor (2027.x)"**
- **Strong**: vendor-shipped or independently reproduced on current Inventor.
- **Moderate**: credible self-reported live testing on current Inventor plus automated tests or CI, and active maintenance.
- **Weak**: claims only, old target version, or very young/abandoned code.
- **Insufficient**: closed source or stub; cannot assess.

---

## 1. Bottom line

1. **Autodesk ships an official Inventor MCP server.** It arrived with Inventor 2027.2, which secondary sources say was released 2026-11-03. Autodesk's own help documentation (fetched directly) describes:
   - a **local stdio MCP server** hosted by the **Autodesk Assistant add-in**;
   - a proxy executable `InvMcpStdio.exe` that talks to Inventor over **named pipes**;
   - **46 tools**, including `execute_ilogic_code`, `execute_python_script` and `run_ilogic_rule`.

   The Inventor 2027.2 What's New page calls it the "Inventor Public MCP Server". Its release stage is reported as **Public Beta** by a search-engine index of the Autodesk MCP help landing page, but I could not see that label directly. It is roughly two weeks old, and **I found no independent hands-on report of it.**
2. **The official server is not a modeling server.** It has **no dedicated sketch or feature-creation tools** (no sketch, extrude, revolve, hole and so on). Geometry creation is possible only through its iLogic and Python code-execution tools. Its strengths are documents, parameters, iProperties, Model States and representations, assemblies (place and state), iLogic rule management, and PDF/image export.
3. **At least 25 community or third-party Inventor MCP projects exist** (all created in 2026, except one 2023 non-MCP precursor). Most are small, single-author and less than six months old. The three most substantial open-source projects are:
   - **bimwright/ipt-mcp**: C#, Apache-2.0, 150 tools, in-process add-in for Inventor 2022–2027, token-authenticated local transport, 122 test files with CI.
   - **cenitu/inventor-mcp-server**: Python/COM, MIT, 65 tools, on PyPI and the official MCP Registry, maintained fork of NeonGlay.
   - **tylerwarner33/autodesk-inventor-mcp**: C#, MIT, 35 tools, add-in plus named pipe, a Roslyn C# escape hatch with an audit log, CI.

   **None has a multi-maintainer community, a stable 1.x API for long, independent live verification, or a security review.** All are effectively single-maintainer projects.
4. **Answer to "Is there a production-ready Inventor MCP we could adopt instead of building?"**
   - **Not production-ready yet, but strong candidates exist.** Preliminary evidence suggests the official Autodesk server (beta, 2027.2+ only) plus one community server is a viable adopt-or-fork baseline.
   - No option is demonstrably production-ready. Evidence strength for that conclusion is **Moderate**: it rests on vendor docs and source reading, without live testing.
   - The decision hinges on the target Inventor versions. If the team must support **2022–2027.1**, Autodesk's server is unavailable, and forking or adopting **ipt-mcp** (permissive, broadest version range, best security posture) or **cenitu** (simplest, Python) is the realistic route.

---

## 2. Summary table of all candidates

Columns:
- **Stars/Forks**: approximate, from the GitHub search API on 2026-11-17.
- **Commits (authors)**: from `git log` on the default branch.
- **Tools**: count I verified in source, with the source file noted in §4.
- **Escape hatch**: arbitrary code execution tool.
- **Evidence**: works on current Inventor; see scale in §0.

### 2.1 Official

| # | Project | Owner | License | Lang / arch | Tools | Escape hatch | Inventor | Status | Evidence |
|---|---|---|---|---|---|---|---|---|---|
| 0 | **Autodesk Inventor MCP Server** (`InvMcpStdio.exe`, ships with the Autodesk Assistant add-in) | Autodesk | Proprietary (part of Inventor) | Add-in hosts server; stdio proxy → named pipes | **46** (from Autodesk tool reference) | `execute_ilogic_code`, `execute_python_script` (module screening, "not a sandbox") | **2027.2+ only** | Public Beta per search index (not directly verified) | **Moderate–Strong** (vendor-shipped and documented; no independent reports yet) |

### 2.2 Open source with substantive code

| # | Project | License | Lang / arch | ★ / forks | First → last commit | Commits (authors) | Tools | Escape hatch | Claimed Inventor | Evidence |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | [bimwright/ipt-mcp](https://github.com/bimwright/ipt-mcp) | Apache-2.0 | C# .NET 8 server + per-year in-process add-in; TCP loopback (2022–24) or named pipe (2025–27), token auth | 19 / 11 | 2026-05-29 → 2026-11-16 | 234 (1) | **150** | `inventor_send_code` (on by default, can be disabled), `ilogic_run_snippet` | 2022–2027 | **Moderate** |
| 2 | [cenitu/inventor-mcp-server](https://github.com/cenitu/inventor-mcp-server) (fork-continuation of NeonGlay) | MIT | Python, pywin32 COM out-of-process, FastMCP/MCPServer, stdio | 7 / 1 | 2026-06-10 → 2026-11-15 | 124 (3, of which 118 by cenitu) | **65** | `execute_python` (unrestricted, persistent namespace) | dev on 2026, "137/137 pass on 2027.1" | **Moderate** |
| 3 | [tylerwarner33/autodesk-inventor-mcp](https://github.com/tylerwarner33/autodesk-inventor-mcp) | MIT | C# .NET 10 server (dotnet tool) + add-in; named pipe per release | 3 / 0 | 2026-09-21 → 2026-11-16 | 165 (1 human + 2 bots) | **35** | `inventor_eval_csharp` (Roslyn), `inventor_run_ilogic`, `inventor_run_plugin` | 2025–2027 | **Moderate** |
| 4 | [gustacassel/InventorBridge](https://github.com/gustacassel/InventorBridge) | MIT | C# .NET 8, out-of-process COM, Roslyn C# scripts | 0 / 0 | 2026-10-28 → 2026-11-16 | 21 (1) | **19** | `run_csharp`, `ilogic_script` | 2024–2027 | Weak–Moderate |
| 5 | [hsavas/inventor-mcp](https://github.com/hsavas/inventor-mcp) | **Non-commercial custom license** | Python pywin32 COM, FastMCP | 9 / 6 | 2026-06-23 → 2026-10-20 | 11 (3) | **109** | `inventor_run_script` (Python; can be disabled by env var) | unspecified; contributor fixed Inventor 2020 bugs | Weak–Moderate |
| 6 | [OC-JG/InventorMCP](https://github.com/OC-JG/InventorMCP) | MIT | Python COM; "recipe"-driven parametric builder plus DFM analyser (Node) | 0 / 0 | 2026-08-17 → 2026-09-28 | 251 (2: 197 "Claude", 54 human) | **31** | `run_inventor_script` | unspecified | Weak–Moderate |
| 7 | [RobertBarnettProjects/inventor-forge](https://github.com/RobertBarnettProjects/inventor-forge) | MIT | Python COM; hand-rolled MCP JSON-RPC (no SDK); UI automation for Stress Analysis | 0 / 0 | 2026-11-07 → 2026-11-12 | 12 (1) | README says 160; ~170 registrations found | `execute_python` | "tested on 2027" | Weak |
| 8 | [Andiveli/mcp-cad](https://github.com/Andiveli/mcp-cad) | **Conflict: LICENSE file is GPL-3.0, README badge says MIT** | C# .NET 8, early-bound COM | 1 / 0 | 2026-05-26 → 2026-06-29 | 111 (1) | **~118** | none found | 2025+ | Weak |
| 9 | [ioseluiz/mcp_server_autodesk_inventor](https://github.com/ioseluiz/mcp_server_autodesk_inventor) | none declared | Python FastMCP server + C# .NET 8 add-in; named pipe; Inno Setup installer | 1 / 1 | 2026-05-14 → 2026-08-07 | 25 (3) | **52** | none found | 2026 | Weak |
| 10 | [wuwenhou/inventor2022-mcp](https://github.com/wuwenhou/inventor2022-mcp) | MIT | C# .NET 10, COM via STA thread (Vietnamese docs) | 0 / 0 | 2026-11-14 | 1 (1) | **77** | none found | **2022 only** | Weak |
| 11 | [oakplank/InventorMCP](https://github.com/oakplank/InventorMCP) | MIT (+Apache notices) | C# server + add-in; current-user authenticated named pipe | 1 / 2 | 2026-07-15 | 5 (1) | **14** (mostly read-only) | **explicitly none** | live-tested 2024.3 | Weak |
| 12 | [xtylerai2026-oss/autodesk-inventor-mcp](https://github.com/xtylerai2026-oss/autodesk-inventor-mcp) | MIT | Python COM, stdio | 0 / 0 | 2026-08-06 | 1 (1) | **24** | `run_python_com`, `manage_ilogic_rule` | 2027 (alpha) | Weak |
| 13 | [effectustasi/autodesk-inventor-mcp](https://github.com/effectustasi/autodesk-inventor-mcp) | MIT | Single-file Python stdlib; spawns PowerShell per call → COM | 1 / 0 | 2026-10-03 | 4 (1) | **6** | `script` (arbitrary PowerShell) | dev on 2027 | Weak |
| 14 | [NeonGlay/inventor-mcp](https://github.com/NeonGlay/inventor-mcp) (**archived**, upstream of #2) | MIT | Python COM, FastMCP | 53 / 23 | 2026-06-10 | 4 (1) | **36** (README says 34) | `execute_python` | 2026 | Weak (fresh installs break; see §4.14) |
| 15 | [dariusnaicker/inventor-ai](https://github.com/dariusnaicker/inventor-ai) | MIT | Python COM, FastMCP | 8 / 4 | 2026-04-29 → 04-30 | 6 (2) | **36** | `inventor_run_python` | 2026 | Weak |
| 16 | [deadbringer17/inventor-so-mcp](https://github.com/deadbringer17/inventor-so-mcp) | MIT (Python) + Apache-2.0 (C# bridge derived from ipt-mcp) | Hybrid: C# bridge derived from ipt-mcp + Python tools from NeonGlay; adds Quest 3 XR client | not retrieved | → 2026-10-29 | 66 (shallow clone) | ~36 Py + C# bridge | yes (inherited) | 2027 | Weak |

### 2.3 Commercial or closed source (code not inspectable)

| # | Product | Vendor | Licensing | Claimed tools | Inventor | Evidence |
|---|---|---|---|---|---|---|
| 17 | **Inventor AI** ([Modern-CAD/inventor-ai-mcp-server](https://github.com/Modern-CAD/inventor-ai-mcp-server), docs-only repo) | ModernCAD Pty Ltd (AU) | Proprietary; vendor onboarding | 470 (Nov 2026); 255 in the Apr 2026 blog | 2025+ | Insufficient |
| 18 | **AEC Pilot** ([hmertkaramann/aec-pilot-mcp](https://github.com/hmertkaramann/aec-pilot-mcp), docs-only) | 7Dimensions OÜ | Commercial, 14-day trial; in the official MCP Registry as `com.aecpilot/aec-pilot` | 550+ across 9 Autodesk apps incl. Inventor | not stated | Insufficient |
| 19 | **Pierski Technologies "Inventor MCP" 1.1.0** ([pierskitech.com](https://pierskitech.com/products/INVENTORMCP)) | Pierski Technologies (Boyne City, MI) | "Free and MIT licensed", but **no source link found** | 100 + generic COM get/set/invoke | **2026 only** | Insufficient |
| 20 | **ForgeMCP Desktop: Inventor** ([forgemcp.ai](https://forgemcp.ai/servers/autodesk/inventor)) | ForgeMCP | $50/seat/month add-on; "Early Access final beta" | 57 of 86 | 2024+ | Insufficient |

### 2.4 Stubs, toys, docs-only and non-compliant (for completeness)

| # | Project | What it actually is |
|---|---|---|
| 21 | [Cadtastic-Solutions/Autodesk-Inventor-MCP](https://github.com/Cadtastic-Solutions/Autodesk-Inventor-MCP) | Apache-2.0 scaffold. 5 session-management tools only (`inventor_versions_list`, `inventor_session_start/adopt/list/terminate`). Uses C# MCP SDK `0.3.0-preview.4`. 10 commits on one day (2026-05-21). |
| 22 | [gkniksch/Inventor2025_SimpleMcpServer](https://github.com/gkniksch/Inventor2025_SimpleMcpServer) | C# .NET 8, 4 tools: `execute_inventor_csharp`, `get_document_context`, `query_geometry`, `reflect_inventor_api`. Inventor 2025. 3 commits (2026-11-11). |
| 23 | [Learn-to-innovate-and-Surpass/06-InventorBridge](https://github.com/Learn-to-innovate-and-Surpass/06-InventorBridge) | Python, 12 narrow tools (plate and "quickchanger" templates, preview, explode). Niche. |
| 24 | [Abhishek25099611/InventorMcpServer](https://github.com/Abhishek25099611/InventorMcpServer) | **Not MCP-compliant.** Hand-rolled JSON loop answering `listTools`/`callTool` instead of `initialize`/`tools/list`/`tools/call`. Tools: `echo`, `get_inventor_version`, `create_cylinder`. 1 commit. |
| 25 | [allen2123231/inventor-mcp](https://github.com/allen2123231/inventor-mcp) | Traditional-Chinese repackaging. 36 Python tools, the same count as NeonGlay; likely derivative, but provenance not verified. 5 commits. |
| 26 | [bschke/mcp-inventor](https://github.com/bschke/mcp-inventor) | German, 19 Python tools, 1 commit (2026-11-08), no license. |
| 27 | [HanDuDew/Han-s-Inventor-MCP](https://github.com/HanDuDew/Han-s-Inventor-MCP) | Inventor 2023, 8 Python tools, "inspired by neka-nat/freecad-mcp". 10 commits, last 2026-05-12. |
| 28 | [omzambare11/InventorMcpServer](https://github.com/omzambare11/InventorMcpServer) | C#, ~31–33 tool attributes, no README, no license. Last commit 2026-07-02. |
| 29 | [esimers/AutodeskInventor27-MCPServer](https://github.com/esimers/AutodeskInventor27-MCPServer) | A **documentation** MCP: serves Inventor SDK CHM help. Does not drive Inventor. |
| 30 | [dkoustubh/AutoDesk-Inventor-MCP](https://github.com/dkoustubh/AutoDesk-Inventor-MCP) | README is about an "OmniCAD" platform. No MCP tool decorators found. |
| 31 | reactiv/inventor-mcp, papcont/autodesk-inventor-mcp_1, 4engtechlab/4ENG-Inventor-MCP | Empty or near-empty repos (0–8 KB). tratechlab/4ENG-Inventor-MCP has a release named "4ENG-Inventor-2027-MCP", but the clone held only a README. |
| 32 | [AndrzejKalinowski/inventor-mcp](https://github.com/AndrzejKalinowski/inventor-mcp) | 1-commit snapshot of hsavas (89 tools). Its author later contributed joint tools upstream to hsavas. |
| 33 | [emirgoletic-jpg/ai-cad](https://github.com/emirgoletic-jpg/ai-cad), [EdwardNguyen2854/cad-agent](https://github.com/EdwardNguyen2854/cad-agent), [MAvinash24/autodesk-cad-mcp-agent](https://github.com/MAvinash24/autodesk-cad-mcp-agent) | Concept or README-only repos (0–3 KB). Not functional Inventor MCPs. |

### 2.5 Inventor–AI bridges that are not MCP servers (context)

| Project | What it is |
|---|---|
| **Autodesk Assistant in Inventor** | Tech-preview add-in since Inventor 2027 (March 2026). In 2027.2 it also hosts the public MCP server above. |
| [BladerunnerxRC/Claude-AI-Assistant-Autodesk-Inventor](https://github.com/BladerunnerxRC/Claude-AI-Assistant-Autodesk-Inventor) | .NET 4.8 add-in with a docked chat panel calling the Anthropic API. Injects document context and produces iLogic code to copy. MIT, 58 commits. **Not MCP.** |
| [sumitwadkar7/inventor-copilot](https://github.com/sumitwadkar7/inventor-copilot) | External Python process driving Inventor 2024 through COM with an Azure AI deployment. 1 commit. Not MCP. |
| [kivanckizildemir/InventorChatGPT](https://github.com/kivanckizildemir/InventorChatGPT) | Autodesk University 2023 class MFG602716, "DIY: Create a ChatGPT-Based App To Control Inventor". 2023, the earliest precursor found. |
| [autodesk-platform-services/dll2llm](https://github.com/autodesk-platform-services/dll2llm) | Autodesk APS sample (MIT, 27★) that generates LLM context files from .NET DLLs. Relevant as an API-docs-for-agents technique. |
| [RobertBarnettProjects/hermes-shaperotator](https://github.com/RobertBarnettProjects/hermes-shaperotator) | Hermes Agent plugin that uses inventor-forge as its Inventor backend. |

---

## 3. Autodesk official status

### 3.1 What is shipped (primary sources, fetched directly from help.autodesk.com)

**Inventor 2027.2 What's New → "AI Enhancements"** (`/cloudhelp/2027/ENU/Inventor-WhatsNew/files/INV2027-2-AI-ENHANCEMENTS.htm`) says:

> "Inventor Public MCP Server — Inventor provides a local MCP (Model Context Protocol) server that allows an MCP-capable AI agent, such as Cursor, Claude Code, or GitHub Copilot, to inspect and modify the model in an Inventor session that you already have open. The agent has access to the same set of tools used by the built-in Autodesk Assistant."

The same page says Autodesk Assistant in 2027.2 gained iLogic and Python script execution, event triggers, file upload (5 files), and Vault interoperability.

**Autodesk MCP Server help → "Autodesk Inventor MCP Server"** (`/cloudhelp/ENU/ADSKMCP-InventorMcp/files/...`): four pages (overview, connect your client, tools, troubleshooting). Key facts, quoted or closely paraphrased:

- **Connection:** "Local MCP Server (STDIO). The AI client runs on the same Windows computer as Inventor."
- **Availability:** "Autodesk Inventor 2027.2 or later." Requires the Autodesk Assistant add-in (bundled) and **Inventor signed in to an Autodesk account**: "The server rejects connections while Inventor is signed out, and stops running tools if you sign out during a session."
- **Architecture:** `AI client ──stdio──▶ InvMcpStdio.exe ──named pipes──▶ Inventor (Autodesk Assistant Add-In)`
  - The executable lives at `C:\Program Files\Autodesk\Inventor 20XX\Bin\AutodeskAssistant\InvMcpStdio.exe` and takes the argument `/Inventor`.
  - `/InvRO` (Inventor Read-Only) is rejected.
  - "The proxy never starts Inventor."
- **Limits:**
  - "Only one external AI client can be connected to an Inventor instance at a time."
  - With several Inventors open, it connects to "the first supported instance that is ready".
  - "Cancelling a request from the AI client does not stop a tool that is already running."
- **Transactions:** "Most write tools run inside a single Inventor transaction, so each change appears as one entry in the Undo list." Exceptions: iLogic rules run by `execute_ilogic_code` or `run_ilogic_rule` are not rolled back if they fail partway. Changes to external rule files, global event triggers and external rule directories cannot be undone.
- **Code execution:**
  - "`execute_python_script` and `execute_ilogic_code` run code with your Windows user permissions."
  - "Python scripts are screened for unsafe modules, but this screening is not a sandbox." The blocked list is `os`, `subprocess`, `pathlib`, `io`, network libraries and `exec`/`eval`.
  - Python 3.13+ is required. Timeout is 30 s by default, 300 s maximum.
  - `run_ilogic_rule` supports a `check_only` security scan.
- **Annotations:** read-only tools are marked read-only; modifying tools are marked as such.
- **Units:** "document values such as dimensions and masses are returned in the document's own units". There is no unit normalization layer.
- **Capabilities advertised:** "This server does not expose MCP resources or prompts. Only the tools capability is advertised."
- **Turning it off:** unload the Autodesk Assistant add-in, or delete its manifest from `%ALLUSERSPROFILE%\Autodesk\Inventor 20XX\Addins\Assistant`. This also disables Assistant.
- **Data flow:** "Information that tools return … is passed to your AI client and the AI model it uses."

**Full official tool list (46)**, from `inventormcp_tools.html`:

| Group | Tools |
|---|---|
| Files and documents | `get_file_information`, `open_file`, `get_files`, `close_documents`, `get_all_open_documents`, `get_document_info`, `save_document`, `get_document_settings`, `get_document_selections`, `find_document_objects`, `update_document`, `export_document_as_pdf`, `export_document_as_image`, `edit_document_settings` |
| Properties | `get_document_property_sets`, `get_document_properties`, `get_occurrence_properties`, `edit_properties`, `delete_properties` |
| Parameters | `get_document_parameters`, `create_user_parameter`, `edit_parameter`, `delete_parameters` |
| Views | `set_view` |
| Components | `place_component`, `get_component_states`, `set_component_state`, `get_referenced_components` |
| Representations | `get_representations`, `get_member_table`, `edit_member_table`, `edit_representation` |
| Projects and materials | `get_project_info`, `find_materials` |
| Scripting guidance | `get_script_documentation` (ILOGIC / PYTHON / INVENTORAPI lookup) |
| iLogic | `execute_ilogic_code`, `list_ilogic_rules_and_forms`, `read_ilogic_rule`, `run_ilogic_rule`, `create_ilogic_rule`, `edit_ilogic_rule`, `delete_ilogic_rule`, `manage_external_rule_directories` |
| Python | `execute_python_script` |
| Event triggers | `get_event_triggers`, `edit_event_triggers` |

Observations:
- There are **no sketch, feature, constraint or drawing-view creation tools.** Modeling must go through `execute_ilogic_code` or `execute_python_script`.
- There is no viewport capture returned as MCP image content. `export_document_as_image` writes a file instead.
- `save_document` "overwrites without confirmation" when given an existing `outputPath`. There is no Vault integration in the MCP tools.

### 3.2 Release stage: announced, beta or shipped?

- **Shipped in product:** yes, from 2027.2. Release date 2026-11-03 per [n23d.com](https://n23d.com/cad/openai-codex-cad-23x-inventor-2027-2-mcp-fusion-informed-design/) and [borisinventor.com](https://borisinventor.com/our-blog/tpost/autodesk-inventor-ai-au-2026). The autodesk.com "2027.2 what's new" blog itself returned 403; I saw it only as a search snippet.
- **Release-stage label:** a search engine's index of `help.autodesk.com/view/ADSKMCP/ENU/` shows an Inventor card reading "A local MCP server that provides AI-assisted tools for querying, analyzing, and modifying Autodesk Inventor 2027.2 documents, including iLogic and Python automation … 46 Tools", labelled **Public Beta**.
  - When I rendered that landing page through WebFetch, it showed only six cards: AutoCAD/Civil 3D and Revit as "Tech Preview (Public Beta)"; Fusion Compute as "Public Beta"; Fusion Data, Fusion local and Product Help as "General Availability". There was no Inventor card, possibly a stale render.
  - The help table of contents I downloaded does include the four Inventor MCP pages.
  - **Conclusion: treat it as Public Beta / Tech Preview. The label is not directly verified.** The Inventor What's New page does not label it beta.
- **Autodesk Assistant in Inventor** was labelled **"Autodesk Assistant (Tech Preview)"** in the Inventor 2027 What's New. The Autodesk developer blog of 2026-04-22 said it "does not yet create or modify geometry directly". The 2027.2 help says Assistant can "Create, edit, and manage features, properties, and representations".
- **Not in Autodesk's public integration repo or the MCP Registry:**
  - [Autodesk/ai-integrations](https://github.com/Autodesk/ai-integrations) (last commit 2026-10-29) contains only Product Help and Fusion.
  - The official MCP Registry has `com.autodesk.aps.api/fusion-local-mcp` and `com.autodesk.aps.api/product-help-mcp`, and no Autodesk Inventor entry.
  - The Inventor server ships only inside the Inventor installer.

### 3.3 Announced but not shipped (AU 2026, September 2026)

From the Autodesk newsroom post dated 2026-09-15:
- "Autodesk Assistant now unlocks Inventor's robust APIs and iLogic foundation to analyze models…"
- "Inventor will also become the first Fusion Connected Client". This is future tense.
- The post does not mention MCP.

BORIS/CADChain (a third-party vendor) characterises the next-generation Assistant and the Fusion Connected Client as "announced, not shipping", with "No date given".

### 3.4 Other Autodesk products, for contrast (not Inventor)

- **Fusion:** local Fusion MCP (GA, in the MCP Registry), Fusion Data MCP (remote, GA), Fusion Compute MCP (remote, Public Beta).
- **Revit:** 2027.2 local MCP server (Tech Preview / Public Beta).
- **AutoCAD and Civil 3D:** local MCP server (Tech Preview / Public Beta, 7 tools).
- **Product Help:** remote MCP (GA). It covers Inventor documentation but does not drive Inventor.
- Stale "Inventor has no official MCP" statements appear in the xtylerai2026-oss README, the dariusnaicker README, a Leo AI blog, and search snippets of autodesk.com/solutions pages. All predate 2027.2 and are now **contradicted** by Autodesk's own help.

---

## 4. Per-project deep dives

> Tool lists below come from source scans. Where a README's count disagrees with the source, both are given.

### 4.0 Autodesk Inventor MCP Server (official)

Covered in §3. Assessment for adoption:

| Aspect | Notes |
|---|---|
| **Pros** | Vendor-built. In-process (add-in) execution, so no COM cross-process marshalling cost. Transactions per write tool. Read-only annotations. Rule `check_only` security scan. Python module screening. Explicit admin kill-switch. Tool list served live from the running version. Assistant shares the same tools, so Autodesk dogfoods them. |
| **Cons** | 2027.2+ only. Sign-in required. One agent per Inventor instance. No modeling primitives. No image content returned. No resources or prompts. Beta. Proprietary, so it cannot be extended. iLogic tools run outside transactions. Behaviour across multiple Inventor instances is "first ready instance". |
| **Unknowns** | Whether tool calls or telemetry go through Autodesk cloud. Licensing and entitlement (subscription only? Inventor LT/Professional?). Stability. Roadmap to GA. |
| **Evidence** | **Moderate–Strong** that it exists and works as documented on 2027.2. **Insufficient** for production-readiness: there are no independent reports. |

### 4.1 bimwright/ipt-mcp (Apache-2.0), the most engineered open-source option

**Metadata**

| Item | Value |
|---|---|
| Author | GitHub org "bimwright"; all 234 commits by one author ("Khoa.Le"), 2026-05-29 → 2026-11-16 |
| Stars / forks | ~19 / 11 |
| Releases | v0.1.0 and v0.2.1 (the v0.1.0 ZIP ships add-ins for **2025 and 2027** only; other years must be built locally) |
| Issues | 2 total: #1 closed, #2 open |
| PRs | 0 |
| Sibling projects | Same org has rvt-mcp (135★), dwg-mcp and nwd-mcp. ipt-mcp's `server.json` declares `io.github.bimwright/ipt-mcp`, but **it is not in the official MCP Registry**; its siblings are. |

**Architecture**
- `Bimwright.Ipt.Server.exe`: a .NET 8 MCP stdio server built on the official C# MCP SDK. It has no Inventor reference.
- `Bimwright.Ipt.Plugin.InvNN.dll`: an `ApplicationAddInServer`, one thin shell per year 2022–2027, compiled from a shared source glob.
- Transport is **NDJSON over TCP on `IPAddress.Loopback` (2022–2024, net48)** or **named pipe (2025–2027)**. Each add-in session generates a **random token**, written to a discovery file in `%LOCALAPPDATA%\Bimwright\ipt-mcp\inventor-<year>-<pid>.json` (SECURITY.md).
- Work is marshalled onto Inventor's STA thread via a hidden message-only WinForms control (`InventorStaDispatcher`).
- Multi-instance targeting: `inventor_list_available_targets` and `inventor_switch_target`, with error codes AMBIGUOUS_TARGET, TARGET_LOST and TARGET_REBIND_REQUIRED.
- **Document confirmation before writes** (`document_target_v1`): a write returns TARGET_UNCONFIRMED until the user confirms the active document.

**Tools: 150**, from `[McpServerTool(Name="inventor_…")]` in `src/server/Tools/*.cs`, matching the README's "150 tools".
- `--disable-send-code` → 144 tools; `--read-only` → 39 tools; 19 toolsets selectable with `--toolsets`.
- Names (prefix `inventor_` omitted): accept_bake_suggestion, add_balloon, add_constraint, add_drawing_dimension/note/symbol/table/view, add_section_view, add_sheet, add_sketch_constraint/dimension/point_dimension, batch_execute, build_solids, capture_sheet, capture_view, chamfer, change_ipart_member, check_interference, circular_pattern, close_document(s), close_sketch, combine, create_bake_issue_draft, create_bim_connector, create_design_view, create_imate, create_ipart_factory, create_parameter, create_part, create_sketch, create_work_axis/plane/point, delete_code_module, delete_drawing_items, delete_occurrences, derive_envelope, dismiss_bake_suggestion, draw_arc/circle/line/rectangle/text, edit_drawing_annotation/table/view, edit_sheet, export_bodies_sat, export_drawing, export_dxf/sat/step/stl, extrude, fillet, find_view_geometry, flat_pattern, generate_ipart_members, get_assembly_bom, get_current_target, get_document_info/state, get_drawing_info, get_ipart_info, get_iproperty, get_mass_properties, get_parameter, get_report_history, health, hide_view_edges, hole, ilogic_get_rule, ilogic_list_rules, ilogic_read_log, ilogic_run_external_rule, ilogic_run_rule, ilogic_run_snippet, ilogic_upsert_rule, link_excel_table, list_available_targets, list_bake_suggestions, list_baked_tools, list_bodies, list_code_modules, list_commands, list_constraints, list_edges, list_excel_tables, list_faces, list_features, list_interfaces, list_iproperty_sets, list_occurrences, list_open_documents, list_parameters, list_sketch_points, loft, manage_checkpoints, measure_min_distance, new_assembly, new_drawing, new_part, new_sheet_fabrication_part, open_document(s), place_ipart_member, place_occurrence(s), probe_brep, project_geometry, rectangular_pattern, reload_excel_table, report_task_result, reset_appearance, revolve, run_baked_tool, run_command, save_all, save_checkpoint, save_code_module, save_document, send_code, set_appearance, set_camera, set_drawing_styles, set_ipart_row, set_iproperty, set_material, set_occurrence_state, set_parameter, set_sheet_fabrication_thickness, set_title_block, set_units, set_view_orientation, set_view_state, sheet_fabrication_cut/face/flange/info, sketch_on_view, switch_target, sync_ipart_excel, undo, unwrap_try, update_ipart_table, view_fit.

**Capabilities and handling**

| Aspect | Details |
|---|---|
| Escape hatch | `inventor_send_code` (C#, compiled in the add-in; Roslyn references are mentioned in the commit log) is **enabled by default**. It can be removed with `--disable-send-code` or killed host-side with `BIMWRIGHT_INVENTOR_PLUGIN_DISABLE_SEND_CODE=1`. Also `ilogic_run_snippet`. A "ToolBaker" can turn repeated scripts into "baked" tools (`list_bake_suggestions`, `run_baked_tool`). |
| Screenshot | `capture_view` and `capture_sheet` write PNG files. |
| Units | "All length inputs are in mm, angles in degrees; the add-in converts to Inventor's internal centimetres/radians." |
| Undo / transactions | `batch_execute` runs in a document transaction. Rule execution and code are blocked inside batches because "their effects can escape its document transaction". There are `undo`, `save_checkpoint` and `manage_checkpoints` (SHA-256-verified file copies). |
| Error handling | Structured error codes: HOST_BUSY (a modal dialog is open, so the command was not started), TIMEOUT (outcome unknown), TARGET_BUSY, UNAUTHORIZED. `SilentOperation` is on by default for save, open and export. There is an output-size guard (1 MiB budget, spill files). |

**Maturity**

| Signal | Evidence |
|---|---|
| Tests | 122 test files (xUnit) |
| CI | `.github/workflows/build.yml` runs on `windows-latest` with `dotnet build` and `dotnet test` (no Inventor in CI). Live Inventor 2027 "disposable fixture" records are in `docs/testing` and `docs/benchmarks`. |
| Documentation | Extensive, in 4 languages. |
| Third-party mention | [Vagon blog](https://vagon.io/blog/automate-autodesk-inventor-with-claude-code-on-vagon) (2026-10/11) recommends it. Vagon is a cloud-PC vendor with a commercial interest. |
| External bug report | Issue #1 (2026-08-19, closed): "WithTools(IEnumerable<Type>) in Program.cs never registers a tools/list handler". This was a release where clients saw no tools, fixed afterwards. |

**Security**
- Authenticated local transport (token), loopback-only TCP, current-user data dirs.
- `ExportPathPolicy.cs` requires export paths "under an allowed output root (user profile, temp directory, or BIMWRIGHT_INVENTOR_EXPORT_ROOT)".
- `SecretMasker` and `ErrorSanitizer` strip paths from errors returned to the model.
- A read-only mode is available.
- Downside: send_code is on by default.
- **Evidence: Moderate.**

**Worth borrowing**
- The per-year add-in shells built from one shared source, with a version-agnostic server.
- STA marshalling without ExternalEvent.
- Token-authenticated discovery files.
- HOST_BUSY versus TIMEOUT semantics, where the "not started" versus "unknown outcome" distinction is very useful for agents.
- Document-confirmation gating for writes.
- Toolset gating and read-only mode.
- Export-root allowlist.
- Checkpoints.
- mm/deg at the edge.

**Naive or risky**
- Very large surface (150 tools) consumes context; toolsets mitigate this.
- Docs are extremely verbose and over-engineered (the health/update-check prose).
- A single maintainer pushes 10+ commits per day, so there is churn and API-break risk.
- Only 2025 and 2027 binaries are shipped.

### 4.2 cenitu/inventor-mcp-server (MIT), the best Python/COM option

**Metadata**

| Item | Value |
|---|---|
| Origin | "Maintained continuation of NeonGlay/inventor-mcp", keeping the full upstream history |
| Commits | 118 by cenitu, 4 by NeonGlay, 2 "Claude"; 2026-06-10 → 2026-11-15 |
| Stars / forks | ~7 / 1 |
| Releases | Tags v0.2.0 … v1.1.0; `pyproject` version 1.1.0, "Development Status :: 4 - Beta" |
| Distribution | Published on **PyPI** (`inventor-mcp-server`) and in the **official MCP Registry** (`io.github.cenitu/inventor-mcp-server`; the registry shows 1.0.1 as latest). Listed in the punkpeye awesome list and on Glama. |
| Issues and PRs | 7 issues (all self-filed; 1 open: "Run the smoke test on Inventor 2024 / 2025"). 52 PRs, self-merged. |

**Architecture**
- Python 3.12+, pywin32 late-bound COM out of process (`GetActiveObject` / `DispatchEx`).
- Deliberately avoids `gencache.EnsureDispatch`, because "they generate a gen_py cache that breaks plain GetActiveObject".
- MCP SDK 1.x `FastMCP` or 2.x `MCPServer`, auto-detected, with `mcp.run(transport="stdio")`.
- Calls run on a dedicated Inventor thread with `SilentOperation`.
- A `keeper.py` process holds server-started Inventors across client restarts.
- Supports several Inventor instances via the `instance=` parameter.

**Tools: 65**, from `inventor_mcp_server/server.py`: execute_python, load_library, reload_api, capture_view, transaction, connect, list_instances, get_status, find_faces, find_edges, inspect_model, manage_features, manage_parameters, manage_sheet_metal_rules, manage_materials, manage_constraints, manage_occurrences, manage_documents, create_document, add_hole, sheet_metal_cut, add_assembly_cut, place_components, constrain, manage_call, save_document, export_document, create_sketch, draw_rectangle, draw_circle, draw_line, draw_polygon, draw_closed_profile, draw_arc, draw_slot, draw_pattern, check_sketch, offset_profile, project_edges, extrude, revolve, fillet, chamfer, shell, loft, sweep, add_circular_pattern, add_rectangular_pattern, add_mirror, add_sheet_metal_face, add_sheet_profile, add_flange, add_corner_round, export_flat_pattern, export_bom, edit_properties, convert_surfaces_to_solid, create_drawing, use_content_center, add_frame_members, import_step, check_interference, check_manufacturability, measure_distance, sweep_rotation. It also serves the API field notes as the MCP resource `inventor://notes/api`.

**Capabilities and handling**

| Aspect | Details |
|---|---|
| Escape hatch | `execute_python`: arbitrary Python in the server process with `inv`, `app`, `doc`, `comp` and `tg` pre-bound and a persistent namespace. Optional `undo_on_error=True` wraps the run in a `TransactionManager.StartTransaction` and aborts on exception. |
| Screenshot | `capture_view` returns images to the agent and restores the user's camera. |
| Units | Tools in mm; the docstring warns "Internal COM units are CENTIMETERS". |
| Undo | A `transaction` begin/commit/abort tool. |
| Errors | COM HRESULT names plus hints. A call stuck behind a modal dialog returns within 50 s with the dialog title read from the window list; `manage_call` follows or cancels it. |
| Feature reporting | Every feature reports a volume and topology delta, which helps agent self-verification. |
| File protection | `save_document` refuses library and Content Center files; protected paths can be set via `INVENTOR_MCP_PROTECTED`. |

**Maturity**
- Live smoke tests in `tests/smoke_test.py` and `tests/assembly_smoke_test.py`, run by contributors: "137/137 pass on Inventor 2027.1", assembly "30/30".
- CI (`ci.yml`, windows-latest) only does import, description, YAML and wheel checks, because "Inventor can't run on a CI runner".
- Agent Skills are included (`skills/inventor-modeling`, `inventor-din-parts`).

**Security**
- stdio only (no network listener).
- `execute_python` gives the model full machine access with the user's rights. There is no flag to disable it that I found; I did not exhaustively verify this.
- **Evidence: Moderate.**

**Worth borrowing**
- Volume and topology delta after every feature.
- Dialog-detection timeout that does not block.
- The "never use gencache" rule.
- Undo-on-error wrapper for scripts.
- Field notes on Inventor 2026 and 2027 API quirks: the flange distance is ignored, angles are in radians, `CreateLinearPlacementDefinition` needs a BiasPoint, and so on.
- Protected-file list.
- Keeper process for instance persistence.

**Naive or risky**
- Unrestricted Python execution.
- Single real maintainer.
- Out-of-process COM is slower than in-process for bulk work. This is not measured here.

### 4.3 tylerwarner33/autodesk-inventor-mcp (MIT), developer-oriented, best code-exec safeguards

**Metadata**

| Item | Value |
|---|---|
| Commits | 159 by Tyler Warner, plus dependabot and changelog bots; 2026-09-21 → 2026-11-16 |
| Stars | ~3 |
| Version | 0.4.7 (`Directory.Build.props`); no tags or releases ("No team feed and no release pipeline exist yet") |
| Issues / PRs | 0 issues; 35 PRs |

**Architecture**
- A .NET 10 `InventorMcp.Server` distributed as a dotnet tool (`dotnet tool exec`) on stdio.
- `InventorMcp.AddIn` runs inside Inventor 2025, 2026 or 2027 and hosts the named pipe `InventorMcp.Bridge.<year>`.
- The server verifies the session, executable path and owner of the pipe host before sending.
- It uses the official C# SDK `ModelContextProtocol` 2.4.0 plus `ModelContextProtocol.Extensions.Skills`.

**Tools: 35**, from `Source/InventorMcp.Server/McpTools/*.cs`, excluding test tools: inventor_session, inventor_start, inventor_use_release, inventor_documents, inventor_close_documents, inventor_file_info, inventor_test_copy, inventor_assembly_tree, inventor_parameters, inventor_set_parameter, inventor_set_parameters, inventor_evaluate_expression, inventor_properties, inventor_set_property, inventor_health, inventor_features, inventor_pattern_elements, inventor_hole_check, inventor_styles, inventor_update, inventor_activity, inventor_eval_csharp, inventor_run_ilogic, inventor_ilogic_rules, inventor_ilogic_rule_get, inventor_ilogic_rule_set, inventor_api_lookup, inventor_skill, inventor_orientation, inventor_run_plugin, inventor_drawing_layout, inventor_export_sheet_image, inventor_auto_balloon, inventor_dialogs, inventor_dialog_click.

**Capabilities and handling**

| Aspect | Details |
|---|---|
| Escape hatch | `inventor_eval_csharp` (Roslyn script). Safeguards: every snippet is appended to `executed-code.log` with a SHA-256 before it runs; it is **refused on documents with unsaved changes** unless `allowUnsavedChanges` is passed; it runs inside a `SilentOperationScope`; output is capped at 60k characters. |
| Limits | `INVENTORMCP_TOOLS=read-only` and `INVENTORMCP_CONFIRM_RISKY_CALLS=true`, which uses MCP elicitation to confirm. |
| Dialogs | A dialog policy file auto-answers known dialogs or leaves them for the user. |
| Screenshot | `inventor_export_sheet_image` returns PNG image content (drawings only). |
| Units | Parameter results carry the internal cm/rad value as well as the display value. |
| Activity feed | `inventor_activity` is an ordered feed of Inventor events, including committed transactions. |

**Maturity**
- 43 test files.
- CI builds the add-in for each Inventor release matrix and runs unit tests (no Inventor).
- OpenTelemetry is optional and **off by default**.

**Concern: it vendors `Autodesk.Inventor.Interop.dll`** for 2025 and 2026 under `Libs/Inventor/`. Redistributing Autodesk interop binaries may conflict with Autodesk's SDK terms. That is not verified; bimwright explicitly refuses to redistribute.

**Evidence: Moderate** for 2025–2027. The focus is plugin development and inspection rather than modeling: there are no sketch or feature tools, and modeling goes through `eval_csharp`.

**Worth borrowing**
- Code-execution audit log.
- Unsaved-changes guard.
- Env-var tool tiers.
- Dialog inspection and click tools.
- `inventor_activity` event feed.
- `inventor_api_lookup` against the vendored API docs.
- Skills over MCP.

### 4.4 gustacassel/InventorBridge (MIT)

**Metadata:** 21 commits by one author, 2026-10-28 → 2026-11-16. No releases.

**Architecture**
- .NET 8 out-of-process COM; "nothing is installed inside Inventor".
- `run_csharp` compiles and runs C# against the Inventor API "as one undo step on the active document and is rolled back on an error".
- An `ARCHITECTURE.md` (v1, 2026-10-25) surveys NeonGlay, effectustasi, mcp-cad and hsavas, and explicitly states "Do not copy code" from hsavas because of its license.

**Tools: 19**, from `src/InventorBridge.Mcp/Tools/InventorTools.cs`: inventor_status, inventor_start, run_csharp, script_result, inspect, bom, export, check, undo_last, ilogic_list, ilogic_set, ilogic_run, ilogic_script, vault_info, vault_change, log_note, topology, screenshot, api_lookup, guide.

**Distinctive features**
- A generated **API reference for Inventor 2024–2027** under `docs/reference`: about 2,250 files, plus `members.tsv` with `// since 20xx` markers.
- **Vault** read and check-in/out through Inventor's Vault connection.
- Company-specific guide folders (`INVENTORBRIDGE_GUIDES`).

**Maturity:** 13 test files. **Evidence: Weak–Moderate**, because it is three weeks old.

**Worth borrowing:** the interop-reflection-generated API docs served to the agent; the "few tools, one strong executor" philosophy; the transaction-per-script default; company guide injection.

### 4.5 hsavas/inventor-mcp: broad coverage, but non-commercial license

**Metadata**

| Item | Value |
|---|---|
| Commits | 11, by hsavas, Andrzej Kalinowski and "daman"; 2026-06-23 → 2026-10-20 |
| Stars / forks | ~9 / 6 |
| License | **"Inventor MCP Non-Commercial License … Any commercial use of the Software requires prior written permission"**. GitHub reports it as NOASSERTION. |

**Architecture:** Python pywin32 COM (uses `win32com` constants, so the gencache path applies), FastMCP over stdio. A machine-wide lock prevents launching duplicate Inventors.

**Tools: 109**, from `inventor/tools/*.py`, matching the commit "Test all 109 tools". They cover:
- 21 sketch tools, 21 feature tools, 14 assembly tools, 5 joint tools, 14 drawing tools;
- 3D sketch, sheet metal, threads, work geometry, parameters, iProperties, mass, appearance;
- `inventor_screenshot`, `inventor_zoom_all`, `inventor_export`;
- `inventor_run_script`.

The full name list is in the scan output; examples are `inventor_assembly_add_joint`, `inventor_drawing_add_base_view` and `inventor_sheet_metal_flat_pattern`.

**Handling**
- Escape hatch `inventor_run_script` runs arbitrary Python. The README warns it has "full access to your machine"; it can be disabled with `INVENTOR_MCP_DISABLE_SCRIPT=1`.
- Units: mm and degrees, converted in `inventor/units.py`.
- I found no transaction handling (no `StartTransaction`).
- `INVENTOR_MCP_TRACE` provides JSONL tracing.

**Maturity:** `tests/functional_test.py` and `check_single_instance.py` require live Inventor. There is no CI. **Evidence: Weak–Moderate.**

**Takeaway:** a useful coverage reference (joints with `preserve_position`, auto-jointing fasteners, drawings), but **the license disqualifies it for commercial adoption or forking.**

### 4.6 OC-JG/InventorMCP (MIT)

**Metadata:** 251 commits, of which **197 are attributed to "Claude"** and 54 to OC-JG; 2026-08-17 → 2026-09-28. Quiet since then.

**Architecture:** a Python MCP SDK server (FastMCP/MCPServer compat). It works from a JSON "recipe": a part description with named parameters and expressions is validated, then built. There is a DFM analyser submodule that needs Node.

**Tools: 31**, from `inventor_mcp/tools/*.py`: connect, session_status, new_part, open_part, save_part, close_part, activate_part, run_inventor_script, part_recipe_schema, validate_recipe, drawing_recipe_schema, drawing_reading_schema, build_part_from_recipe, apply_operations, set_parameters, edit_feature, rebuild_part, inspect_part, select_topology, measure_part, export_model, capture_view, check_manufacture, improve_for_manufacture, discover_dfm_roles, declare_dfm, promote_parameters, read_dfm_report, protect_geometry, compare_manufacture, dfm_capabilities.

**Maturity:** about 64 test files. CI runs pytest on ubuntu and windows without Inventor. **Evidence: Weak–Moderate.**

**Worth borrowing:** the recipe-schema approach, which validates offline before touching CAD and makes parametric intent explicit; a backend abstraction that allows offline tests.

### 4.7 RobertBarnettProjects/inventor-forge (MIT)

**Metadata:** 12 commits in one week (2026-11-07 → 2026-11-12).

**Architecture:** Python and pywin32, with a **hand-rolled JSON-RPC MCP layer** (no SDK; `server.py` handles `initialize`, `tools/list` and `tools/call`, and echoes the client's `protocolVersion`). It uses one STA thread and reads the 5,000+ `k…` enum constants from the type library at run time.

**Tools:** the README claims 160. The decorator scan found about 170 registrations, including 2 underscore-prefixed probe tools and some duplicates; `TOOLS.md` has 171 rows. Areas:
- sketch, features, standard parts, inspect, parameters, materials;
- sheet metal, assembly, model states;
- 34 drawing tools with standards-based QA (AS 1100, ISO, ASME, DIN, BS 8888, JIS, GB);
- Stress Analysis (18 study tools plus analyze_stress and analyze_modes).

**Notable risk:** `analyze_stress` and `analyze_modes` **drive the mouse and keyboard for 30–60 s**, because Stress Analysis has no API. The tool requires a `user_notified=true` second call.

**Evidence: Weak**, because of age, although the FEA validation example against beam theory is a nice touch.

**Worth borrowing:** reading enums from the type library at run time rather than hard-coding them; automatic cut-direction selection; drawing QA checks.

### 4.8 Andiveli/mcp-cad

**Metadata:** 111 commits by one author ("eSeylaA69"), 2026-05-26 → 2026-06-29. Releases v0.1.0 and v0.2.0 with a GUI installer.

**License conflict:** the LICENSE file is GPL-3.0, while the README says "MIT — fully open". This must be resolved before any reuse.

**Architecture:** C# .NET 8, early-bound COM, official C# SDK 1.3.0. "Provider-agnostic", with SolidWorks "in development". Built by "Spec-Driven Development … 8 days, 80+ tools".

**Tools:** about 118 `[McpServerTool]` methods (90 atomic, 25 "skill", 5 template, 2 macro). Examples: sketch_line, extrude, asm_constraint_mate, weld_fillet, capture_viewport_image, macro_god_part, template_run. There is no arbitrary-code tool.

**Evidence: Weak.** It has been stale since June. Its tag-based entity naming (`@hole_center`) is worth noting.

### 4.9 ioseluiz/mcp_server_autodesk_inventor (no license declared)

**Metadata:** 25 commits by 3 authors, 2026-05-14 → 2026-08-07. Tags v1.0.4 … v1.0.9. Spanish documentation.

**Architecture:** a Python FastMCP server packaged with PyInstaller as `inventor-mcp.exe`, which talks over the named pipe `\\.\pipe\InventorMCPBridge` to a C# .NET 8 add-in for Inventor 2026.
- The pipe has no authentication; the README says the "default ACL already limits access to the user who creates it". I have not verified that claim for .NET's default pipe ACL.
- There is an Inno Setup installer that auto-registers with Claude Desktop.
- It **vendors `lib/Autodesk.Inventor.Interop.dll`** (redistribution concern).

**Tools: 52**, from `mcp_server/main.py`. Sketch, features, parameters, iProperties, work geometry, `render_screenshot`, STEP/STL/DXF export, interference, assembly constraints and joints, BOM, patterns and mirrors.

**Maturity:** 12 test files and two CI workflows (tests and release). **Evidence: Weak.** The repo has **no license**, so it legally cannot be reused.

### 4.10 wuwenhou/inventor2022-mcp (MIT)

1 commit (2026-11-14), so it is likely a public export of private work. Release v1.5.1. C# .NET 10 for Inventor 2022.

**77 tools**, from `src/InventorMcp/*.cs`, matching the README. They include skeleton/layout parametric linking, frame members, sheet metal, preview images and recovery of unsaved parts. **Weak** for current Inventor, which it does not target.

### 4.11 oakplank/InventorMCP (MIT)

5 commits on one day (2026-07-15). C# server plus add-in, an "authenticated, current-user named pipe", and "There is no TCP listener and no endpoint for running arbitrary code."

**14 tools**, from `src/InventorMCP.Server/Tools/InventorTools.cs`: inventor_list_sessions, get_capabilities, get_context, list_documents, get_document_properties, get_selection, list_parameters, set_parameter_expression, list_assembly_occurrences, orient_view, focus_occurrence, set_occurrence_visibility, capture_viewport (returns MCP image content), get_operation_status. Tool annotations (ReadOnly, Destructive, Idempotent) are used.

Live-tested on Inventor 2024.3. **Weak.** It is a clean **security-conservative reference design**.

### 4.12 xtylerai2026-oss/autodesk-inventor-mcp (MIT)

1 commit (2026-08-06), described as alpha. Python COM.

**24 tools**: inventor_status, new_document, open_document, save_document, close_document, describe_model, measure, get_parameters, set_parameters, screenshot, sketch_and_extrude, loft_sections, revolve_profile, pattern_bodies, boolean_combine, place_occurrence, pattern_occurrences, add_constraint, check_interference, get_bom, import_geometry, export_document, run_python_com, manage_ilogic_rule.

**Valuable `docs/design.md` with "Things that turned out not to be true", each backed by a test:**
- Out-of-process COM does produce real solids on 2027.
- `iLogicAutomation.AddRule` executes the rule as it adds it.
- Inventor object names are **localised** (for example `'XY 平面'`), so lookups by English name break.
- A gen_py cache returns typed wrappers that hide members.

Its claim that Inventor has no official MCP "and none is announced" is now **outdated**. **Weak.**

### 4.13 effectustasi/autodesk-inventor-mcp (MIT)

4 commits (2026-10-03), tag v0.2.1. A single stdlib-only Python file with a hand-rolled MCP JSON-RPC layer. Each tool call **spawns `powershell.exe`** and drives COM from PowerShell.

**6 tools**: status, model_info, smallest_faces, face_neighbors, unwrap_try (rolls back by default), and script (arbitrary PowerShell).

A focused inspection tool for import defects. It is listed in the punkpeye awesome list, on Glama and in Enterprise DNA. **Weak** for general use. The process-per-call design is robust but slow (not measured).

### 4.14 NeonGlay/inventor-mcp (MIT, archived), the most-starred project

53★ and 23 forks, but only 4 commits, all on 2026-06-10; the repo is **archived**.

**36 `@mcp.tool` functions** in `src/server.py` (the README says 34): execute_python, reload_api, inspect, list_edges, list_faces, transaction, connect, status, create_part, save_document, export_document, create_sketch, draw_*, extrude, revolve, fillet, chamfer, hole, hole_linear, circular_pattern, get/set/add_parameter, sheet-metal tools, find_edge, find_face, list_features, delete_feature, suppress_feature.

**Clearly broken for fresh installs**, per open issues #5 and #6 (2026-09-09) and my reading of the code:
- `pyproject` pins `mcp[cli]>=1.0.0` with no upper bound, while `server.py` does `from mcp.server.fastmcp import FastMCP`. According to cenitu's README and OC-JG's compat shim, MCP SDK 2.x renamed this to `MCPServer`.
- `server.py` uses a relative import (`from . import inventor_api`), which fails when the script is run directly.

PR #2 (by cenitu) reported "57/57 pass" on Inventor 2027 before the fork. **Use cenitu instead.**

### 4.15 dariusnaicker/inventor-ai (MIT)

6 commits over 2 days (2026-04-29/30), so it is the earliest Python COM server found. **36 tools** (the README says "30+"), including `inventor_run_python` and `inventor_undo`. Targets Inventor 2026. **Weak** (abandoned).

### 4.16 deadbringer17/inventor-so-mcp

66 commits (shallow clone), last commit 2026-10-29. A hybrid: a C# bridge **derived from bimwright/ipt-mcp** (Apache-2.0, attribution preserved) plus Python tools from NeonGlay. It adds a Unity **Quest 3 XR** client. It targets Inventor 2027. It is notable mainly as evidence that ipt-mcp is being forked. **Weak.**

### 4.17 Commercial and closed products (not verifiable)

| Product | What I could see |
|---|---|
| **ModernCAD "Inventor AI"** | Docs-only repo: README, TOOLS.md (470 names in 24 domains, including Scripting `script_execute` and many iLogic tools), EXAMPLES.md. LICENSE: "proprietary commercial software". The Apr 2026 blog claimed 255 tools; Nov 2026 claims 470. It talks to Inventor "locally over COM". No price is published; onboarding is by the vendor. **Insufficient.** |
| **AEC Pilot** | Commercial with a trial; in the official MCP Registry. An add-in-hosted local MCP across 9 Autodesk apps, with an `execute_*_code` tool per app that can be switched off. **Insufficient.** |
| **Pierski Technologies Inventor MCP 1.1.0** | The vendor page says it is free, "MIT licensed", Inventor 2026, 100 tools plus "run-script and get, set and invoke tools reach the whole Inventor object model", with a PowerShell installer. **No source repository is linked**, so the MIT claim cannot be verified. It was mentioned in an Autodesk forum thread (snippet only). **Insufficient.** |
| **ForgeMCP Desktop: Inventor** | $50/seat/month on top of a plan; "Early Access final beta"; Inventor 2024+; local bridge; 57 of 86 tools; scripting is "an opt-in escape hatch" and off by default; the build is code-signed. **Insufficient.** |

---

## 5. Cross-cutting observations

### 5.1 Two architectural camps

1. **Out-of-process COM**, either Python (pywin32) or C#: cenitu, hsavas, OC-JG, inventor-forge, xtylerai, effectustasi, InventorBridge, mcp-cad.
   - Pros: nothing to install in Inventor, and a server crash does not take Inventor down.
   - Cons: cross-process marshalling, Windows ROT gives "first instance only" (cenitu's keeper works around this), and dialogs block COM (cenitu and inventor-forge detect them through the window list).
2. **In-process add-in plus local IPC** (named pipe or loopback TCP): **Autodesk official**, ipt-mcp, tylerwarner33, ioseluiz, oakplank, Cadtastic.
   - Pros: speed, the UI thread is reachable, events are available.
   - Cons: a per-version build and deploy, Inventor must be restarted to update, and redistribution of interop DLLs is a concern.
   - Autodesk itself chose this model.

### 5.2 Escape hatches are universal

Almost every non-trivial server, including Autodesk's, exposes arbitrary code: Python, C#/Roslyn, iLogic VB.NET or PowerShell. Only oakplank and mcp-cad deliberately omit one.

Safeguards ranked by strength (my judgement):
1. tylerwarner33: audit log, unsaved-changes guard, read-only tier, elicitation.
2. ipt-mcp: on by default but disable-able at two layers, kept out of batch transactions, export-root policy.
3. Autodesk: Python module screening ("not a sandbox"), rule `check_only` scan.
4. hsavas: env-var disable.
5. cenitu and NeonGlay: none beyond stdio-only.

### 5.3 Units

All community servers normalise to mm and degrees at the tool boundary and convert to Inventor's internal cm and radians. Autodesk returns "the document's own units", and parameter tools take unit-bearing expressions such as `"120 mm"`.

### 5.4 Transactions and undo

- Autodesk: per write tool, except iLogic.
- ipt-mcp: `batch_execute` plus file checkpoints.
- cenitu: an explicit begin/commit/abort tool plus `undo_on_error` for scripts.
- InventorBridge: every script is one undo step.
- hsavas, dariusnaicker and effectustasi: none found.

### 5.5 Recurring field knowledge worth harvesting

| Source | Knowledge |
|---|---|
| cenitu `notes/inventor-api-notes.md` | Inventor 2026/2027 enum values, the flange-distance bug, the BiasPoint argument, edge renumbering. |
| xtylerai `docs/design.md` | Localisation of object names, AddRule auto-executes the rule, gen_py wrapper pitfalls. |
| tylerwarner33 | `DialogSettings.json` catalogue of the dialogs that block automation. |
| ipt-mcp | HOST_BUSY / TIMEOUT / TARGET_* error taxonomy. |
| InventorBridge | Generated API reference from the interop DLLs; the Autodesk APS `dll2llm` sample is a similar idea. |

---

## 6. Evidence table: "Is there a production-ready Inventor MCP we could adopt instead of building?"

| Supporting evidence (yes, adoptable) | Contradictory evidence (no) | Uncertain / mixed | Gaps / unknowns |
|---|---|---|---|
| Autodesk ships a documented local MCP server in Inventor 2027.2 with 46 tools, transactions on most write tools, read-only annotations and an admin kill-switch. Source: Autodesk help, fetched directly. | Autodesk's server appears to be **Public Beta** (search-index snippet of the help landing page) and is about 2 weeks old. There are no independent field reports. | The release-stage label could not be seen directly; the What's New page does not say beta. | Whether tool calls or telemetry route through Autodesk cloud. Entitlement and licensing (subscription, Inventor Professional vs standard, LT). Roadmap to GA. |
| ipt-mcp: Apache-2.0, 150 tools, 2022–2027, token-authenticated IPC, 122 test files in CI, live 2027 fixture records, read-only and toolset gating, export-root allowlist. A third-party blog (Vagon) recommends it. | Autodesk's server has **no sketch or feature-creation tools**; modeling needs iLogic or Python code execution. It supports only 2027.2+, **one agent per Inventor instance**, and requires sign-in. | ipt-mcp ships prebuilt add-ins only for 2025 and 2027. An earlier release had a "no tools registered" bug (issue #1, fixed). | Behaviour of the official server across multiple open Inventors beyond "first supported instance". Performance at scale (large assemblies). |
| cenitu: MIT, on PyPI and the official MCP Registry. Self-reported 137/137 live smoke on 2027.1 and 30/30 assembly. Active (last commit 2026-11-15). Rich modeling tools, capture_view, transactions. | **Every community project is effectively single-maintainer.** The top star count is 53, on an archived repo. ipt-mcp has 19★ and 0 external PRs; cenitu's issues and PRs are self-filed. Bus factor is 1 everywhere. | Self-reported test results (cenitu, ipt-mcp, inventor-forge) are not independently reproduced. CI never runs Inventor in any project. | No security audit of any project. No published benchmarks of reliability (success rate of agent tasks). |
| tylerwarner33: MIT, CI builds for 2025/2026/2027, mature code-execution safeguards. | NeonGlay (most visible) is archived and **breaks on fresh install** with MCP SDK 2.x. Several others are 1-commit drops, empty repos or non-compliant (Abhishek). | tylerwarner33 and ioseluiz vendor Autodesk interop DLLs, which may be a redistribution concern. | Whether Autodesk's EULA or SDK terms restrict third-party in-process add-ins that expose MCP. Not researched. |
| Several commercial vendors (ModernCAD, AEC Pilot, ForgeMCP, Pierski) claim production use. | Licensing blockers: hsavas is non-commercial; ioseluiz and bschke declare no license; mcp-cad has a GPL-3.0 vs MIT conflict. | Commercial claims (470 tools, "in production use") come from vendor marketing. | Commercial products cannot be inspected; pricing is opaque except ForgeMCP ($50/seat/month). |
| Many projects converge on the same proven patterns (mm/deg at the edge, SilentOperation, STA marshalling, transactions, image capture), which lowers the risk of building or forking. | No project offers a stable API contract: all are versioned 0.x except cenitu 1.1, and churn is high (ipt-mcp makes 10+ commits per day). | Forum (snippet only): one experienced user prefers plain COM scripts from Claude Code without MCP for production output (drawings, DXF). | Autodesk forum threads were inaccessible (403), so community sentiment is under-sampled. Reddit and YouTube searches found no Inventor-MCP-specific discussions or demos. |

**Verdict and evidence strength:**
- Evidence demonstrates that adoptable building blocks exist: Autodesk's server on 2027.2+, and permissively licensed ipt-mcp, cenitu and tylerwarner33.
- Preliminary evidence suggests none is production-ready in the sense of GA status, multi-maintainer support, independent validation and security review.
- **Overall: Moderate.** Source and vendor docs were read; nothing was tested live.

---

## 7. Registry and directory coverage (searched for "Inventor")

| Directory | Inventor entries found |
|---|---|
| **Official MCP Registry** (`registry.modelcontextprotocol.io`) | `io.github.cenitu/inventor-mcp-server` (latest 1.0.1) and `com.aecpilot/aec-pilot` (multi-app). **No Autodesk Inventor entry**: Autodesk publishes only `fusion-local-mcp` and `product-help-mcp`. bimwright publishes rvt, dwg and nwd, but **not** ipt-mcp. Searching "inventor" mostly matches "inventory" servers, which were filtered out. |
| **punkpeye/awesome-mcp-servers** | cenitu/inventor-mcp-server and effectustasi/autodesk-inventor-mcp. |
| appcypher and wong2 awesome-mcp-servers | none. |
| mlightcad/awesome-cad | Inventor appears only as a file format, not an MCP. |
| **Glama** (via search) | NeonGlay, cenitu, effectustasi, hsavas, xtylerai2026-oss, Andiveli/mcp-cad (Glama lists "32 tools"; I counted about 118 in the current source). |
| Smithery, mcp.so, PulseMCP, mcpservers.org (via search) | **No Inventor listings found.** Not exhaustive, because those sites were not crawled directly. |
| Other aggregators seen in search results | Enterprise DNA (effectustasi), claudemarket.ai and remoteopenclaw.com (hsavas), mcprepository.com (xtylerai), mcpmarket.com ("Inventor: AI-Driven Parametric 3D Modeling", probably NeonGlay or cenitu). cline/mcp-marketplace issue #3254 is a submission for cenitu's server. |
| **Autodesk App Store** | No Inventor LLM or MCP add-in found by search. Not verified by browsing the store. |

---

## 8. Things I could not verify (flagged)

1. The official Inventor MCP **release-stage label** (Public Beta). Seen only in a search-engine snippet; the landing-page render I obtained omitted the Inventor card.
2. The **2027.2 release date** (2026-11-03). From two secondary sources; the autodesk.com blog returned 403.
3. Whether the official server needs a particular Inventor edition or subscription, and whether it sends any data to Autodesk cloud.
4. All **"works on Inventor 20xx"** claims for community servers. These are self-reported; no live test was performed.
5. Content of the Autodesk forum threads 14161719, 14113425, 14285262, 14336267 and community-blog 14104913. Only search snippets were available (403). This includes an unverified forum claim that Autodesk Assistant is "based on Claude".
6. Pierski Technologies' MIT claim (no source repo found). ModernCAD's and AEC Pilot's tool counts and production claims (closed source).
7. Stars and forks for deadbringer17, AndrzejKalinowski and tratechlab (the search API returned nothing for those).
8. The exact inventor-forge tool count (README 160 vs about 170 registrations found).
9. Whether vendoring `Autodesk.Inventor.Interop.dll` (tylerwarner33, ioseluiz) is permitted by Autodesk's terms.
10. Whether .NET's default named-pipe ACL restricts access to the creating user, as ioseluiz asserts.

---

## 9. URLs consulted, with credibility notes

### Autodesk primary (high credibility for "what Autodesk says"; vendor self-description, not independent)

| URL | Note |
|---|---|
| https://help.autodesk.com/cloudhelp/ENU/ADSKMCP-InventorMcp/files/ADSKMCP_InventorMcp_autodesk_inventor_mcp_html.html | Official overview of the Inventor MCP. Fetched raw. **Primary.** |
| https://help.autodesk.com/cloudhelp/ENU/ADSKMCP-InventorMcp/files/ADSKMCP_InventorMcp_ide_integration_html.html | Client setup, `InvMcpStdio.exe` path. **Primary.** |
| https://help.autodesk.com/cloudhelp/ENU/ADSKMCP-InventorMcp/files/inventormcp_tools.html | Full 46-tool reference. **Primary.** |
| https://help.autodesk.com/cloudhelp/ENU/ADSKMCP-InventorMcp/files/ADSKMCP_InventorMcp_troubleshooting_html.html | Limits, error messages, transaction caveats. **Primary.** |
| https://help.autodesk.com/view/ADSKMCP/ENU/data/toctree.json | Help TOC that lists the Inventor MCP pages. Primary, structural. |
| https://help.autodesk.com/view/ADSKMCP/ENU/ | Landing page (JS-rendered). The WebFetch render showed 6 cards and no Inventor card; possibly stale. |
| https://help.autodesk.com/cloudhelp/ENU/ADSKMCP-Governance/files/ (3 governance pages) | Server-type and governance guidance. Primary. |
| https://help.autodesk.com/cloudhelp/ENU/ADSKMCP-CommonContent/files/ADSKMCP_CommonContent_about_autodesk_mcp_servers_html.html | Generic MCP overview; no Inventor mention. |
| https://help.autodesk.com/cloudhelp/2027/ENU/Inventor-WhatsNew/files/INV2027-2-AI-ENHANCEMENTS.htm | 2027.2 What's New: "Inventor Public MCP Server". **Primary.** |
| https://help.autodesk.com/cloudhelp/2027/ENU/Inventor-WhatsNew/files/INV2027-ADSK-ASSISTANT.htm | 2027 "Autodesk Assistant (Tech Preview)". Primary. |
| https://help.autodesk.com/cloudhelp/2027/ENU/Inventor-Autodesk-Assistant/files/ABOUT-AUTODESK-ASSISTANT-INVENTOR.htm | Assistant capabilities (2027–2027.2). Primary. |
| https://help.autodesk.com/cloudhelp/2027/ENU/Inventor-WhatsNew/files/INV-What-s-New-in-Inventor-2027.htm (via WebFetch) | Index of 2027.x updates. Primary. |
| https://help.autodesk.com/view/INVNTOR/2027/ENU/data/toctree.json | Inventor 2027 help TOC. Primary, structural. |
| https://blog.autodesk.io/from-commands-to-conversations-exploring-autodesk-assistant-in-inventor-2027/ | Autodesk developer blog, Takehiro Kato, 2026-04-22. Vendor employee; dated before 2027.2. |
| https://adsknews.autodesk.com/en/news/autodesk-ai-design-manufacturing-au-2026/ | Autodesk newsroom, 2026-09-15. Vendor PR. |
| https://www.autodesk.com/blogs/design-and-manufacturing/autodesk-inventor-2027-2-whats-new/ | **403.** Content seen only as search snippets. |
| https://www.autodesk.com/solutions/autodesk-ai/autodesk-mcp-servers | **403.** Search snippets suggest an older version listing no Inventor server. |
| https://github.com/Autodesk/ai-integrations (cloned) | Official Autodesk repo; Fusion and Product Help only; last commit 2026-10-29. Primary. |
| https://registry.modelcontextprotocol.io/v0/servers?search=… | Official MCP Registry API. Primary for listings. |

### Repositories (primary for their own code; authors are mostly unknown individuals with no stated affiliation)

| URL | Note |
|---|---|
| github.com/bimwright/ipt-mcp | Single dev; also maintains rvt-mcp (135★). |
| github.com/cenitu/inventor-mcp-server | Single real maintainer. Includes PR #2 on NeonGlay (seen via search snippet). |
| github.com/tylerwarner33/autodesk-inventor-mcp | Single dev with bots. |
| github.com/gustacassel/InventorBridge | Single dev, 3 weeks old. |
| github.com/hsavas/inventor-mcp | 3 contributors; non-commercial license. |
| github.com/OC-JG/InventorMCP | About 78% of commits attributed to "Claude", so the code is AI-authored. |
| github.com/RobertBarnettProjects/inventor-forge | 1 week old. |
| github.com/Andiveli/mcp-cad | AI-built ("SDD"); license conflict. |
| github.com/ioseluiz/mcp_server_autodesk_inventor | No license. |
| github.com/wuwenhou/inventor2022-mcp | 1-commit export. |
| github.com/oakplank/InventorMCP | 1 day of commits. |
| github.com/xtylerai2026-oss/autodesk-inventor-mcp | 1 commit; useful design notes. |
| github.com/effectustasi/autodesk-inventor-mcp | The author also publishes Blender and Unreal tools. |
| github.com/NeonGlay/inventor-mcp | Archived; issues #5 and #6 read via the search API. |
| github.com/dariusnaicker/inventor-ai | Abandoned after 2 days. |
| github.com/deadbringer17/inventor-so-mcp | Fork-derivative of ipt-mcp and NeonGlay. |
| github.com/Modern-CAD/inventor-ai-mcp-server | Vendor marketing repo; conflict of interest. |
| github.com/hmertkaramann/aec-pilot-mcp | Vendor marketing repo; conflict of interest. |
| github.com/Cadtastic-Solutions/Autodesk-Inventor-MCP | Consultancy scaffold. |
| github.com/gkniksch/Inventor2025_SimpleMcpServer | Small project. |
| github.com/Learn-to-innovate-and-Surpass/06-InventorBridge | Small project. |
| github.com/Abhishek25099611/InventorMcpServer | Non-compliant. |
| github.com/allen2123231/inventor-mcp | Minor. |
| github.com/bschke/mcp-inventor | Minor. |
| github.com/HanDuDew/Han-s-Inventor-MCP | Minor. |
| github.com/omzambare11/InventorMcpServer | Minor. |
| github.com/esimers/AutodeskInventor27-MCPServer | Minor (docs server). |
| github.com/dkoustubh/AutoDesk-Inventor-MCP | Minor. |
| github.com/AndrzejKalinowski/inventor-mcp | Minor (hsavas snapshot). |
| github.com/tratechlab/4ENG-Inventor-MCP | Minor. |
| github.com/BladerunnerxRC/Claude-AI-Assistant-Autodesk-Inventor, github.com/sumitwadkar7/inventor-copilot, github.com/kivanckizildemir/InventorChatGPT | Non-MCP context. |
| github.com/autodesk-platform-services/dll2llm | Autodesk APS sample; metadata via search API only. |
| github.com/AnCode666/multiCAD-mcp (raw README) | AutoCAD-family only; excluded. |
| GitHub search API (13 queries) | Primary listing data; star counts change daily. |

### Directories and lists

| URL | Note |
|---|---|
| https://raw.githubusercontent.com/punkpeye/awesome-mcp-servers/main/README.md | Community-curated; inclusion is not vetting. |
| https://raw.githubusercontent.com/appcypher/awesome-mcp-servers/main/README.md, https://raw.githubusercontent.com/wong2/awesome-mcp-servers/main/README.md, https://raw.githubusercontent.com/mlightcad/awesome-cad/main/README.md | No Inventor MCP entries. |
| glama.ai/mcp/servers/... (NeonGlay, cenitu, xtylerai, Andiveli, hsavas) | Seen via search results only; auto-scraped listings with possibly stale tool counts. |
| enterprisedna.co, claudemarket.ai, remoteopenclaw.com, mcprepository.com, mcpmarket.com | Seen in search results only; low-credibility aggregators. |

### Third-party articles (medium credibility; note conflicts)

| URL | Note |
|---|---|
| https://n23d.com/cad/openai-codex-cad-23x-inventor-2027-2-mcp-fusion-informed-design/ | Jessica Nicole, 2026-11-10. Trade news paraphrasing Autodesk's release; no direct testing. |
| https://borisinventor.com/our-blog/tpost/autodesk-inventor-ai-au-2026 | CADChain/BORIS, a vendor of an Inventor add-in, so there is a **conflict of interest**. Date inconsistency: dated 13 Oct but cites 3 Nov events. Mostly secondary citations. |
| https://vagon.io/blog/automate-autodesk-inventor-with-claude-code-on-vagon | Mert Boz. Cloud-PC vendor (**commercial interest**); recommends ipt-mcp; dates inconsistent. |
| https://moderncad.com.au/blog/inventor-ai-mcp-announcement/ | Nikita Gurov, 2026-04-12. **Vendor self-promotion.** |
| https://pierskitech.com/products/INVENTORMCP | Vendor page; claims are unverifiable. |
| https://forgemcp.ai/servers/autodesk/inventor | Vendor page; commercial. |
| forums.autodesk.com threads 14161719, 14113425, 14285262, 14336267; community blog 14104913 | **403. Snippets only.** User-generated, unverified. |
| Cadalyst AU 2026 blog, DEVELOP3D articles, certi.news, Leo AI blog (getleo.ai) | Seen only as search results or snippets. Leo AI is a competitor vendor and its "Inventor not supported" statement concerns Fusion's MCP. |
