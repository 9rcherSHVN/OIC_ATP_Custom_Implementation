PO-inbound-flow.mmd is a large mermaid file. You are an expert in mermaid scripting. You must split this large file into multiple small diagrams with enhanced readability.


Splitting Strategy

The diagram has 7 natural phase boundaries already marked with rect blocks. Use a three-tier architecture as recommended below:


###Tier 1 — Master Overview Diagram (1 file)
A high-level flowchart LR (not sequence) showing all 8 integrations as nodes with phase labels and data flow arrows. This gives readers the "map" before they dive in. Sequence diagrams are poor at conveying topology — a flowchart does it better.

###Tier 2 — Phase Group Diagrams (3 files)
Group the 7 phases into 3 functional groupings that share participants, reducing repeated participant declarations:

#File				#Phases			#Rationale
diagram-ingest.md		Ph1 + Ph2 + Ph3		All about getting EAM data into ATP — share EAM, ATP, AppEvt
diagram-validate-import.md	Ph4 + Ph5		Share BIP, ERP, ATP — the transformation/submission core
diagram-callback-writeback.md	Ph6 + Ph7		Share EAM, BIP, ATP, AppEvt — result propagation back to EAM


This reduces from 8 re-declared participant blocks in a single file to 3 isolated scopes.

###Tier 3 — Atomic Phase Diagrams (7 files, optional drill-down)
One file per phase, linked from Tier 2. Only generate these for the two most complex phases:

1.diagram-phase4-validation.md — the 5 BIP report loops + dual item creation loops are each worth isolating
2.diagram-phase6-callback.md — the dual-router pattern with 4 BIP calls is confusing without isolation


### Concrete Structural Fixes Per File
Beyond splitting, apply these Mermaid-specific improvements in every sub-diagram:

1. Replace inline Note over walls with box grouping for actors

2. Extract variable assignment chains into a dedicated Note right of block at the start of each phase — avoids mid-flow annotation clutter.

3. Flatten nested loops where depth > 1

The Phase 7 diagram has a loop-within-a-loop-within-a-loop (batch → header → line). Split into two sequence diagrams:

F8-success-path.md — success PO update flow
F8-error-path.md — error header + error line update flow

4. Use critical / option blocks (Mermaid v10.9+) instead of CATCH_ALL notes

5. Fix the typo in variable name (lv_line_nextcursorpositon) — standardize to lv_line_nextcursorposition in the diagram label for documentation accuracy.

Recommended File Naming Convention

/diagram/
├── 00-overview-topology.md          ← Tier 1 flowchart
├── 10-ingest-phases-1-2-3.md        ← Tier 2
├── 20-validate-import-phases-4-5.md ← Tier 2
├── 30-callback-writeback-phases-6-7.md ← Tier 2
├── drill/
│   ├── 04a-validation-bip-loads.md
│   ├── 04b-validation-item-creation.md
│   ├── 06a-callback-import-results.md
│   └── 08a-writeback-success-path.md
│   └── 08b-writeback-error-path.md