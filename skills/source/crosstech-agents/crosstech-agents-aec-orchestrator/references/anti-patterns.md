# crosstech-agents-aec-orchestrator — Anti-Patterns Reference

## Anti-Pattern 1: Skipping Between-Hop Validation

**Wrong**:
```
Hop 1: Export IFC from FreeCAD
Hop 2: Push quantities to ERPNext  ← Directly, no validation
Result: ERPNext receives zero quantities because FreeCAD export had no IfcElementQuantity
```

**Why it fails**: Silent data loss at technology boundaries is invisible until the final output is wrong. By then, the root cause is buried 2-3 hops back.

**Correct**: ALWAYS validate after EACH hop. At minimum: check entity count, spot-check 3 elements for properties, verify file is parseable.

```
Hop 1: Export IFC from FreeCAD
  → VALIDATE: schema version, product count > 0, sample properties exist
Hop 2: Push quantities to ERPNext
  → VALIDATE: BOM items generated, quantities > 0
```

---

## Anti-Pattern 2: Loading a Single Skill for a Multi-Boundary Request

**Wrong**:
```
User: "Convert my Revit model to a Three.js web viewer"
Agent: Loads only crosstech-impl-ifc-to-threejs
Result: Misses the Revit → IFC boundary entirely
```

**Why it fails**: The request crosses TWO boundaries (Revit → IFC, IFC → Three.js). Loading only one skill means the other boundary is handled without guidance, introducing errors.

**Correct**: ALWAYS decompose into individual boundary crossings first. Load one skill per boundary.

```
User: "Convert my Revit model to a Three.js web viewer"
Agent: Decomposes into:
  1. Revit → IFC (covered by Revit/BIM package)
  2. IFC → Three.js (crosstech-impl-ifc-to-threejs — DEFERRED)
  3. Foundation: crosstech-core-ifc-schema-bridge (IFC involved)
  4. Foundation: crosstech-core-coordinate-systems (Y-up/Z-up swap needed)
```

---

## Anti-Pattern 3: Guessing at Error Causes Instead of Loading the Error Skill

**Wrong**:
```
Hop 2 fails: model appears at wrong location in QGIS
Agent: "Try changing the EPSG code to 28992"  ← Guessing without diagnosis
Result: Still wrong — actual cause was missing IfcMapConversion, not wrong EPSG
```

**Why it fails**: Coordinate errors have multiple possible root causes (wrong CRS, missing georeferencing, unit mismatch, axis flip). Guessing wastes time and can introduce new errors.

**Correct**: ALWAYS load `crosstech-errors-coordinate-mismatch` and follow the symptom-to-root-cause decision tree.

```
Hop 2 fails: model appears at wrong location in QGIS
Agent: Loads crosstech-errors-coordinate-mismatch
  → Symptom: "Model at (0,0) / Gulf of Guinea"
  → Root cause: Missing IfcMapConversion
  → Fix: Add georeferencing using ifcopenshell.api.run("georeference.add_georeferencing")
```

---

## Anti-Pattern 4: Restarting the Entire Pipeline After a Single Hop Failure

**Wrong**:
```
4-hop pipeline: FreeCAD → IFC → Docker → n8n
Hop 3 (Docker) fails
Agent: Restarts from Hop 1 (FreeCAD export)
Result: Wasted 20 minutes re-running successful hops
```

**Why it fails**: Earlier hops already produced valid output (verified by checkpoints). Re-running them wastes time and risks introducing new errors if inputs have changed.

**Correct**: NEVER restart from the beginning. Fix the failed hop, retry ONLY that hop, and continue from there.

```
Hop 3 (Docker) fails
Agent:
  1. Diagnose the Docker failure
  2. Fix the issue (e.g., update container image)
  3. Retry Hop 3 using the existing output from Hop 2
  4. Validate Hop 3 output
  5. Continue to Hop 4
```

---

## Anti-Pattern 5: Chaining Too Many Hops Without Checkpoints

**Wrong**:
```
6-hop pipeline with no intermediate validation:
FreeCAD → IFC → Speckle → Blender → IFC → Docker → n8n
Final output: completely wrong, no way to know where it broke
```

