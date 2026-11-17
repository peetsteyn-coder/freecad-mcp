# Inventor API Reference for the MCP

A curated subset of the Autodesk Inventor API that the [tool catalog](03-tool-catalog.md) depends on, with every item tagged by how it was verified. The full research notes (923 lines, ~390 tagged items, every source with a credibility note) are in [`research/B_inventor_api.md`](research/B_inventor_api.md).

**Nothing here was executed** — the research environment had no Windows/Inventor. "Verified" means checked against a written primary source. Items tagged `[U]` must be confirmed on a real Inventor in Phase 0 (02 §12.2) before code depends on them.

| Tag | Meaning |
|---|---|
| `[V:doc]` | Official Inventor API reference page — `https://help.autodesk.com/cloudhelp/<year>/ENU/Inventor-API/files/<Object>_<Member>.htm` (2025/2026/2027) |
| `[V:adn]` | Autodesk developer material (blog.autodesk.io incl. Brian Ekins' *Mod the Machine* archive; Autodesk-owned GitHub orgs; aps.autodesk.com) |
| `[V:interop]` | Present in a reflection dump of Autodesk's `Autodesk.Inventor.Interop.dll` for 2024–2027 ([InventorBridge `members.tsv`](https://github.com/gustacassel/InventorBridge)) |
| `[V:code]` | Seen in working community code / field notes — evidence a call shape works, not a spec |
| `[V:2nd]` | Weak secondary (news, reseller, search extract of a 403 page) |
| `[U]` | Unverified |

**Independently re-checked by the author of this document** (beyond the research agent): Ekins' in/out-of-process timings; `ShellFeatures.CreateDefinition` signature and "Introduced in version 2026"; STL `ExportUnits` and STEP `ApplicationProtocolType` values on TranslatorSettings; the 46-tool Autodesk MCP list.

---

## 1. Platform baseline

| Release | Shipped | Add-in runtime | Source |
|---|---|---|---|
| 2024 | 2023 | .NET Framework 4.8 | `[V:doc]` CreatingAnAddIn_Overview |
| 2025 | 2024 | **.NET 8** ("ported to .Net 8 from .Net Framework") | `[V:doc]` H25 WhatsNew |
| 2026 | 2025 | .NET 8 | `[V:adn]` |
| 2027 | Mar 2026 | **.NET 10** recommended | `[V:doc]` H27 WhatsNew |
| 2027.2 | 2026-11-03 | .NET 10; adds Autodesk's local MCP server | `[V:2nd]` date; server `[V:doc]` (Autodesk MCP help) |

- Windows x64 only. Registry-free `.addin` manifests required since 2024 `[V:adn]`.
- SDK: `C:\Users\Public\Documents\Autodesk\Inventor 20xx\SDK` (`developertools.msi`); interop at `<Inventor>\Bin\Public Assemblies\Autodesk.Inventor.interop.dll`; type library `RxInventor.tlb` in `Bin` `[V:adn]`.
- Apprentice Server (free, read-only geometry, RW properties) is a **separate installer since 2026**; not usable inside an add-in `[V:doc]`.
- Design Automation for Inventor (cloud): engines `Autodesk.Inventor+2025/2026/2027`, 2 Flex tokens per processing hour, file-in/file-out, minutes of latency `[V:adn]` — a possible future batch backend, not interactive.

## 2. Hosting models

| Criterion | Python pywin32 out-of-process | .NET EXE out-of-process | **In-process .NET add-in + IPC** (chosen) |
|---|---|---|---|
| Per-call overhead | High — 100k trivial calls 74.8 s (61.3 s with `UserInteractionDisabled`) | Same | **0.24 s** for the same 100k calls `[V:adn]` re-verified |
| Why | Calls are marshalled across processes and served only when Inventor's UI is idle | Same | "Inventor dedicates all of its processing to the API calls" |
| Modal dialogs | Call blocks until answered | Same | Same block, but the add-in can detect and refuse new work |
| `RPC_E_CALL_REJECTED` | Yes; pywin32 has no `CoRegisterMessageFilter` `[V:src]` | Yes; P/Invoke message filter | N/A (same apartment) |
| Version coupling | Low (late binding) | Interop per version | One build per .NET generation |
| Iteration speed | Fastest | Fast | Slow (restart Inventor to reload) |

Use pywin32 only for prototyping/probing in Phase 0 (cenitu's server is a good probe).

### 2.1 Add-in skeleton `[V:interop]` / `[V:doc]`
```csharp
public class McpAddInServer : Inventor.ApplicationAddInServer
{
    public void Activate(Inventor.ApplicationAddInSite site, bool firstTime) { /* site.Application */ }
    public void Deactivate() { /* release COM refs, GC.Collect() */ }
    public void ExecuteCommand(int commandID) { }
    public object Automation => null;
}
```
Manifest (`.addin`, registry-free): `<Addin Type="Standard">` with `ClassId`, `ClientId` (keep stable across releases), `DisplayName`, `Description`, `Assembly`, `LoadOnStartUp`, `UserUnloadable`, `SupportedSoftwareVersionGreaterThan` `[V:doc]`/`[V:adn]`. Locations: `%APPDATA%\Autodesk\Inventor 20xx\Addins\` (per user), `%ALLUSERSPROFILE%\Autodesk\Inventor Addins\` (all users) `[V:doc]`. 2027 adds `UseInventorAssemblyContext` for dependency isolation `[V:doc]`. .NET 8/10 add-ins need `<EnableDynamicLoading>true</EnableDynamicLoading>` and `<UseWindowsForms>` for the dispatcher control `[V:adn]`/`[V:code]`.

### 2.2 Main-thread dispatcher (no Autodesk-documented pattern; community-proven)
Create a hidden WinForms `Control` in `Activate` (runs on Inventor's STA thread), force its handle, run the pipe listener on a background thread that never touches the API, and post each request with `Control.BeginInvoke` `[V:code]` (bimwright/ipt-mcp ARCHITECTURE). Add the FreeCAD MCP semantics on top: per-request cancel-on-timeout, stuck detection, status served off-thread (see [05](05-lessons-from-freecad-mcp.md) L1–L4).

## 3. Threading, robustness, transactions

| API | Use | Tag |
|---|---|---|
| `Application.Ready` / `ApplicationEvents.OnReady` | Don't serve requests before start-up completes | `[V:doc]` |
| `Application.SilentOperation` (bool) | Suppress prompts around open/save/close/export; restore afterwards | `[V:doc]` |
| ↳ limitation | Save still fails while in sketch / flat-pattern / in-place edit → exit edit first (`ActivatedObject.ExitEdit()` or `AppReturnTopCmd`) | `[V:adn]` |
| `Application.UserInterfaceManager.UserInteractionDisabled` | Lock UI during a long op — **on `UserInterfaceManager`, not `Application`**; always reset in `finally` | `[V:doc]` |
| `Application.ScreenUpdating` | Stop redraws during batches | `[V:doc]` |
| `PlanarSketch.DeferUpdates` | 500 sketch lines: 18.5 s → 1.6 s | `[V:interop]` + `[V:adn]` |
| `Application.AssemblyOptions.DeferUpdate` | 500 constrained occurrences: 699 s → 54 s | `[V:interop]` + `[V:adn]` |
| `TransactionManager.StartTransaction(Document, DisplayName) As Transaction` | One undo step per tool call | `[V:doc]` |
| `Transaction.End()` / `Transaction.Abort()` | Commit / roll back (fail if not current) | `[V:doc]` |
| `TransactionManager.UndoTransaction()`, `CommittedTransactions`, `TransactionEvents` (`OnCommit/OnAbort/OnUndo/OnRedo`) | `undo` tool; detect user edits | `[V:doc]` |
| `Application.ErrorManager` | Human-readable reason after a failed call | `[V:interop]` + `[V:code]` |
| `Application.MainFrameHWND` | Detect a modal dialog (main window disabled) via Win32, without calling Inventor | `[V:interop]` + `[V:code]` |
| HRESULTs | `RPC_E_CALL_REJECTED 0x80010001`, `RPC_E_SERVERCALL_RETRYLATER 0x8001010A` (out-of-process busy), `RPC_E_DISCONNECTED 0x80010108` (stale object after undo/close/delete) | `[V:code]` |

## 4. Units

| Fact | Tag |
|---|---|
| Database length unit = **centimetres**; "All API functions use the database units … as input and output" | `[V:doc]` UOM_Overview |
| Angles = **radians** (e.g. `AddAngleConstraint`, hole `BottomTipAngle`) | `[V:doc]` |
| Mass = kilograms | `[U]` (consistent with community notes) |
| `Variant` dimension args accept a **number** (database units) or a **string** (`"10 mm"`, `"Thickness*2"`) — strings create named parameters; strings without units use document units | `[V:doc]` |
| `Document.UnitsOfMeasure`: `GetValueFromExpression(expr, units)`, `ConvertUnits(v, in, out)`, `GetStringFromValue`, `IsExpressionValid`, `GetLocaleCorrectedExpression` ("1.0 in" → "1,0 in" in German), `LengthUnits`, `AngleUnits` | `[V:doc]` |
| `SketchCircles.AddByCenterRadius` radius and all `TransientGeometry` coordinates are cm | `[V:doc]` |

**Rule for the add-in:** convert mm→cm and deg→rad in exactly one module; pass user-supplied expressions through as strings.

## 5. Enumerations used by the catalog `[V:doc]` (2026 enum pages)

| Enum | Values |
|---|---|
| `DocumentTypeEnum` | 12290 part · 12291 assembly · 12292 drawing · 12293 presentation · 12289 unknown |
| `PartFeatureOperationEnum` | 20481 join · 20482 cut · 20483 intersect · 20484 surface · 20485 new body |
| `PartFeatureExtentDirectionEnum` | 20993 positive · 20994 negative · 20995 symmetric |
| `DimensionOrientationEnum` | 19201 horizontal · 19202 vertical · 19203 aligned |
| `AssemblyJointTypeEnum` | 102401 rigid · 102402 rotational · 102403 slide · 102404 cylindrical · 102405 planar · 102406 ball |
| `DrawingViewStyleEnum` | 32257 hidden line · 32258 hidden line removed · 32259 shaded · 32260 from base · 32261 shaded hidden line |
| `ViewOrientationTypeEnum` | 10753 default · 10754 top · 10755 right · 10756 back · 10757 bottom · 10758 left · 10759 iso top-right · 10760 iso top-left · 10761 iso bottom-right · 10762 iso bottom-left · 10763 arbitrary · 10764 front · 10765 current |
| `DimensionTypeEnum` | 60161 aligned · 60162 horizontal · 60163 vertical · 60166 diametric · 60169 angular |
| `ShellDirectionEnum` / `ShellMethodEnum` | 41217 inside (default) / 130305 sharp (default) |
| Sheet-metal subtype GUID | `{9C464203-9BAE-11D3-8BAD-0060B0CE6BB4}` |
| `HealthStatusEnum` | 11778 up to date … 11781 in error — `[U]`; read from type library |

**Do not hard-code enums from community code.** Load them from the installed type library at start-up (`[V:adn]` approach; one community MCP already does this).

## 6. Version drift (renamed or hidden members)

Hidden members usually remain callable late-bound but vanish from docs and IntelliSense. The add-in must pick per capability map.

| Operation | ≤2024 | 2025 | 2026 | 2027 | Tag |
|---|---|---|---|---|---|
| Shell definition | `CreateShellDefinition` | `CreateShellDefinition` | **`CreateDefinition`** | `CreateDefinition` | `[V:doc]` re-verified |
| Flange definition | `CreateFlangeDefinition` | **`CreateDefinition`** | `CreateDefinition` | `CreateDefinition` | `[V:doc]` + `[V:interop]` |
| Contour flange | `CreateContourFlangeDefinition` | same | **`CreateDefinition`** | same | `[V:interop]` |
| Drilled hole by distance | `AddDrilledByDistanceExtent` | **`…Extent2`** (+ drill point types) | `…2` | `…2` | `[V:doc]` |
| Hole linear placement | `CreateLinearPlacementDefinition` | hidden (callable) | hidden | hidden | `[V:interop]` + `[V:code]` |
| Retrieve model dims | `GeneralDimensions.Retrieve` | `Sheet.RetrieveAnnotations` | `RetrieveAnnotations2` (2026.1) | `…2` | `[V:doc]` |
| 3-point arc (transient) | `CreateArc2dByThreePoints` | `…2` (2025.1) | `…2` | `…2` | `[V:doc]` |
| Drawing breaks | `BreakOperations.Add` | same | same | `AddWithOptions` | `[V:doc]` |
| Slot features | — | — | — | `SlotFeatures` (new) | `[V:interop]` |
| Camera image with options | — | — | `SaveAsBitmapWithOptions` (2026.1) | same | `[V:doc]` |

## 7. Translator add-ins (export/import)

Pattern `[V:doc]` (TranslatorAddIn_SaveCopyAs, Export-to-STEP sample):
```text
addin = app.ApplicationAddIns.ItemById("{GUID}")   ; Activate() if not Activated
ctx   = app.TransientObjects.CreateTranslationContext(); ctx.Type = kFileBrowseIOMechanism
opts  = app.TransientObjects.CreateNameValueMap()
if addin.HasSaveCopyAsOptions(doc, ctx, opts): opts.Value("…") = …     ; out-of-process use opts.Add(name, value)
med   = app.TransientObjects.CreateDataMedium(); med.FileName = path
addin.SaveCopyAs(doc, ctx, opts, med)                                  ; 2025+: source may be a FlatPattern
```

| Format | GUID | Cross-check |
|---|---|---|
| STEP | `{90AF7F40-0C01-11D5-8E83-0010B541CD80}` | `[V:doc]` sample + `[V:adn]` |
| IGES | `{90AF7F44-0C01-11D5-8E83-0010B541CD80}` | `[V:code]` |
| STL (export) | `{533E9A98-FC3B-11D4-8E7E-0010B541CD80}` | `[V:code]` (3 repos) |
| DWG | `{C24E3AC2-122E-11D5-8E91-0010B541CD80}` | `[V:adn]` |
| DXF | `{C24E3AC4-122E-11D5-8E91-0010B541CD80}` | `[V:code]` |
| PDF (2D) | `{0AC6FD96-2F4D-42CE-8BE0-8AEA580399E4}` | `[V:adn]` |
| SAT | `{89162634-02B6-11D5-8E80-0010B541CD80}` | `[V:adn]` |
| Parasolid binary / text | `{A8F8F8E5-BBAB-4F74-8B1B-AC011251F8AC}` / `{8F9D3571-3CB8-42F7-8AFF-2DB2779C8465}` | `[V:code]` |
| JT | `{16625A0E-F58C-4488-A969-E7EC4F99CACD}` | `[V:code]` |
| OBJ export | `{F539FB09-FC01-4260-A429-1818B14D6BAC}` | `[V:code]` |

Options (TranslatorSettings `[V:doc]`, re-verified):
- **STEP** `ApplicationProtocolType`: 2 = AP203, **4 = AP214IS (default)**, 5 = AP242. ⚠ Autodesk's STEP *sample* says 3 = AP214 — verify by reading `FILE_SCHEMA` in the output header.
- **STL** `ExportUnits`: 2 in, 3 ft, **4 cm (default)**, **5 mm**, 6 m, 7 micron. ⚠ A community MCP sets 4 believing it is mm. `OutputFileType` 0 binary / 1 ASCII. `Resolution` 0 high / 1 medium / 2 low / 3 custom.
- **2D PDF/DWG/DXF** option names: `[U]` (DWG/DXF usually driven by an exported `.ini`).
- **Sheet-metal flat pattern DXF:** `SheetMetalComponentDefinition.DataIO.WriteDataToFile("FLAT PATTERN DXF?AcadVersion=R12&OuterProfileLayer=Outer&InvisibleLayers=…", path)` `[V:doc]`.

## 8. Geometry references (the "fillet that edge" problem)

- B-Rep objects (`Face`, `Edge`, `Vertex`) **die on any model change**; indices renumber after every feature `[V:adn]`/`[V:code]`.
- **Reference keys** (`[V:doc]` ReferenceKeyManager, Face_GetReferenceKey; `[V:adn]` Ekins "Understanding Reference Keys"):
  `ReferenceKeyManager.CreateKeyContext()` → `Face.GetReferenceKey(ByRef key, KeyContext)` → `KeyToString` / `StringToKey` → `SaveContextToArray` (persist context with keys) → `LoadContextFromArray` → `CanBindKeyToObject` → `BindKeyToObject(key, context, [out MatchType])`.
  Rules: keys are document-specific; never compare key bytes; **binding may return an `ObjectCollection` when a face was split** (first = best match); binding fails if the entity is gone.
  Marshalling `ByRef Byte()` through pywin32 is `[U]` → do reference-key work **in the add-in** (C#).
- **Transient keys** (`Face.TransientKey`, `SurfaceBody.BindTransientKeyToObject`) — cheap, valid only until the next change `[V:adn]`.
- **Attributes** (`AttributeSets.Add`, `AttributeManager.FindObjects`) and **iLogic named entities** (`iLogicEntityNameSet` / `GetNamedEntities(doc).SetName/TryGetEntity`) persist in the file `[V:doc]`/`[V:2nd]` — use only when the user wants names saved.
- **Proxies:** assembly constraints/joints need `ComponentOccurrence.CreateGeometryProxy(Geometry, out Result)` — geometry "as seen through this occurrence" `[V:doc]`/`[V:adn]`.

## 9. Screenshots

`Application.ActiveView.Camera` → set `ViewOrientationType` → `Fit()` → `ApplyWithoutTransition()` → **`View.Update()`** (Apply alone didn't redraw `[V:code]`) → `Camera.SaveAsBitmap(path, w, h, [top], [bottom])` (format by extension) `[V:doc]`; 2026.1+ `SaveAsBitmapWithOptions` (transparent background) `[V:doc]`. Behavior when Inventor is minimized/occluded: `[U]` → Phase 0 test + blank-image detection (FR-VIS-05). Invisible documents have no view `[U]`.

## 10. iLogic

- Add-in ClientId `{3BDD8D79-2179-4B11-8A5A-257B1C0263AC}` → `ApplicationAddIns.ItemById(...)` → `Activate()` → `.Automation` (`IiLogicAutomation`) `[V:adn]`.
- `RunRule(doc, name)`, `RunExternalRule(doc, name)`, `RunRuleWithArguments(doc, name, NameValueMap)`, `RunRuleDirect…`, `GetNamedEntities(doc)` `[V:doc]` (iLogic API 2021).
- ⚠ `AddRule(doc, name, text)` "will be compiled and run" **and persists in the document** `[V:doc]`.
- ⚠ 2025+: external .NET Framework hosts saw `RunRule` failures; external EXEs using iLogic must be .NET 8 `[V:adn]`/`[V:2nd]`.
- On 2027.2+, prefer Autodesk's MCP server for rule management (it has 8 iLogic tools).

## 11. Events worth subscribing to

`ApplicationEvents`: `OnActivateDocument`, `OnCloseDocument`, `OnDocumentChange`, `OnSaveDocument`, `OnActiveProjectChanged`, `OnReady`; `ModelingEvents`: `OnNewFeature`, `OnFeatureChange`, `OnDelete`, `OnParameterChange`; `TransactionManager.TransactionEvents`: `OnCommit`, `OnAbort`, `OnUndo`, `OnRedo` (there is **no** "UndoEvents" object) `[V:doc]`. Use them to invalidate handles and tell the client about user edits (FR-TRX-04).

## 12. Licensing and terms (needs legal review)

From a reseller-hosted copy of Autodesk's Terms of Use / AUP (effective 2023-05-15; live pages returned 403) `[V:2nd]`:
- APIs may be used "solely for Your internal business use".
- AUP: no use of offerings **or their output** to train ML/AI systems → MCP logs must not feed training.
- Single-user Autodesk IDs may not be shared → no multi-user "Inventor as a service" on desktop licences.
- Autodesk's own 2027.2 local MCP shows that an external agent driving the signed-in user's own session is a sanctioned pattern.
- Redistribution of `Autodesk.Inventor.Interop.dll` (two community repos vendor it): **unverified** — do not redistribute.

## 13. Pitfalls checklist (top 20)

1. Renamed/hidden members per release → capability map from the type library (§6).
2. cm/rad database units; numbers vs strings in `Variant` args (§4).
3. Out-of-process calls ~300× slower and served only when the UI is idle (§2).
4. `UserInteractionDisabled` lives on `UserInterfaceManager`; always reset.
5. Modal dialogs block calls indefinitely → `SilentOperation` + detection + `HOST_BUSY`.
6. `SilentOperation` doesn't rescue saves during sketch/flat-pattern/in-place edit.
7. Never blindly retry writes after `RPC_E_CALL_REJECTED` or a timeout.
8. B-Rep references die after any change → reference keys + handle re-binding; handle split results.
9. Assembly constraints need geometry proxies.
10. `ComponentOccurrences.Add` needs a saved file on disk.
11. `Documents.Add` may not activate the new document → `Activate()` and track by handle.
12. pywin32: `CastTo`/`EnsureDispatch` silently switch to early binding and can corrupt `gen_py`; `constants` is empty without makepy.
13. STEP AP numbering inconsistent in Autodesk's own pages → check output header.
14. STL default unit is cm (4); mm is 5.
15. Thread designations must match the installed thread table exactly.
16. `AddDetailView` and `PartsLists.Add` require the sheet to be active.
17. `GetTemplateFile` can return "" → fall back to `FileOptions.TemplatesPath` (there is no `Application.FileLocations`).
18. Saving assemblies can rewrite dirty library/Content Center files → `Save2(False)`.
19. Localized names (origin planes `'XY 平面'`, BOM view names) → use indices/GUIDs/types.
20. iLogic `AddRule` runs and persists code; generated rules can attach to events.

## 14. Unverified items to close in Phase 0

Mass unit text (kg); `HealthStatusEnum` values; `DisplayModeEnum` values; `PointIntentEnum` / geometry-intent values; `Face.CreatedByFeature`; `SketchEntity.Construction`; `PartFeature.Suppressed`/`Delete`; `MeasureTools.GetAngle`; `Application.SoftwareVersion` member path; reference-key marshalling out-of-process; screenshot behavior when minimized; whether `ControlDefinition.Execute` of modeling commands blocks; 2D PDF/DWG/DXF translator option names; Pack and Go member names; Inventor LT / Nastran / Tube & Pipe / Cable & Harness API availability; pythonnet/comtypes with Inventor; out-of-process event pumping.
