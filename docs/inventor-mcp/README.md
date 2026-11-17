# Autodesk Inventor MCP — research and functional specification

*Prepared 2026-11-17 in the `freecad-mcp` repository, using this FreeCAD MCP as the reference implementation.*

## Executive summary

**Does an Inventor MCP already exist? Yes, but none is a production-ready modeling server.**

| What exists | Verdict | Evidence strength |
|---|---|---|
| **Autodesk's official Inventor MCP server** — ships with Inventor **2027.2+** (≈2026-11-03), 46 tools, stdio proxy `InvMcpStdio.exe` → Autodesk Assistant add-in | Good for documents, iProperties, parameters, Model States, iLogic rules, PDF/image export. **No sketch or feature tools** — modeling only by having the LLM write iLogic/Python. 2027.2+ only, sign-in required, one client per Inventor, likely beta. | Strong (exists; Autodesk docs verified twice) / Insufficient (readiness; 2 weeks old) |
| **~25 community projects**, best: [bimwright/ipt-mcp](https://github.com/bimwright/ipt-mcp) (Apache-2.0, C#, 150 tools, Inventor 2022–2027), [cenitu/inventor-mcp-server](https://github.com/cenitu/inventor-mcp-server) (MIT, Python, 65 tools), [tylerwarner33/autodesk-inventor-mcp](https://github.com/tylerwarner33/autodesk-inventor-mcp) (MIT, C#, 35 tools) | Useful references or fork bases; all single-maintainer, 0.x, unaudited, self-reported testing. | Moderate |
| **Commercial** (ModernCAD, AEC Pilot, ForgeMCP, Pierski) | Closed; claims unverifiable. | Insufficient |

**Recommendation.**
1. **Don't build from zero blind.** Run a short **Phase 0 evaluation** on a Windows machine with your Inventor version: Autodesk's server, ipt-mcp and cenitu against the 8 numeric acceptance tasks in [02 §12.3](02-functional-specification.md#123-acceptance-task-suite-numeric-oracles), plus the API verification harness.
2. If none clears the bar — the likely outcome for **typed, verifiable modeling** on pre-2027.2 Inventor — **build the modeling-first MCP specified here**: an in-process C# Inventor add-in (all API calls on Inventor's main thread) + a thin MCP server (Python, reusing this repo's patterns), ~31 typed core tools, stable geometry handles, numeric verification on every write, one undo step per call, gated code execution. Design it to **coexist** with Autodesk's server on 2027.2+.
3. Forking **ipt-mcp** (Apache-2.0) is the strongest shortcut if Phase 0 shows its modeling tools work well; this spec then serves as the gap list.

**Why typed modeling tools rather than "let the LLM write iLogic"?** Benchmarks consistently show LLM CAD code *compiles* (92–96%) but is *structurally wrong* often (48% of CADPrompt failures), image-to-CAD IoU is only 0.19–0.28 for frontier models, and Autodesk Research's own agent validated only 22.3% of designs on the first attempt. The failures are in structure, geometric references and units — exactly what typed tools with explicit units, stable handles and numeric verification address. LLMs do much better **editing** existing models (BenchCAD edit accuracy ≈0.85). See [01 §5](01-landscape-research.md#5-what-the-research-says-about-llms-doing-cad).

## Documents

| # | Document | What's in it |
|---|---|---|
| 01 | [Landscape research](01-landscape-research.md) | Autodesk official server (all 46 tools), community/commercial Inventor MCPs, lessons from ~30 CAD MCP servers (Fusion, Revit, SolidWorks, Blender, Rhino, …), LLM-CAD benchmark evidence, four-column evidence tables, bias self-audit |
| 02 | [Functional specification](02-functional-specification.md) | Scope, use cases, architecture & decisions, 81 functional and 15 non-functional requirements with acceptance criteria, data contracts (units, handles, envelopes, error codes), NFRs, threat model, packaging, test strategy, acceptance task suite, phases, risks, open decisions |
| 03 | [Tool catalog](03-tool-catalog.md) | Every MCP tool: parameters (with units), returns, annotations, phase, and the exact Inventor API call chain with verification tags; coverage comparison vs Autodesk's server |
| 04 | [Inventor API reference](04-inventor-api-reference.md) | Hosting models (in- vs out-of-process numbers), add-in skeleton, threading/transactions, units, enums, version drift table, translator GUIDs and options, reference keys, screenshots, iLogic, events, licensing, pitfalls, unverified list |
| 05 | [Lessons from FreeCAD MCP](05-lessons-from-freecad-mcp.md) | This repo's architecture and 18 design rules derived from bugs fixed in its git history |
| — | [`research/`](research/) | Raw research notes with every URL and a credibility note per source |

## Decisions needed from you

| ID | Question | Default if you don't decide |
|---|---|---|
| D-1 | Build, fork ipt-mcp, or adopt Autodesk + a community server? | Decide after Phase 0 |
| D-2 | MCP server in **Python** (reuse this repo) or **C#** (one toolchain, no Python on CAD PCs)? | Python |
| D-3 | Which Inventor versions must be supported? | 2025–2027 |
| D-4 | Allow a code-execution escape hatch (C#/iLogic)? | Yes, off by default, Phase 4 |
| D-5 | Internal tool, open source, or commercial? | Internal (Autodesk terms: "internal business use") pending legal review |

## How this research was done (and its limits)

- Three parallel research threads (existing Inventor MCPs; Inventor API; comparable CAD MCPs + LLM-CAD evidence), each told to run opposing queries, read source rather than READMEs, and tag every claim by evidence strength. ~60 repositories cloned and read; Autodesk help pages fetched directly; MCP Registry queried.
- The synthesis author independently re-verified the highest-impact claims: Autodesk's 46-tool list and "no modeling tools"; the 2027.2 What's New wording; ipt-mcp's and cenitu's tool counts in source; Ekins' 74.8 s vs 0.24 s timing; `ShellFeatures.CreateDefinition` (new in 2026); STL/STEP translator option values; three benchmark numbers in the paper bodies; one CVE.
- **Not done:** nothing was run against Inventor (no Windows/Inventor available); Autodesk forums and autodesk.com blogs returned HTTP 403 (seen only as search snippets); Autodesk's terms were read from a reseller-hosted 2023 copy. Every unverified API item is tagged `[U]` and listed in [04 §14](04-inventor-api-reference.md#14-unverified-items-to-close-in-phase-0) for Phase 0.
