# Tool Catalog — Inventor Modeling MCP

Companion to [02-functional-specification.md](02-functional-specification.md). Each tool lists its parameters (with units), its return value, its MCP annotations, the phase it ships in, and the **Inventor API call chain** the add-in uses, with the verification tag from [04-inventor-api-reference.md](04-inventor-api-reference.md):
`[V:doc]` official API reference · `[V:adn]` Autodesk developer material · `[V:interop]` interop-DLL member list · `[V:code]` community working code · `[U]` unverified — **must be confirmed in Phase 0 before implementation**.

**Conventions for every tool**
- Lengths in **mm**, angles in **degrees**, mass in **kg**; parameter *expressions* are strings passed to Inventor verbatim (`"120 mm"`, `"Width/2"`).
- `doc` is optional everywhere; default is the active document.
- Mutating tools return the envelope in 02 §7.3 (`result`, `verify`, `undo`, `warnings`) and accept `include_view: bool = false` and `view: ViewPreset = "iso"`.
- Errors use the codes in 02 §7.4, with `isError: true`.
- Annotations: **RO** = `readOnlyHint`; **D** = `destructiveHint`; **I** = `idempotentHint`.
- `ViewPreset` = `iso | front | back | top | bottom | left | right`.

---

## Toolset `core` — enabled by default (31 tools)

### Session

#### `get_status` — RO · I · P1
Health and context. **Served off Inventor's main thread** (works while Inventor is busy).
| Param | Type | Notes |
|---|---|---|
| `detail` | bool = false | include capability map |

Returns: `{inventor: {version, build, pid, ready}, dispatcher: {state: idle|busy|stuck, operation, running_s}, modal_dialog: {open, title}, active_document, capabilities?, toolsets, read_only}`
API: `Application.SoftwareVersion` `[U]` (member name to confirm), `Application.Ready` `[V:doc]`; dialog detection via `Application.MainFrameHWND` enabled state + Win32 `EnumWindows` (no Inventor call) `[V:interop]`/`[V:code]`; capabilities from type-library reflection at add-in start.

#### `list_documents` — RO · I · P1
Returns `[{handle, name, full_path, type: part|sheet_metal|assembly|drawing|presentation, dirty, active}]`.
API: `Application.Documents` → `Document.FullFileName`, `DocumentType` (`DocumentTypeEnum` 12290 part, 12291 assembly, 12292 drawing, 12293 presentation) `[V:doc]`, `SubType` (sheet-metal GUID `{9C464203-9BAE-11D3-8BAD-0060B0CE6BB4}`) `[V:doc]`, `Dirty` `[V:interop]`.

### Documents

#### `new_document` — write · P1
| Param | Type | Notes |
|---|---|---|
| `type` | `part \| sheet_metal \| assembly \| drawing` | required |
| `template` | string? | name or allow-listed path; default = Inventor default for the type |
| `units` | `metric \| imperial` = metric | selects template system of measure |
| `name` | string? | display name; actual name is returned |

Returns `{doc, name, full_path: null|string, template_used}`.
API: `FileManager.GetTemplateFile(DocumentType, [SystemOfMeasure], [DraftingStandard], [SubType])` `[V:doc]` → fallback: glob `Application.FileOptions.TemplatesPath` `[V:interop]` (`GetTemplateFile` can return "" `[V:code]`) → `Documents.Add(DocumentType, Template, CreateVisible=True)` `[V:doc]` → `doc.Activate()` (Add may not activate `[V:code]`). Sheet metal: set `PartDocument.SubType` to the sheet-metal GUID `[V:doc]`.
Errors: `TEMPLATE_NOT_FOUND`.

#### `open_document` — write · P1
| Param | Type | Notes |
|---|---|---|
| `path` | string | allow-listed; `.ipt/.iam/.idw/.dwg/.ipn` or importable (`.stp/.step/.igs/.sat`) |
| `visible` | bool = true | invisible docs have no view → no screenshots |

API: `Documents.Open(FullDocumentName, OpenVisible)` / `OpenWithOptions(name, NameValueMap, visible)` `[V:interop]` inside `Application.SilentOperation = True` `[V:doc]` (restored after). STEP opens import through the translator `[V:code]`.
Errors: `PATH_NOT_ALLOWED`, `FILE_NOT_FOUND`.

#### `save_document` — write · P1
| Param | Type | Notes |
|---|---|---|
| `path` | string? | save-as target (allow-listed); omitted = save in place |
| `copy` | bool = false | save-copy-as |
| `overwrite` | bool = false | |
| `include_dependents` | bool = false | assemblies: false → `Save2(False)` |

API: exit active edit first (`doc.ActivatedObject.ExitEdit()` or `ControlDefinitions.Item("AppReturnTopCmd").Execute()`) `[V:adn]`; `Document.Save2(SaveDependents, [DocumentsToSave])` `[V:interop]` / `Document.SaveAs(FileName, SaveCopyAs)` `[V:doc]` under `SilentOperation` `[V:doc]`. `SaveAs` fails if a doc with that name is open `[V:code]`.
Errors: `IN_EDIT_MODE`, `FILE_EXISTS`, `PATH_NOT_ALLOWED`.