**Why it fails**: Error propagation across 6 unvalidated hops makes debugging impossible. A unit mismatch at hop 2 becomes a geometry explosion at hop 5.

**Correct**: NEVER chain more than 4 hops. Insert validation checkpoints after EVERY hop. For pipelines longer than 4 hops, split into sub-pipelines with verified intermediate outputs.

---

## Anti-Pattern 6: Assuming Same Units Across Technologies

**Wrong**:
```python
# FreeCAD uses mm internally, IFC file uses meters
freecad_length = 5000  # mm
# Directly used as IFC quantity without conversion
ifc_quantity = freecad_length  # Bug: this is 5000 meters, not 5 meters
```

**Why it fails**: Different AEC tools use different internal units. FreeCAD uses mm, Revit uses feet internally, IFC can use any unit defined in IfcUnitAssignment.

**Correct**: ALWAYS check units on BOTH sides of a boundary crossing. ALWAYS apply conversion factors explicitly.

```python
# Read IFC unit scale
import ifcopenshell.util.unit
scale = ifcopenshell.util.unit.calculate_unit_scale(model)
# scale = 0.001 if IFC uses mm, 1.0 if IFC uses meters

# Convert FreeCAD mm to IFC units
freecad_length_mm = 5000
ifc_length = freecad_length_mm * 0.001 / scale  # Correct conversion
```

---

## Anti-Pattern 7: Ignoring Deferred Skills and Proceeding Without Guidance

**Wrong**:
```
User: "Send my IFC model to Speckle and view in Blender"
Agent: Skills crosstech-impl-speckle-blender is DEFERRED
Agent: Proceeds anyway without informing user, writes untested code
Result: Incorrect Speckle object conversion, lost materials
```

**Why it fails**: Deferred skills exist because the boundary is complex enough to warrant dedicated guidance. Proceeding without that guidance leads to the exact errors the skill would prevent.

**Correct**: ALWAYS inform the user when a required skill is deferred. Provide the workaround (available foundation skills + manual steps) and clearly state what package must be installed.

---

## Anti-Pattern 8: Loading Foundation Skills Without a Specific Boundary Skill

**Wrong**:
```
User: "Map IFC quantities to ERPNext"
Agent: Loads only crosstech-core-ifc-schema-bridge
Result: User gets schema information but no ERPNext mapping guidance
```

**Why it fails**: Foundation skills (`core/`) provide reference knowledge but NOT implementation patterns for specific boundaries. The `impl/` skill contains the actual mapping logic.

**Correct**: Foundation skills supplement boundary skills — they do NOT replace them. ALWAYS load the specific `impl/` or `errors/` skill AND the relevant foundation skills.

```
User: "Map IFC quantities to ERPNext"
Agent: Loads:
  1. crosstech-core-ifc-schema-bridge (foundation — IFC quantity structure)
  2. crosstech-impl-ifc-erpnext-costing (boundary — actual mapping logic)
```

---

## Anti-Pattern 9: Silent Retry Without Logging

**Wrong**:
```
Hop 2 fails → retry → fails → retry → fails → retry → succeeds on 4th try
No logs kept of why it failed 3 times
Result: Intermittent bug masked, will fail again in production
```

**Why it fails**: Silent retries mask root causes. If a hop fails intermittently, there is a real bug (race condition, resource limit, network timeout) that will recur.

**Correct**: ALWAYS log error details before retrying. NEVER retry more than 3 times. After 3 failures, escalate to human review with all error logs.

---

## Anti-Pattern 10: Treating the Orchestrator as a Single-Skill Loader

**Wrong**:
```
User: "How do I use IfcOpenShell?"
Agent: Loads crosstech-agents-aec-orchestrator
Result: Orchestrator has no IfcOpenShell API details — it only routes
```

**Why it fails**: The orchestrator is a ROUTING skill, not a content skill. It tells you WHICH skills to load, not HOW to use individual technologies.

**Correct**: The orchestrator is loaded when multiple technologies are involved or when the user needs help selecting the right approach. For single-technology questions, load the relevant `core/` or `impl/` skill directly.
