# SVS AI Manager — ML Trained Model Files

This repository contains trained YOLO and classification models for the SVS AI Manager system, along with training notebooks and datasets.

---

## SVS AI Manager: Installation Verification — Technical Plan & Class Mapping

### Pii (Pre-Installation Inspection) Checklist → AI Model Architecture

### System Overview

The SVS AI Manager validates each step of an ice machine installation by analyzing technician photos in real time. The system uses a two-layer architecture: YOLO26 object detection identifies components and their locations, then a classification layer makes condition and compliance judgments on cropped regions.

**Current model:** YOLO26s-seg trained on 651 images across 20 classes. Box mAP50: 0.794. Dataset growing with each annotation cycle.

---

### Step-by-Step Verification Map

Each row maps an installation checklist step to the AI model classes, pass/fail criteria, and implementation notes.

| Step | Check | AI Layer | Model Classes | Pass Criteria | Notes |
|------|-------|----------|---------------|---------------|-------|
| 1 | Nameplate / Data Plate | YOLO + OCR | `data_sticker`, `nameplate` | Nameplate detected, text readable. OCR extracts model/serial. | Also covers Step 3 + Step 13 (condenser data plate). Gatekeeper model verifies photo quality first. |
| 2 | Unit Disconnect | YOLO | `disconnect_ice_head` | Disconnect switch/breaker detected in photo. | Confirms electrical disconnect is accessible and visible. |
| 4 | Leak Test (Unit) | YOLO + Classification | `leak_test_unit` | Wet surface detected = PASS. Dry = needs retest. | Classification layer determines wet vs dry. Hardest visual check. |
| 5+6 | Water Line + Water Stop | YOLO + Classification | `water_line`, `water_stop`, `valve` | Water line detected. Stop valve detected and open. Material classified. | Same photo covers both. Classification identifies copper/PEX/beverage. Valve state: open vs closed. |
| 7 | Water Line Material Change | Classification | `step_07_water_line_material_change` | Yes/No input. If material changes, classify type. | Materials: Copper, PEX, Beverage. Cannot be garden hose. |
| 8 | Drain Line Size | Classification | `step_08_drain_line_size` | Measurement input. Cannot be a water hose. | Measurement input from tech. Photo verification. |
| 9 | Drain Line | YOLO | `floor_drain`, `drain_outlet`, `drain_line_vent`, `air-gap_fitting` | Drain line present. Air gap fitting detected above floor drain. Vent present. | 2.5" air gap above drain (spatial check). Drain outlet + floor drain both detected. |
| 10 | Bin Drain Line | YOLO | `bin_drain_line` | Connection detected at bin. PVC or copper material. | Gets kicked out by staff sometimes. Must verify it is connected. |
| 11 | Bin Control | YOLO + Classification | `bin_control` | Cable connected and routed clean. Not loose or at risk of scoop damage. | Three bin controls per install (one per unit). Classification: `connected_routed_clean` / `connected_loose` / `disconnected`. |
| 12 | Cube Separator & Guide Mounting | YOLO + Classification | `cube_separator_guides` | Guides not attached. No ice buildup on evaporator. | Ice buildup = possible refrigerant leak. Critical safety flag. |
| 13 | Condenser Data Plate | YOLO + OCR | `data_sticker` (reuse) | Condenser nameplate detected and readable. | Same class as Step 1. System knows context from workflow step. |
| 14 | Disconnect Condenser | YOLO | `disconnect_condenser` | Disconnect detected. Will not fail if upper unit (roof). | New class needed. Similar visual to Step 2 disconnect. |
| 15 | Remote Unit (Condenser) | YOLO | `remote_unit`, `IceMaker`, `BackOfMachine`, `SideOfMachine` | Full-frame photo of unit. Not cropped/close-up. | Straight-on photo required. Gatekeeper rejects close-ups showing only screws. |
| 16 | Secured Unit | YOLO | `secured_unit` | Unit visually secured/mounted. | Full frame photo showing mounting. |
| 17 | Leak Test (Line Set) | YOLO + Classification | `leak_test_line_set` | Condensation visible = PASS. Bubbles = LEAK = FAIL. | Condenser copper lines at weld points. Moisture OK, bubbles NOT OK. Critical safety. |
| 18 | Water Filtration | YOLO + Classification | `water_filter`, `installation_date`, `pressure_gauge` | Date visible. All indicators green (NO BLUE). Copper lines ONLY. | ANY BRAIDED LINES = VIOLATION. Tech must create work order to GC. Half-diameter copper lines required. |
| 19 | Filtration Type | Classification | `step_19_filtration_type` | Water-Wise or Aquamor identified. | Yes/No classification. Identifies brand/type. |
| 20 | Pressure Gauge | YOLO + Classification | `pressure_gauge` | 30-80 PSI = GREEN. 80-100 PSI = WARNING (yellow alert to tech). | Reading classification. Inner diameter matters, not outer. Alert system for high pressure. |
| 21 | Unit On / Ice On | YOLO | `unit_on` | Unit display/indicator shows powered on and making ice. | Final verification step. Installation complete. |