#### `close_document` — write · **D** · P1
| Param | Type | Notes |
|---|---|---|
| `save` | bool = false | if false and dirty → requires `discard_changes=true` |
| `discard_changes` | bool = false | |

API: `Document.Close(SkipSave)` `[V:interop]`.

### Inspection

#### `get_model_tree` — RO · I · P1
| Param | Type | Notes |
|---|---|---|
| `depth` | int = 2 | assemblies: occurrence depth |
| `include` | list = all | `features, sketches, work_features, bodies, parameters, occurrences` |
| `page` | int = 0 | output capped at 32 KB per page |

Returns ordered features `{handle, name, type, suppressed, health, health_message, driving_parameters[]}`, sketches `{handle, name, plane, profile_closed}`, bodies `{handle, name, volume_mm3, faces}`, occurrences `{handle, name, file, grounded, suppressed, transform_mm_deg}`.
API: `PartComponentDefinition.Features` (iterate `PartFeatures`) `[V:doc]`, `Sketches`, `WorkPlanes/WorkAxes/WorkPoints`, `SurfaceBodies` `[V:interop]`; feature health via `PartFeature.HealthStatus` → `HealthStatusEnum` (11778 up-to-date … 11781 in-error per community) `[U]` officially — read enum from type library; error text `Application.ErrorManager` `[V:interop]`; occurrences `AssemblyComponentDefinition.Occurrences` → `Transformation` (Matrix, cm) `[V:interop]`.

#### `describe_geometry` — RO · P2
The model's way of "seeing" faces and edges. Returns handles + descriptors.
| Param | Type | Notes |
|---|---|---|
| `source` | `body \| feature \| selection` = body | `selection` reads the user's current selection |
| `target` | handle? | body or feature handle |
| `kind` | `faces \| edges \| vertices` = faces | |
| `filter` | object? | `{surface_type: plane\|cylinder\|cone\|torus\|sphere\|bspline, curve_type: line\|circle\|arc\|ellipse\|bspline, normal: [x,y,z], direction: [x,y,z], tol_deg: 2, near_point_mm: [x,y,z], radius_mm: {min,max}, created_by: feature handle, largest: n, smallest: n}` |
| `limit` | int = 50 | |

Returns `[{handle: "face:F7@…", label: "F7", type, normal|axis, centroid_mm|midpoint_mm, radius_mm?, area_mm2|length_mm, created_by, adjacent: [labels]}]`.
API: `SurfaceBody.Faces/Edges/Vertices` `[V:interop]`; `Face.SurfaceType`, `Edge.GeometryType`, `Face.Geometry`, evaluators `[V:interop]`; owning feature via `Face.CreatedByFeature` `[U]`; stable handle = `Face.GetReferenceKey(key, KeyContext)` + `ReferenceKeyManager.KeyToString` under a per-document `CreateKeyContext()` kept by the add-in `[V:doc]`/`[V:adn]` (Ekins); selection via `Document.SelectSet` `[V:doc]`.
Notes: transient keys are cheaper but die on any model change `[V:adn]`; never select by screen coordinates (anti-pattern A10).

#### `measure` — RO · I · P2
| Param | Type | Notes |
|---|---|---|
| `a`, `b` | handle | faces, edges, vertices, work features, occurrences |
| `kind` | `min_distance \| angle` = min_distance | |

Returns `{value_mm | value_deg, points_mm?}`.
API: `MeasureTools.GetMinimumDistance(EntityOne, EntityTwo, ...)` `[V:interop]` (returns cm; tuple out-of-process `[V:code]`); `MeasureTools.GetAngle` `[U]`.

#### `get_mass_properties` — RO · P1
Returns `{mass_kg, volume_mm3, area_mm2, center_of_mass_mm, bbox_mm: {min, max}, material, accuracy}`.
API: `doc.Update()` then `PartComponentDefinition.MassProperties` (`Mass`, `Volume`, `Area`, `CenterOfMass`, `Accuracy`, `CacheResultsOnCompute=False` for assemblies) `[V:doc]`; `ComponentDefinition.RangeBox` `[V:interop]` (loose on B-spline faces `[V:code]`). Mass unit kg `[U]` (consistent with community notes).

### Sketching

#### `create_sketch` — write · P2
| Param | Type | Notes |
|---|---|---|
| `on` | `XY \| XZ \| YZ \| <work plane name> \| <face handle>` | required |
| `offset_mm` | number = 0 | creates a work plane offset if ≠ 0 |
| `name` | string? | |

Returns `{sketch, name, origin_mm, x_axis, y_axis, normal}` (sketch-space → model-space mapping).
API: origin planes `WorkPlanes.Item(1)=YZ, (2)=XZ, (3)=XY` `[V:adn]` (by index, never by localized name); offset `WorkPlanes.AddByPlaneAndOffset(Plane, Offset)` `[V:interop]`; `PlanarSketches.Add(PlanarEntity, [UseFaceEdges])` `[V:doc]`; mapping via `PlanarSketch.SketchToModelSpace(Point2d)` `[V:interop]` (on XZ, sketch X maps to world −X `[V:code]`).

#### `add_sketch_geometry` — write · P2
Adds many entities in one transaction.
| Param | Type | Notes |
|---|---|---|
| `sketch` | handle | |
| `entities` | list | each one of: `line{p1,p2}`, `polyline{points, closed}`, `rectangle{p1,p2}` / `rectangle{center,width,height}`, `circle{center,radius}`, `arc{center,start,end,ccw}` / `arc{p1,mid,p2}`, `polygon{center,radius,sides,inscribed}`, `slot{p1,p2,width}`, `point{p, hole_center}`, `spline{points}`; all mm in sketch space; `construction: bool` |

Returns `{created: [{index, type, handle}], profiles: [{index, closed, area_mm2}]}`.
API: `PlanarSketch.DeferUpdates = True` during batch `[V:interop]`; `TransientGeometry.CreatePoint2d(x_cm, y_cm)` `[V:doc]`; `SketchLines.AddByTwoPoints` `[V:interop]`, `SketchLines.AddAsTwoPointRectangle(p1, p2)` `[V:doc]`, `SketchCircles.AddByCenterRadius(center, radius_cm)` `[V:doc]`, `SketchArcs.AddByCenterStartEndPoint(center, start, end, ccw)` / `AddByThreePoints` `[V:interop]`, `SketchPoints.Add(Point2d, HoleCenter)` `[V:interop]`, `SketchSplines.Add(ObjectCollection)` `[V:interop]`; polygon/slot composed from lines/arcs + constraints. Construction flag via `SketchEntity.Construction` `[U]`.

#### `get_sketch` — RO · P2
Returns entities, constraints count, profiles found (`Profiles.AddForSolid` dry-run inside an aborted transaction) with `closed` flags and open-end locations.
API: `PlanarSketch.SketchEntities` `[U]`, `Profiles.AddForSolid([Combine], …)` `[V:doc]` (open sketch → bare `E_INVALIDARG` `[V:code]` → mapped to `PROFILE_NOT_CLOSED`).

### Part features

All feature tools: return `{feature, name, health}` + `verify` (volume, bodies, faces, bbox before/after). A feature that computes with errors is aborted → `FEATURE_FAILED` with `ErrorManager` text.

#### `extrude` — write · P2
| Param | Type | Notes |
|---|---|---|
| `sketch` | handle | |
| `profile` | int? | profile index from `get_sketch`; default = all closed regions combined |
| `extent` | `distance \| two_distances \| symmetric \| through_all \| to_face` | |
| `distance` | number (mm) \| expression | for distance/symmetric |
| `distance2` | number \| expression | for two_distances |
| `to` | face handle | for to_face |
| `direction` | `positive \| negative` = positive | relative to sketch normal |
| `operation` | `new_body \| join \| cut \| intersect` = join (new_body if no body) | |
| `taper_deg` | number = 0 | |
| `name` | string? | |

API: `Profiles.AddForSolid()` `[V:doc]` → `ExtrudeFeatures.CreateExtrudeDefinition(Profile, PartFeatureOperationEnum)` `[V:doc]` → `ExtrudeDefinition.SetDistanceExtent(Distance, PartFeatureExtentDirectionEnum)` `[V:doc]` | `SetThroughAllExtent(dir)` | `SetToExtent(ToEntity, ExtendToFace)` | `SetDistanceExtentTwo(Distance)` `[V:interop]` | `TaperAngle` `[V:doc]` → `ExtrudeFeatures.Add(def)` `[V:interop]`.
Enums `[V:doc]`: operation join 20481, cut 20482, intersect 20483, new body 20485; direction positive 20993, negative 20994, symmetric 20995.
Note: number → cm conversion; expression strings create named parameters `[V:doc]`.

#### `revolve` — write · P2
| Param | Type | Notes |
|---|---|---|
| `sketch`, `profile` | handle, int? | |
| `axis` | handle | sketch line, work axis or linear edge |
| `angle_deg` | number? | omitted = full 360° |
| `direction` | `positive \| negative \| symmetric` | |
| `operation` | as extrude | |

API: `RevolveFeatures.AddFull(Profile, AxisEntity, Operation)` `[V:doc]` / `AddByAngle(Profile, AxisEntity, Angle_rad, ExtentDirection, Operation)` `[V:interop]`.

#### `hole` — write · P2
| Param | Type | Notes |
|---|---|---|
| `placement` | `{sketch_points: [handles] }` \| `{face, edge1, offset1_mm, edge2, offset2_mm}` \| `{point_mm, direction}` | |
| `type` | `drilled \| counterbore \| countersink \| spotface` = drilled | |
| `diameter_mm` | number? | xor `thread` |
| `thread` | `{standard, designation, class, right_handed=true, full_depth=true}` | validated against installed thread table |
| `depth` | `{distance_mm} \| through_all` | |
| `drill_point_deg` | number = 118 | |
| `cbore` / `csink` | `{diameter_mm, depth_mm}` / `{diameter_mm, angle_deg}` | |