> **Legend:** Critical safety check &nbsp;|&nbsp; Warning/judgment call &nbsp;|&nbsp; Final sign-off

---

### Violation & Alert Rules

These rules define automatic actions when the model detects specific conditions. Critical violations generate work orders.

| Condition | Automated Action | Severity |
|-----------|-----------------|----------|
| Braided lines at water filtration | HARD FAIL — Tech creates work order to General Contractor | CRITICAL |
| Garden hose used as water line | HARD FAIL — Immediate replacement required | CRITICAL |
| Bubbles at line set weld points | HARD FAIL — Refrigerant leak. Do not proceed. | CRITICAL |
| Ice buildup on cube separator/evaporator | FLAG — Possible refrigerant leak. Inspect further. | HIGH |
| Bin control cable loose/disconnected | FLAG — Risk of cable damage from ice scoop | HIGH |
| Pressure gauge 80-100 PSI | WARNING — Alert tech. May be acceptable per unit spec. | MEDIUM |
| Blue indicator on water filter | FAIL — Filter not properly seated or expired | HIGH |
| Missing drain line vent | FAIL — Risk of siphoning and contamination | HIGH |
| No air gap at drain termination | FAIL — Health code violation | HIGH |
| Bin drain line missing/disconnected | FAIL — Must be reconnected before sign-off | MEDIUM |

---

### Classes Still Needed

The following classes need to be added to the YOLO model or classification layer to achieve full checklist coverage.

#### New YOLO Detection Classes

| Class | Description |
|-------|-------------|
| `disconnect_ice_head` | Electrical disconnect at ice head unit |
| `disconnect_condenser` | Electrical disconnect at condenser/roof |
| `leak_test_unit` | Wet surface indicating successful leak test |
| `leak_test_line_set` | Condensation vs bubbles at copper weld points |
| `bin_control` | Cable routing and connection state at ice bin |
| `cube_separator_guides` | Guide mounting and ice buildup detection |
| `remote_unit` | Full-frame condenser photo |
| `secured_unit` | Unit mounting verification |
| `unit_on` | Powered-on indicator/display |
| `roof_penetration` | Candy cane PVC routing through roof |
| `valve` (open/closed) | Water stop valve state |

#### New Classification Classes

| Class | Description |
|-------|-------------|
| `braided_line_violation` | CRITICAL: braided vs copper at filtration |
| `water_line_material` | copper / PEX / beverage / garden hose (violation) |
| `bin_control_state` | `connected_routed_clean` / `connected_loose` / `disconnected` |
| `pressure_reading` | 30-80 (pass) / 80-100 (warning) / 100+ (fail) |
| `filter_indicator_color` | green (pass) / blue (fail) |
| `leak_state` | condensation (pass) / bubbles (fail) / dry (retest) |
| `rooftop_condition` | Condenser mounting quality assessment |
| `drain_line_size` | Measurement verification |
| `filtration_type` | Water-Wise / Aquamor identification |

---

### Data Collection Priority

Based on current model performance and business impact, focus annotation effort in this order:

| Priority | Class / Task | Current Status | Why | Target Images |
|----------|-------------|----------------|-----|---------------|
| 1 | `braided_line_violation` | Not yet trained | Highest business impact — triggers GC work orders | 100+ |
| 2 | `air-gap_fitting` | 0.045 mAP50 | Only 1 validation instance | 50-80 |
| 3 | `drain_line_vent` | 0.400 mAP50 | Health code requirement | 50-80 |
| 4 | `water_stop` | 0.322 mAP50 | Critical for valve verification | 80-100 |
| 5 | `drain_outlet` | 0.554 mAP50 | Needed for air gap verification | 60-80 |
| 6 | `water_line` | 0.585 mAP50 | Material classification depends on this | 80-100 |
| 7 | `bin_drain_line` | 0.624 mAP50 | Frequently displaced by staff | 50-80 |
| 8 | `bin_control` (new) | Not yet trained | Cable routing safety | 60-100 |
| 9 | `leak_test` classes (new) | Not yet trained | Safety-critical verification | 80-100 |
| 10 | `disconnect` classes (new) | Not yet trained | Electrical safety | 50-80 |

**Total estimated new images needed for full checklist coverage:** 600-900 additional annotated images.
Combined with existing 651 images, target dataset size is approximately **1,200-1,500 images** across all classes.