API: `HoleFeatures.CreateSketchPlacementDefinition(ObjectCollection of SketchPoints)` `[V:doc]` (sketch should contain only points `[V:code]`) | `CreatePointPlacementDefinition(Point, Direction)` `[V:interop]` | linear placement `CreateLinearPlacementDefinition(...)` hidden after 2024 but callable late-bound `[V:code]` → `AddDrilledByThroughAllExtent(Placement, DiameterOrTapInfo, Direction)` `[V:doc]` / `AddDrilledByDistanceExtent2(Placement, DiameterOrTapInfo, Depth, Direction, [DrillPointType], [BottomTipAngle])` `[V:doc]` (2025+; legacy non-`2` name on ≤2024) + CBore/CSink variants `[V:doc]`; taps `HoleFeatures.CreateTapInfo(RightHanded, ThreadType, Designation, Class, FullTapDepth, [ThreadDepth])` `[V:doc]` — `ThreadType` must equal a thread-table sheet name exactly (e.g. "ISO Metric profile") `[V:code]`.
Verify: volume must decrease; if not, flip direction once and re-check `[V:code]`.

#### `fillet` — write · P2
| Param | Type | Notes |
|---|---|---|
| `edges` | handle[] | |
| `radius` | number (mm) \| expression | |
| `tangent_chain` | bool = true | |

API: `TransientObjects.CreateEdgeCollection()` `[V:interop]` (must be EdgeCollection, not ObjectCollection) → `FilletFeatures.AddSimple(Edges, Radius, [AllFillets], [AllRounds], [AutomaticEdgeChain], …)` `[V:doc]`.
Tip returned to the model: fillet large radii first; edge handles re-bind after each feature.

#### `chamfer` — write · P2
| Param | Type | Notes |
|---|---|---|
| `edges` | handle[] | |
| `mode` | `distance \| distance_angle \| two_distances` | |
| `distance_mm`, `distance2_mm`, `angle_deg` | number | |
| `face` | handle? | reference face for asymmetric modes |

API: `ChamferFeatures.AddUsingDistance(Edges, Distance, …)` `[V:doc]`, `AddUsingDistanceAndAngle(Edges, Face, Distance, Angle, …)`, `AddUsingTwoDistances(Edges, Face, D1, D2, …)` `[V:interop]`.

#### `shell` — write · P2
| Param | Type | Notes |
|---|---|---|
| `remove_faces` | handle[] | faces left open |
| `thickness_mm` | number | |
| `direction` | `inside \| outside \| both` = inside | |

API (capability-switched): **2026+** `ShellFeatures.CreateDefinition([InputFaces], [Solids], [Thickness], [Direction=kInsideShellDirection 41217], [Method=kSharpShellMethod 130305], [MoreOptions])` → `ShellFeatures.Add(def)` `[V:doc]` (verified: "Introduced in version 2026"); **≤2025** `ShellFeatures.CreateShellDefinition([InputFaces], [Thickness], [Direction])` → `Add` `[V:interop]`.

#### `pattern` — write · P2
| Param | Type | Notes |
|---|---|---|
| `kind` | `rectangular \| circular \| mirror` | |
| `features` | handle[] | parent features (fillets/chamfers only with their parent) |
| `rectangular` | `{direction1: handle, count1, spacing1_mm, direction2?, count2?, spacing2_mm?}` | |
| `circular` | `{axis: handle, count, angle_deg = 360, fit_within_angle = true}` | |
| `mirror` | `{plane: handle}` | |

API: `RectangularPatternFeatures.CreateDefinition(ParentFeatures, XDirectionEntity, NaturalXDirection, XCount, XSpacing, [XSpacingType])` `[V:doc]` → `AddByDefinition` `[V:interop]`; `CircularPatternFeatures.CreateDefinition(ParentFeatures, AxisEntity, NaturalAxisDirection, Count, Angle, [FitWithinAngle])` `[V:doc]`; `MirrorFeatures.CreateDefinition(ParentFeatures, MirrorPlaneEntity, [ComputeType])` `[V:doc]`.

#### `create_work_feature` — write · P2
| Param | Type | Notes |
|---|---|---|
| `kind` | `plane \| axis \| point` | |
| `plane` | `{offset_from: handle\|XY\|XZ\|YZ, offset_mm}` \| `{origin_mm, x_axis, y_axis}` | |
| `axis` | `{through: edge/cylinder handle}` \| `{point_mm, direction}` | |
| `point` | `{at_mm}` \| `{vertex: handle}` | |
| `grounded` | bool = true | assembly-level work features need grounding `[V:code]` |

API: `WorkPlanes.AddByPlaneAndOffset(Plane, Offset)`, `WorkPlanes.AddFixed(OriginPoint, XAxis, YAxis)`, `WorkAxes.AddFixed(Point, Axis)`, `WorkPoints.AddFixed(Point)` `[V:interop]`; other `AddBy…` variants `[U]`.

#### `edit_feature` — write · D (delete) · P2
| Param | Type | Notes |
|---|---|---|
| `feature` | handle | |
| `action` | `rename \| suppress \| unsuppress \| delete \| set_parameter` | |
| `new_name` / `parameter` + `expression` | | |

API: `PartFeature.Name`, `Suppressed` `[U]` (member names to confirm), `PartFeature.Delete()` `[U]`; parameter edits via `Parameter.Expression` `[V:interop]` then `doc.Update()` `[V:doc]`.

### Parameters and properties

#### `get_parameters` — RO · I · P1
| Param | Type | Notes |
|---|---|---|
| `kinds` | list = [model, user] | `model, user, reference` |
| `names` | string[]? | |

Returns `[{name, kind, expression, value (mm|deg|unitless), units, comment, driven_features}]`.
API: `ComponentDefinition.Parameters` → `ModelParameters`, `UserParameters`, `ReferenceParameters` → `Parameter.Expression`, `Value` (database units) `[V:doc]`/`[V:interop]`; formatting `UnitsOfMeasure.GetStringFromValue` `[V:doc]`.

#### `set_parameters` — write · P2
| Param | Type | Notes |
|---|---|---|
| `set` | `[{name, expression}]` | expressions with units |
| `create_user` | `[{name, expression, units}]` | |
| `update` | bool = true | |

API: `UnitsOfMeasure.IsExpressionValid` / `GetLocaleCorrectedExpression` `[V:doc]`; `Parameter.Expression = …` `[V:interop]`; `UserParameters.AddByExpression(Name, Expression, UnitsSpecifier)` `[V:doc]`; `doc.Update()` `[V:doc]`.

#### `get_properties` — RO · I · P2
Returns standard + custom iProperties, material, appearance.
API: `Document.PropertySets.Item(<internal GUID name>)` — Summary `{F29F85E0-4FF9-1068-AB91-08002B27B3D9}`, Document Summary `{D5CDD502-2E9C-101B-9397-08002B2CF9AE}`, Design Tracking `{32853F0F-3444-11D1-9E93-0060B03C1CA6}`, User Defined `{D5CDD505-2E9C-101B-9397-08002B2CF9AE}` `[V:doc]`; `PartDocument.ActiveMaterial`, `ActiveAppearance` `[V:doc]`.

#### `set_properties` — write · P2
| Param | Type | Notes |
|---|---|---|
| `properties` | `{name: value}` | standard names (Part Number, Description, Title, …) or custom |
| `material` | string? | library material name; `did_you_mean` on miss |
| `appearance` | string? | |
| `targets` | handle[]? | occurrences/documents for bulk edits (assemblies) |

API: `PropertySet.Item(name).Value = …`; custom: `PropertySet.Add(Value, Name)` `[V:interop]`/`[V:doc]`; material: `Application.AssetLibraries` → `Asset.CopyTo(doc)` → `PartDocument.ActiveMaterial = asset` `[V:interop]`/`[V:code]`.

### View

#### `get_view` — RO · P1
| Param | Type | Notes |
|---|---|---|
| `view` | ViewPreset = iso | |
| `size` | int = 768 | long edge, clamped 256–1568 |
| `fit` | `all \| <handles>` = all | |
| `labels` | `none \| faces \| edges` = none | P3: overlay `F#/E#` labels matching `describe_geometry` |
| `style` | `shaded \| shaded_edges \| wireframe` | `[U]` (`DisplayModeEnum` values unverified) |

Returns an MCP **image** content block (PNG) + one line of text (view, size).
API: `Application.ActiveView` → `View.Camera` → `Camera.ViewOrientationType = ViewOrientationTypeEnum` (iso-top-right 10759, front 10764, top 10754, right 10755, back 10756, bottom 10757, left 10758) `[V:doc]` → `Camera.Fit()` → `Camera.ApplyWithoutTransition()` → `View.Update()` (needed before capture `[V:code]`) → `Camera.SaveAsBitmap(FullFileName, Width, Height, [TopColor], [BottomColor])` `[V:doc]` (2026.1+: `SaveAsBitmapWithOptions`) → restore saved camera. Blank-image detection server-side.

### Export

#### `export` — write · P2
| Param | Type | Notes |
|---|---|---|
| `format` | `step \| iges \| sat \| parasolid \| jt \| stl \| obj \| pdf \| dwg \| dxf` | |
| `path` | string | allow-listed output root |
| `step` | `{protocol: ap203\|ap214\|ap242 = ap242}` | |
| `stl` / `obj` | `{binary: true, units: mm, resolution: high\|medium\|low}` | `units: mm` → `ExportUnits = 5` (**not 4** = cm, the default) `[V:doc]` verified |
| `overwrite` | bool = false | |

Returns `{path, bytes, format, verified: {fresh: true, step_schema?}}`.
API: `Application.ApplicationAddIns.ItemById(GUID)` (activate if needed) → `TransientObjects.CreateTranslationContext()` (`Type = kFileBrowseIOMechanism`) + `CreateNameValueMap()` + `CreateDataMedium()` → `TranslatorAddIn.HasSaveCopyAsOptions(doc, ctx, opts)` → set options (`NameValueMap.Add` out-of-process `[V:code]`) → `TranslatorAddIn.SaveCopyAs(doc, ctx, opts, medium)` `[V:doc]`. GUIDs in 04 §7. STEP `ApplicationProtocolType`: 2 AP203, 4 AP214IS (default), 5 AP242 per TranslatorSettings `[V:doc]` verified — but Autodesk's STEP sample says 3 = AP214, so **check `FILE_SCHEMA` in the written header**. PDF/DWG/DXF 2D option names `[U]`.

### Transactions and documentation

#### `batch` — write · P2
| Param | Type | Notes |
|---|---|---|
| `operations` | `[{tool, args}]` (≤ 25) | any core/toolset write tool except `run_code`, `undo`, `close_document` |
| `name` | string = "MCP batch" | transaction name |

Returns per-op results; stops at first failure and aborts the whole transaction.
API: `TransactionManager.StartTransaction(Document, DisplayName)` → … → `Transaction.End()` / `Abort()` `[V:doc]`; aborting in assemblies removes occurrences added inside `[V:code]`.

#### `undo` — write · D · P2
| Param | Type | Notes |
|---|---|---|
| `steps` | int = 1 | MCP transactions only unless `include_user=true` |

API: `TransactionManager.UndoTransaction()` `[V:doc]`; `CommittedTransactions` to locate MCP-named ones `[V:doc]`.

#### `api_lookup` — RO · I · P2
| Param | Type | Notes |
|---|---|---|
| `query` | string | object, member or enum name / substring |
| `limit` | int = 20 | |

Returns `[{object, member, kind, signature, hidden, since?}]` from the **installed** type library (`RxInventor.tlb` / interop reflection) `[V:adn]`.

---

## Toolset `assembly` (P3)

| Tool | Params (abridged) | Inventor API |
|---|---|---|
| `place_component` | `path`, `position_mm`, `rotation_deg {axis, angle}` or `matrix`, `grounded`, `count` | `TransientGeometry.CreateMatrix()` + `SetTranslation`/`SetToRotation` `[V:interop]` → `ComponentOccurrences.Add(FullDocumentName, Position)` `[V:doc]` (file must exist on disk `[V:code]`); `AddByComponentDefinition` `[V:doc]` |
| `place_content_center_part` | `family_path` (e.g. `Fasteners/Bolts/Socket Head Bolts/ISO 4762`), `size` / row filter `{column: value}`, `custom_input`, `position_mm` | `Application.ContentCenter.TreeViewTopNode` → `ChildNodes` / `Families` → `ContentFamily.TableRows` → `ContentFamily.CreateMember(Row, out FailureReason, out FailureMessage, …)` returns file path `[V:doc]` → `Occurrences.Add`; requires `Application.Ready` `[V:code]`; cache navigation (full walk takes minutes `[V:code]`) |
| `transform_occurrence` | `occurrences`, `translate_mm`, `rotate_deg`, `absolute` | `AssemblyComponentDefinition.TransformOccurrences(Occurrences, Transforms, IgnoreConstraints)` `[V:interop]` |
| `set_occurrence_state` | `occurrence`, `grounded?`, `visible?`, `suppressed?`, `delete?` | `ComponentOccurrence.Grounded`, `Visible`, `Suppress()` `[V:interop]` |
| `add_constraint` | `type: mate\|flush\|angle\|insert\|tangent`, `a`, `b` (handles; origin plane/axis names allowed), `offset_mm`, `angle_deg`, `opposed`, `solution` | auto proxies `ComponentOccurrence.CreateGeometryProxy(Geometry, out Result)` `[V:doc]`; `AssemblyConstraints.AddMateConstraint2(...)` `[V:doc]`, `AddFlushConstraint(...)` `[V:doc]`, `AddAngleConstraint(...)` `[V:doc]` (rad), `AddInsertConstraint2(...)` `[V:doc]`, `AddTangentConstraint(...)` `[V:interop]`; then `doc.Update()` (transform reports pre-solve pose until update `[V:code]`) |
| `add_joint` | `type: rigid\|rotational\|slider\|cylindrical\|planar\|ball`, `origin1`, `origin2` (handle + intent: start/mid/end/center) | `AssemblyComponentDefinition.CreateGeometryIntent(Geometry, [Intent])` `[V:interop]` (intent enum values `[U]`) → `AssemblyJoints.CreateAssemblyJointDefinition(JointType, OriginOne, OriginTwo)` `[V:doc]` → `AssemblyJoints.Add(def)` `[V:interop]`; `AssemblyJointTypeEnum` 102401–102406 `[V:doc]` |
| `check_interference` | `occurrences?` (default all), `set2?` | `AssemblyComponentDefinition.AnalyzeInterference(Set1, [Set2])` `[V:doc]` → `InterferenceResult.Volume` (cm³→mm³), `OccurrenceOne/Two` `[V:interop]`/`[V:code]` |
| `get_bom` | `view: structured\|parts_only`, `levels` | `AssemblyComponentDefinition.BOM` → `StructuredViewEnabled`/`PartsOnlyViewEnabled` `[V:doc]` → `BOMViews` select by `ViewType` (names are localized `[V:code]`) → `BOMRows` (`ItemNumber`, `ItemQuantity`, `ComponentDefinitions`) `[V:interop]` |
| `list_targets` / `select_target` | — / `pid` | discovery files (no Inventor call) |

## Toolset `drawing` (P3)

| Tool | Params (abridged) | Inventor API |
|---|---|---|
| `add_sheet` | `size: A4…A0\|custom`, `width_mm`, `height_mm`, `orientation`, `name` | `Sheets.Add([Size], [Orientation], [SheetName], [Width], [Height])` `[V:doc]` (custom size only with `kCustomDrawingSheetSize`) |
| `add_views` | `model` (doc handle/path), `base: {orientation, scale, style, position_mm}`, `projected: [{direction: right\|top\|iso_…}]`, `flat_pattern` | `DrawingViews.AddBaseView(Model, Position, Scale, ViewOrientationTypeEnum, DrawingViewStyleEnum, [ModelViewName], [ArbitraryCamera], [AdditionalOptions])` `[V:doc]` (`SheetMetalFoldedModel=False` for flat pattern `[V:doc]`); `AddProjectedView(ParentView, Position, ViewStyle, [Scale])` `[V:doc]`; styles 32257 hidden-line, 32258 hidden-line-removed, 32259 shaded `[V:doc]` |
| `add_section_view` | `parent`, `line: [[x,y],[x,y]]` (sheet mm), `position_mm`, `depth` | `DrawingView.Sketches.Add()` → `DrawingSketch.Edit()` … line … `ExitEdit()` → `DrawingViews.AddSectionView(ParentView, SectionLineSketch, Position, ViewStyle, …)` `[V:doc]` |
| `add_detail_view` | `parent`, `center_mm`, `radius_mm` or rect, `scale`, `position_mm` | `Sheet.Activate()` first `[V:doc]` → `DrawingViews.AddDetailView(ParentView, Position, ViewStyle, CircularFence, FenceCenterOrCornerOne, FenceRadiusOrCornerTwo, …)` `[V:doc]` |
| `retrieve_dimensions` | `view` | capability-switched: ≤2024 `GeneralDimensions.GetRetrievableDimensions` + `Retrieve` `[V:interop]`; 2025 `Sheet.GetRetrievableAnnotations` + `RetrieveAnnotations` `[V:interop]`; 2026.1+ `Sheet.GetRetrievableAnnotations2` + `RetrieveAnnotations2` `[V:doc]` |
| `add_dimension` | `view`, `a`, `b` (drawing-curve refs), `type`, `text_position_mm` | `Sheet.CreateGeometryIntent(DrawingCurve, [Intent])` → `GeneralDimensions.AddLinear(TextOrigin, IntentOne, [IntentTwo], [DimensionTypeEnum])` `[V:doc]` |
| `add_parts_list` | `view`, `position_mm`, `level` | `Sheet.Activate()` → `PartsLists.Add(ViewOrModel, PlacementPoint, [Level], …)` `[V:doc]` |
| `add_balloons` | `view`, `auto: true` or `[{component, leader_mm}]` | `Balloons.Add(LeaderPoints, …)` `[V:doc]` |
| `set_title_block` | `{prompt_name: value}` | `TitleBlock.SetPromptResultText(DefinitionText, NewValue)` `[V:interop]`; `SilentOperation` to avoid prompted-text dialog `[V:code]` |

## Toolset `sheet_metal` (P3)

| Tool | Params (abridged) | Inventor API |
|---|---|---|
| `sheet_metal_setup` | `convert: bool`, `style`, `thickness_mm` | `PartDocument.SubType = "{9C464203-9BAE-11D3-8BAD-0060B0CE6BB4}"` `[V:doc]`; `SheetMetalStyle.Activate()` (assigning `ActiveSheetMetalStyle` fails `[V:code]`); `SheetMetalComponentDefinition.Thickness` `[V:doc]` |
| `sheet_metal_face` | `sketch`, `profile` | `FaceFeatures.CreateFaceFeatureDefinition(Profile)` → `FaceFeatures.Add(def)` `[V:doc]` |
| `sheet_metal_flange` | `edges`, `angle_deg`, `height_mm`, `placement` | 2025+: `FlangeFeatures.CreateDefinition(Edges, FlangeReferenceType, AngleOrPlane, [Placement], [Distance], [Options])` `[V:interop]`; ≤2024: `CreateFlangeDefinition(Edges, Angle, Distance)`; set height via `SetDistanceHeightExtent` (old API ignores Distance `[V:code]`) |
| `sheet_metal_cut` | `sketch` (on the sheet face) | `CutFeatures.CreateCutDefinition(Profile)` `[V:interop]` (work-plane sketches may silently cut nothing `[V:code]`) |
| `flat_pattern` | `refresh`, `export: {format: dxf\|dwg, path, layers}` | `SheetMetalComponentDefinition.Unfold()` → `FlatPattern.ExitEdit()` `[V:interop]`; DXF: `DataIO.WriteDataToFile("FLAT PATTERN DXF?AcadVersion=R12&OuterProfileLayer=Outer&…", path)` `[V:doc]`; 2025+ translator `SaveCopyAs` accepts a `FlatPattern` source `[V:doc]` |

## Toolset `advanced_features` (P3)

| Tool | Inventor API |
|---|---|
| `add_sketch_constraints` | `GeometricConstraints.AddCoincident/AddHorizontal/AddVertical/AddParallel/AddPerpendicular/AddTangent/AddEqualLength/AddConcentric/AddSymmetry/AddGround` `[V:interop]`; `DimensionConstraints.AddTwoPointDistance(P1, P2, Orientation, TextPoint, [Driven])` `[V:doc]`, `AddDiameter`, `AddRadius`, `AddTwoLineAngle` `[V:interop]`; `DimensionOrientationEnum` 19201–19203 `[V:doc]` |
| `project_geometry` | `PlanarSketch.AddByProjectingEntity(Entity)` `[V:interop]` |
| `sweep` | `PartFeatures.CreatePath(SketchCurve)` `[V:interop]` → `SweepFeatures.CreateSweepDefinition(SweepType, Profile, Path, Operation)` → `Add` `[V:doc]` |
| `loft` | `LoftFeatures.CreateLoftDefinition(Sections ObjectCollection, Operation)` `[V:doc]` → `Add` `[V:interop]` |
| `coil` | `CoilFeatures.AddByPitchAndRevolution(Profile, Axis, Pitch, Revolution, Operation, …)` `[V:interop]` |
| `thread` (cosmetic) | `ThreadFeatures.CreateStandardThreadInfo(...)` → `ThreadFeatures.Add(Face, StartEdge, ThreadInfo, …)` `[V:doc]` |
| `emboss` | `EmbossFeatures.AddEmbossFromFace(Profile, Distance, Direction, …)` `[V:interop]` |
| `slot` (2027+) | `SlotFeatures.CreateSketchPlacementDefinition` → `CreateSlotDefinition` → `Add` `[V:interop]` (2027 only) |

## Toolset `code` (P4, off by default)

#### `run_code` — write · D
| Param | Type | Notes |
|---|---|---|
| `language` | `csharp \| ilogic` | |
| `code` | string | `app`, `doc`, `cd`, `tg` pre-bound |
| `timeout_s` | int = 30 (max 300) | |
| `allow_unsaved` | bool = false | refuse on dirty documents otherwise |

Behavior: requires add-in setting `code_exec.enabled=true`; shows **in-Inventor review dialog** with the code (default on); runs inside a transaction (abort on exception); output capped at 60 KB; appended to audit log with SHA-256. Never persists iLogic rules into the document.
API: C# via Roslyn scripting in the add-in `[V:code]` (InventorBridge, tylerwarner33); iLogic via add-in `{3BDD8D79-2179-4B11-8A5A-257B1C0263AC}` → `Automation.RunRuleDirect…` `[V:adn]`/`[V:doc]` (note `AddRule` compiles **and runs and persists** `[V:doc]`; 2025+ .NET Framework hosts had `RunRule` failures `[V:2nd]`).

---

## MCP prompts and resources

| Kind | Name | Content |
|---|---|---|
| prompt | `modeling_workflow` | Units (mm/deg), sketch→profile→feature order, call `describe_geometry` before selecting edges, check `verify` after every feature, prefer `set_parameters` over rebuilding, use `batch` for multi-step edits, ask the user to select geometry when ambiguous (`describe_geometry(source="selection")`). No hidden directives. |
| prompt | `drawing_workflow` | Base view → projected → retrieve dimensions → parts list; sheet must be active. |
| resource | `inventor://capabilities` | Capability map (version, available APIs). |
| resource | `inventor://thread-table` | Installed thread standards/designations for `hole`/`thread`. |

## Coverage vs Autodesk's official server (2027.2+)

| Area | Autodesk (46 tools) | This catalog |
|---|---|---|
| Documents, properties, parameters | ✔ | ✔ (minimal, needed on all versions) |
| Representations / Model States / member tables | ✔ | ✖ (defer to Autodesk) |
| iLogic rule management, event triggers | ✔ | ✖ (defer to Autodesk) |
| Sketches, part features, work features | ✖ (code only) | ✔ |
| Stable geometry handles + descriptors | ✖ | ✔ |
| Assembly constraints / joints / Content Center | place only | ✔ |
| Drawing views, dimensions, parts lists | ✖ (PDF export only) | ✔ |
| Sheet metal + flat-pattern DXF | ✖ | ✔ |
| Image returned to the model | ✖ (file only) | ✔ |
| Verification deltas on every write | ✖ | ✔ |
| Inventor versions | 2027.2+ | 2025–2027 (+2022–2024 optional) |
