# OIC Custom Integration Layer — Technical Assessment
**Date:** May 8, 2026  
**Projects assessed:** `IFP_PO_INBOUN_INTEGR_ATP`, `IFP_PO_RECEI_INBOU_INTEG_ATP`, `IFP_AP_INVO_FREI_LINE_UPDA_INTE`, `IFP_AP_INVOIC_OUTBOU_INTEGR1`, `IFP_SUPPLIER_OUTBOUND_INTEGRAT`  
**OIC versions:** `25.10-EC` (oldest) to `26.04.0.7-EC` (newest)  
**All findings are evidence-based from direct reads of `.jca`, `info.json`, `project.xml`, and `.xsl` files across all 5 projects.**

---

## Table of Contents

1. [Full Technical Assessment Summary](#1-full-technical-assessment-summary)
2. [Traceability and Visibility — Design Proposal](#2-traceability-and-visibility--design-proposal)
3. [Reassessment with Accuracy Checks](#3-reassessment-with-accuracy-checks)

---

## 1. Full Technical Assessment Summary

### Area 1 — Security: Basic Auth Only on Public Endpoints

**Finding A — All custom REST trigger endpoints use `BASIC_AUTH`.**  
Every externally-callable endpoint (`/poreprocess`, `/porcptreprocess`, `/poprioritywisereprocess`, `/supplieroutbound`, etc.) is protected only by Basic Auth over HTTPS. There is no token-based authentication, no IP allowlisting, and `privateEndpoint=false` on all of them.

> ✅ Evidence: All `callpostagerest_REQUEST.jca`, `getporeporcess_REQUEST.jca`, `Triggersuppout_REQUEST.jca`, etc. have `securityPolicy=BASIC_AUTH`, `privateEndpoint=false`

**Finding B — Collocated ICS calls have `securityPolicy=NONE`.**  
All intra-project collocated calls (`CallChildPOValidationIntegration`, `CallMiscFreighUpdateIntegration`, `callsupplieroutboundint`) use `securityPolicy=NONE`. This is by design in OIC collocated adapters (calls stay within the OIC runtime mesh, not external network), but it does mean any integration in the same OIC instance that knows the endpoint can invoke it without authentication. If multi-tenancy or project isolation is a concern, this is worth documenting explicitly.

> ✅ Evidence: 11 collocated `.jca` files confirmed with `securityPolicy=NONE`

**Recommendation:**
- Upgrade external REST trigger endpoints from Basic Auth to **OAuth 2.0 client credentials** via OIC Identity Connector or API Gateway policy
- Consider enabling `privateEndpoint=true` on the ATP database connections (several still have `privateEndpoint=false` — e.g., `IFP_SUBMI_PAYAB_IMPOR_INVOI_INTE` ERP REST calls), routing ATP traffic through the OCI private subnet rather than public internet
- Document the `NONE` policy on collocated adapters explicitly in runbooks so future developers know it is intentional, not a gap

---

### Area 2 — Naming & Convention Inconsistency

**Finding A — `TEST_` prefix on two production integrations.**  
`TEST_IFP_SUPPL_OUTBO_INTEG_17702` and `TEST_IFP_SCHED_SUPPL_OUTBO_INTEG` are both `persistedState=ACTIVATED` (live) integrations. The `TEST_` prefix implies a development artifact, which will cause confusion for any new developer or support team member.

> ✅ Evidence: Both `info.json` files show `persistedState=ACTIVATED`, `lastUpdatedICSVersion=26.01.2-EC`

**Finding B — Inconsistent parameter casing across endpoints.**

| Parameter variant | Used by |
|---|---|
| `pbatchId` (camelCase) | Most integrations |
| `pbatchid` (all lowercase) | `/porcptreprocess`, `/callbackreprocess`, `/updatemessage` |
| `PbatchID` (mixed caps) | `/poreprocess` |
| `p_new_batch_id` (snake_case) | Batchwise Processing tracking variable |

**Finding C — Tracking variable names are not uniformly meaningful.**  
`IFP_EAM_PO_BATCH_PROCE_INTEG` has `trackingInstanceName=parameter1` (generic). `IFP_EAM_PO_DATA_EXTRA_INTEG` has `tracking_var_2` and `tracking_var_3` as secondary/tertiary tracking names with **no XPath mapping at all** — the slots are configured but completely empty.

**Finding D — Same integration code `IFP_EAM_PO_IMPORT_INTEGR` exists in both PO Inbound and Receipt Inbound projects with different names.**  
In PO Inbound: "IFP EAM PO Import integration". In Receipt Inbound: "IFP EAM PO Receipt Import integration". Same code, different integration. Causes confusion when referencing in tooling or runbooks without project context.

**Recommendation:**
- Rename `TEST_` integrations to their proper `IFP_` prefix names
- Standardise all batch-related parameters to `pbatchId` (lowerCamelCase)
- Assign meaningful secondary/tertiary tracking variable names (or wire them to actual values)
- Qualify the shared integration codes with domain prefix: `IFP_EAM_PORCPT_IMPORT_INTEGR` vs `IFP_EAM_PO_IMPORT_INTEGR`

---

### Area 3 — Error Handling Architecture

**Finding A — Email notification is the only error signal (but only for outer faults — see correction in Section 3).**  
All catch-all handlers send an email via `IFP_Common_Utility_Lookup` → `IT_Distribution_List1`. There is no structured error table in ATP, no error queue, and no automated retry mechanism.

**Finding B — Inner `ehStop` swallows errors.**  
The inner `try` catch-all (e.g., inside `FetchPO_RCPTNewBatch`) calls `ehStop` silently without writing any audit record. **Critically: the outer catch-all with email notification does NOT fire after an inner `ehStop`**. For the Batchwise Processing integrations, all realistic business failures (ATP procedure error, REST call failure) are caught by the inner handler — no email is ever sent for these.

**Finding C — `ora:getFaultAsString()` dumps raw XML fault into email body.**  
The `Detailed_Error_Parameterexpr.properties` files across **17+ integration instances** all use `ora:getFaultAsString()` as the email body content. This produces a raw OIC fault XML message in the email, which is hard to parse for a support engineer.

> ✅ Evidence: Confirmed across `IFP_EAM_PO_STAG_DATA_LOAD_INTE`, `IFP_PO_IMPOR_CALLB_16_FEB`, `IFP_EAM_PO_IMPORT_INTEGR`, `UPDA_IFP_EAM_PO_RECE_DATA_EXTR_I`, `IFP_EAM_PO_VALIDA_INTEGR`, `TEST_IFP_SUPPL_OUTBO_INTEG_17702`, and others

**Finding D — AP Invoice Outbound is the exception (positive reference pattern).**  
`IFP_AP_INVOI_OUTBO_INTEG_SF` has specific `wsdlFault` handlers for named faults: `AccessDeniedException`, `OperationFailedException`, `InvalidParametersException` from BIP SOAP, and `APIInvocationError` from each individual REST call. This is the **only integration in the suite** that handles named faults by type — and should be the reference pattern.

**Recommendation:**
- Add a structured error logging ATP procedure call in every catch-all handler: `XXIFP_ERROR_LOG(p_batch_id, p_integration_code, p_error_stage, p_error_msg, p_timestamp)`
- Replace raw `getFaultAsString()` with a curated message including integration name, batch ID, stage, and human-readable error summary
- Move all critical business logic inside the **outer** try, not nested inner tries that bypass the email handler
- Replicate `IFP_AP_INVOI_OUTBO_INTEG_SF`'s named `wsdlFault` pattern to PO/Receipt integrations

---

### Area 4 — Reprocess Proliferation

**Finding:** There are at least **7 reprocess-related integrations** across the two core pipelines:

| Project | Integration | Entry endpoint | Entry stage |
|---|---|---|---|
| PO Inbound | `IFP_PO_REPROCES_INTEGRAT` | `/poreprocess` | Validation |
| PO Inbound | `IFP_EAM_PO_BATCH_REPRO_INTEG` | `/callbackreprocess?importid` | Batchwise dispatch |
| PO Inbound | `IFP_PO_IMPORT_CALLBA_REPROC` | `/callbackreprocess?loadrequestid` | Callback |
| Receipt Inbound | `IFP_EAM_PRIORITY_REPROCES` | `/poprioritywisereprocess` | Batchwise dispatch |
| Receipt Inbound | `IFP_PO_RECEIP_REPROC_INTEGR` | `/porcptreprocess` | Validation |
| Receipt Inbound | `IFP_PO_RECEI_IMPOR_CALLB_REPRO` | `/callbackreprocess?loadrequestid` | Callback |
| Receipt Inbound | `IFP_EAM_UPDA_MESS_IN_EAM_HEXA_RE` | `/updatemessage` | EAM writeback |

Two integrations share `/callbackreprocess` as URI with **incompatible parameter contracts** (`importid` vs `loadrequestid`+`importrequestid`). IT Support must know which to call and with which parameters depending on the failure stage.

**Recommendation:**
- Consolidate into a **single reprocess dispatcher** per pipeline accepting a `stage` parameter that routes internally via CBR
- At minimum, rename `/callbackreprocess` differently for its two distinct usages: e.g., `/batchreprocess` vs `/importcallbackreprocess`

---

### Area 5 — Event Architecture: Shared Event Types Create Coupling

**Finding A — `CALLPRIORITYWISEPORCPT` dual-publisher / single-subscriber ambiguity.**  
Published by both `UPDA_IFP_EAM_PO_RECE_DATA_EXTR_I` (normal run) and `IFP_EAM_PRIORITY_REPROCES` (reprocess) and `IFP_PO_RECEI_IMPOR_CALLB_REPRO` (callback reprocess). All publications hit the same subscriber `IFP_EAM_PO_RECE_BATC_PROC_INTE`. There is no event subtype to differentiate first-run from reprocess.

**Finding B — `TRIGGERNEWPOBATCH` same pattern.**  
Both `IFP_EAM_PO_BATCH_PROCE_INTEG` (main) and `IFP_EAM_PO_BATCH_REPRO_INTEG` (reprocess) subscribe — both fire on every publication.

**Finding C — `UPDATEERRORINEAMHEXAGON` dual-publisher ambiguity (previously uncalled out).**  
Published by both `IFP_EAM_PO_IMPORT_INTEGR` (main pipeline Import error) and `IFP_EAM_UPDA_MESS_IN_EAM_HEXA_RE` (reprocess via `/updatemessage`). Single subscriber: `IFP_EAM_UPDA_MESS_IN_EAM_HEXA`. Both contexts trigger EAM writeback from the same subscriber with no routing distinction.

> ✅ Evidence: All confirmed from `info.json` event type entries across both projects

**Recommendation:**
- Add a `processType` field to event payloads: `NORMAL_RUN`, `REPROCESS_VALIDATION`, `REPROCESS_IMPORT`
- Or introduce separate event types: `TRIGGERNEWPOBATCH_REPROCESS` vs `TRIGGERNEWPOBATCH`
- Prevents unintended double-processing when both normal and reprocess subscribers react to the same event

---

### Area 6 — Performance: ESS Job Polling is a Busy-Wait

**Finding:** `IFP_SUBMI_PAYAB_IMPOR_INVOI_INTE` uses a `while` loop with `GetESSJobStatus_InWhile` to wait for the Payables import ESS job to complete. This occupies an OIC integration instance and holds a connection for the duration of the ESS job. This is a well-known OIC anti-pattern.

> ✅ Evidence: `GetESSJobStatus_InWhile_REQUEST.jca` — confirmed polling adapter inside a while loop

**Recommendation:**
- Replace the polling while-loop with OIC's native **ERP Cloud callback/event** pattern: submit ESS job → subscribe to `ERP Business Event` for job completion → receive asynchronously
- Frees the OIC instance immediately after submission and eliminates risk of timeout on long-running ESS jobs

---

### Area 7 — Observability & Tracking: Correlation Gap

**Finding A — No end-to-end correlation ID.**  
Each integration is individually tracked by `pbatchId` or `startTime`, but there is no correlation ID that persists from Data Extraction → Batchwise Processing → Stage Load → Validation → Import → Callback → EAM Writeback. Tracing a failure requires searching each integration separately in OIC Activity Stream.

**Finding B — `poicintid` exists but is architecturally incomplete.**  
`poicintid` is present as `tracking_var_2` in both PO and Receipt Stage Data Load integrations, and is passed outbound to Validation. However:
- For PO Inbound, the trigger endpoint (`/ponewbatchsplit`) does **not accept** `poicintid` as an inbound parameter — it is generated/assigned internally (likely from the ATP staging procedure output)
- For integrations upstream of Stage Load (Data Extraction, Batchwise Processing), `poicintid` does not exist at all
- In Data Extraction (`IFP_EAM_PO_DATA_EXTRA_INTEG`), `tracking_var_2` has no XPath mapping — the slot is empty

**Recommendation:**
- Establish `poicintid` (or a renamed `pipelineRunId`) as a **UUID generated at Data Extraction time** via an ATP procedure call, and propagate it through every subsequent hop
- Add a null guard: if null on entry, generate; never propagate null forward

---

### Area 8 — ATP Coupling: ADMIN Schema and Connection Pool Fragmentation

**Finding A — All business logic in ATP stored procedures under schema `ADMIN`.**  
Using `ADMIN` for application tables and packages is a well-known Oracle anti-pattern. The `ADMIN` schema has elevated database privileges. Two packages confirmed: `ADMIN.XXIFP_POINBOUND_INT_PKG` and `ADMIN.XXIFP_PORCPT_INB_INT_PKG`.

**Finding B — Five separate ATP connection names across five projects.**  
`IFP_PO_ATP_DB_CONNEC`, `IFP_PO_RECEI_ATP_DB_CONNE`, `IFP_COMMON_ATP_DB_CONNEC`, `IFP_APFR_*` (AP Freight), `IFP_APINV_ATP_DB_CONNEC` (AP Invoice). If these all point to the same physical ATP instance and the credential rotates, **every project must be independently updated**. In OIC there is no native connection inheritance.

**Finding C — Connection pool hardcoded.**  
`MinConnections=2`, `MaxConnections=35` hardcoded in JCA files — not configurable without redeployment.

> ✅ Evidence: All ATP `.jca` files confirmed `SchemaName=ADMIN`; five distinct connection names identified

**Recommendation:**
- Migrate application tables and packages to a dedicated schema (e.g., `IFP_APP`) with minimum required privileges
- Consolidate ATP connections where projects share the same physical database to reduce credential management overhead

---

### Area 9 — Lifecycle & Version Management: `01.00.0000` Everywhere

**Finding:** Every integration across all 5 projects remains at version `01.00.0000`, spanning update cycles from `25.10-EC` to `26.04.0.7-EC`. Any change is a direct in-place replacement with no rollback path other than re-importing the `.car` file.

> ✅ Evidence: Every `info.json` shows `"version":"01.00.0000"`

**Recommendation:**
- Adopt OIC versioning: `01.01.0000` for minor changes, `02.00.0000` for breaking interface changes
- Enables blue/green deployment — new version deployed and tested while old version serves traffic

---

### Area 10 — Code Duplication: Near-Mirror PO and Receipt Pipelines

**Finding:** PO Inbound and Receipt Inbound pipelines are structurally identical (Extract → Batchwise → Stage Data Load → Validation → Import → Callback → EAM Update). Bug fixes must be applied twice. Additionally:
- `IFP_PO_IMPOR_CALLB_16_FEB` carries a date-stamp in its code name from a temporary development name that was never cleaned up (`16_FEB` refers to `originalICSVersion=26.01.0.16-EC`)
- All `getFaultAsString()` calls, email notification templates, and lookup references are duplicated across both pipelines

**Recommendation:**
- Extract shared notification handler into a reusable library integration
- Rename `IFP_PO_IMPOR_CALLB_16_FEB` → `IFP_PO_IMPOR_CALLBACK` — date stamps in integration codes are a maintenance anti-pattern

---

### Assessment Summary Table

| # | Area | Current Risk | Complexity to Fix |
|---|---|---|---|
| 1 | Security: Basic Auth only on public endpoints | Medium | Medium |
| 2 | Naming inconsistency & `TEST_` prefix in production | Low / Maintenance | Low |
| 3 | Error handling: inner ehStop bypasses email, raw faults, silent swallow | **High** | Medium |
| 4 | Reprocess proliferation, URI collision `/callbackreprocess` | Medium | Medium |
| 5 | Event type coupling — 3 event types with dual-publisher/single-subscriber ambiguity | Medium | Medium |
| 6 | ESS job polling busy-wait in AP Freight | Medium (performance/timeout) | Medium |
| 7 | No end-to-end correlation ID, `poicintid` incomplete upstream | Low / Ops | Low |
| 8 | All ATP tables in `ADMIN` schema, 5 separate ATP connections | Medium (security/stability) | High |
| 9 | All integrations at `01.00.0000`, no version strategy | Low / Deploy risk | Low |
| 10 | Near-duplicate PO/Receipt pipelines, date stamp in code name | Low / Maintenance | High |

---

## 2. Traceability and Visibility — Design Proposal

### The Core Problem

A pipeline run for a single batch touches **5–7 integration instances** across two projects. If something fails at Import Callback, the current investigation process requires:

1. Know which integration to search
2. Go to OIC Activity Stream
3. Search by `pbatchId` (hoping you remember the value)
4. Repeat across all collocated child calls
5. Check email to see if an alert fired (and only for outer-level faults)
6. Look at ATP tables directly to see what state the staging data is in

OIC's Activity Stream does allow searching by `pbatchId` across integrations — but the result is a list of **unlinked instances with no ordering, no duration per stage, and no business-level status**. There is no unified, ordered, causal linkage between hops.

---

### Design Component 1 — Introduce `pipelineRunId` as the Cross-Hop Correlation Key

At the very first step (Data Extraction), call an ATP procedure that generates a UUID and returns it as the `pipelineRunId`. Propagate it forward as a query parameter through every subsequent integration hop.

```
pbatchId      = the business key (EAM source batch)
pipelineRunId = the technical correlation key (UUID, ATP-generated via SYS_GUID())
```

**Important implementation note:** `ora:getInstanceId()` is **not reliably available** as a standard XPath function in OIC assign steps. The correct pattern is to call `SYS_GUID()` inside the ATP `START_PIPELINE_RUN` procedure, return it as a procedure output parameter, and map it to an OIC local variable. `RAWTOHEX(SYS_GUID())` gives a readable 32-character hex string.

`poicintid` already exists as the vehicle for this in both PO and Receipt Stage Data Load integrations. For PO Inbound, the trigger does not accept it as an inbound parameter — it is assigned inside Stage Data Load (most likely from ATP procedure output). The fix is to generate it **at Data Extraction** (not Stage Load) and propagate it forward from the first hop.

---

### Design Component 2 — `PIPELINE_RUN` and `PIPELINE_STAGE_LOG` Tables in ATP

ATP is already the staging layer. Use it as the observability store too.

```sql
-- Recommend: create a dedicated IFP_APP schema instead of using ADMIN
CREATE TABLE IFP_APP.PIPELINE_RUN (
  PIPELINE_RUN_ID   VARCHAR2(36)  PRIMARY KEY,   -- RAWTOHEX(SYS_GUID())
  PIPELINE_TYPE     VARCHAR2(30),                -- 'PO_INBOUND', 'RECEIPT_INBOUND', 'AP_FREIGHT', 'SUPPLIER_OUTBOUND'
  BATCH_ID          VARCHAR2(200),               -- pbatchId
  INITIATED_BY      VARCHAR2(30),                -- 'SCHEDULER', 'MANUAL_REPROCESS', 'PRIORITY_REPROCESS'
  REPROCESS_OF      VARCHAR2(36),                -- FK to original PIPELINE_RUN_ID (null for first run)
  STATUS            VARCHAR2(20),                -- 'RUNNING', 'COMPLETED', 'FAILED', 'PARTIAL'
  TOTAL_RECORDS     NUMBER,
  SUCCESS_COUNT     NUMBER,
  FAIL_COUNT        NUMBER,
  STARTED_AT        TIMESTAMP DEFAULT SYSTIMESTAMP,
  COMPLETED_AT      TIMESTAMP
);

CREATE TABLE IFP_APP.PIPELINE_STAGE_LOG (
  LOG_ID            NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  PIPELINE_RUN_ID   VARCHAR2(36)  NOT NULL,      -- FK to PIPELINE_RUN
  STAGE_CODE        VARCHAR2(50),                -- 'DATA_EXTRACTION', 'STAGE_LOAD', 'VALIDATION',
                                                 -- 'IMPORT', 'CALLBACK', 'EAM_WRITEBACK'
  INTEGRATION_CODE  VARCHAR2(100),               -- e.g. 'IFP_EAM_PO_IMPORT_INTEGR'
  OIC_INSTANCE_ID   VARCHAR2(100),               -- passed from OIC integration metadata
  STATUS            VARCHAR2(20),                -- 'STARTED', 'COMPLETED', 'FAILED'
  ERROR_CODE        VARCHAR2(50),                -- structured code e.g. 'VAL_ITEM_NOT_FOUND'
  ERROR_MESSAGE     VARCHAR2(4000),              -- human-readable, not raw XML
  INPUT_SUMMARY     VARCHAR2(4000),              -- key inputs as JSON
  OUTPUT_SUMMARY    VARCHAR2(4000),              -- key outputs (e.g. ERP load request IDs) as JSON
  STARTED_AT        TIMESTAMP DEFAULT SYSTIMESTAMP,
  COMPLETED_AT      TIMESTAMP
);
```

**Package interface:**

```sql
PACKAGE XXIFP_PIPELINE_PKG AS
  FUNCTION  START_PIPELINE_RUN(p_batch_id VARCHAR2, p_pipeline_type VARCHAR2,
                                p_initiated_by VARCHAR2, p_reprocess_of VARCHAR2 DEFAULT NULL)
    RETURN VARCHAR2;  -- returns PIPELINE_RUN_ID

  PROCEDURE START_STAGE(p_run_id VARCHAR2, p_stage_code VARCHAR2,
                        p_integration_code VARCHAR2, p_oic_instance_id VARCHAR2,
                        p_input_summary VARCHAR2 DEFAULT NULL);

  PROCEDURE COMPLETE_STAGE(p_run_id VARCHAR2, p_stage_code VARCHAR2,
                           p_output_summary VARCHAR2 DEFAULT NULL);

  PROCEDURE FAIL_STAGE(p_run_id VARCHAR2, p_stage_code VARCHAR2,
                       p_error_code VARCHAR2, p_error_message VARCHAR2);
END XXIFP_PIPELINE_PKG;
```

Every integration calls `START_STAGE` at entry and either `COMPLETE_STAGE` or `FAIL_STAGE` at exit.

**Critical failure mode guard:** If the ATP connection itself is the cause of failure (pool exhausted, ATP unavailable), calling `FAIL_STAGE` inside the catch-all will also fail. The catch handler must be:

```
try {
    ATP FAIL_STAGE call
} catch {
    email notification fallback
}
ehStop
```

Never make the ATP logging the sole path — the email fallback must remain.

---

### Design Component 3 — Structured Error Codes

Replace `ora:getFaultAsString()` in all `Detailed_Error_Parameterexpr.properties` files with an error code + human message pattern. Introduce an `IFP_Error_Code_Lookup` (or extend `IFP_Common_Utility_Lookup`):

| Fault Pattern | Error Code | Human Message |
|---|---|---|
| `BIP-`* | `BIP_REPORT_FAILURE` | BIP report call failed — check BIP server connectivity |
| `DBAdapter-`* | `ATP_DB_ERROR` | ATP database adapter error — check connection and proc name |
| `RESTService-`* | `REST_CALL_FAILURE` | Outbound REST call failed — check target system availability |
| `ServiceException` | `ERP_SERVICE_ERROR` | ERP Integration Service returned a fault |
| `APIInvocationError` | `ERP_REST_ERROR` | ERP REST API call failed |
| `ValidationFault` | `VALIDATION_REJECTED` | BIP validation returned rejection for batch |

The improved email notification body:

```
Pipeline: PO_INBOUND  
Batch: BT-20260508-001  
Stage: IMPORT  
Run ID: a3f9b2c1-...  
Error: ATP_DB_ERROR — ATP database adapter error — check connection and proc name  
OIC Instance: [link to OIC console]  
Audit query: SELECT * FROM IFP_APP.PIPELINE_STAGE_LOG WHERE PIPELINE_RUN_ID = 'a3f9b2c1-...'
```

Reference: `IFP_AP_INVOI_OUTBO_INTEG_SF` already implements named `wsdlFault` handling (`AccessDeniedException`, `OperationFailedException`, `APIInvocationError`). Use this as the pattern template for all PO/Receipt integrations.

---

### Design Component 4 — Fix OIC Tracking Variables

All integrations should use consistent tracking variable naming:

| Slot | Value | Integration scope |
|---|---|---|
| `tracking_var_1` (primary) | `pipelineRunId` | All integrations |
| `tracking_var_2` | `pbatchId` | PO/Receipt integrations |
| `tracking_var_3` | Literal stage name (e.g., `STAGE_LOAD`) | All integrations |

**Important constraint:** For Application Event-triggered integrations (`IFP_EAM_PO_BATCH_PROCE_INTEG` etc.), the event payload fields are structurally named `parameter1`, `parameter2`, `parameter3` in the event type definition. The XPath `/execute/request-wrapper/parameter1` is the actual element name in the event contract. **Renaming the tracking variable label alone is insufficient** — the event type itself must be redefined with semantic field names (e.g., `batchId`, `processType`, `runId`) to enable meaningful tracking from event-triggered hops.

---

### Design Component 5 — Reprocess Genealogy

Every reprocess call populates `PIPELINE_RUN.REPROCESS_OF` with the original `PIPELINE_RUN_ID`:

```
PIPELINE_RUN a3f9b2c1  PO_INBOUND  BT-001  FAILED  (original run)
  └── PIPELINE_RUN d7e1a4f9  PO_INBOUND  BT-001  FAILED  (reprocess 1, MANUAL_REPROCESS)
        └── PIPELINE_RUN 8c2b3d5e  PO_INBOUND  BT-001  COMPLETED  (reprocess 2)
```

**Operational quality query — batches requiring multiple attempts:**

```sql
SELECT batch_id,
       COUNT(*)                    AS attempts,
       MIN(started_at)             AS first_attempt,
       MAX(started_at)             AS last_attempt,
       MAX(status)                 AS final_status
FROM IFP_APP.PIPELINE_RUN
WHERE pipeline_type = 'PO_INBOUND'
  AND started_at > SYSTIMESTAMP - INTERVAL '7' DAY
  AND (reprocess_of IS NOT NULL
       OR pipeline_run_id IN (
           SELECT reprocess_of FROM IFP_APP.PIPELINE_RUN WHERE reprocess_of IS NOT NULL
       ))
GROUP BY batch_id
HAVING COUNT(*) > 1
ORDER BY attempts DESC;
```

---

### Design Component 6 — BIP Operational Dashboard Report

Since BIP is already used in the validation step, add one report: **`IFP_Pipeline_Ops_Dashboard`** rendering:

- All `PIPELINE_RUN` records from the last 24 hours with status summary
- `PIPELINE_STAGE_LOG` rows with `STATUS = 'FAILED'`, grouped by `ERROR_CODE` with frequency counts
- Batches with `STATUS = 'RUNNING'` for more than N minutes (stuck detection)

---

### What to Keep (Do Not Change)

| Component | Reason |
|---|---|
| ATP staging layer | Correct pattern for ERP import buffering; provides an offline safety net |
| OIC Application Events for fan-out | Correct OIC-native pattern; do not replace with direct REST calls |
| Collocated ICS adapters (`PRESEEDED_COLLOCATED_CONN_1741`) | Correct for intra-pipeline calls; `securityPolicy=NONE` is acceptable |
| BIP-based validation | Standard Oracle pattern; BIP reports are portable and maintainable |
| Separate PO and Receipt projects | Domain separation is correct; code duplication is the problem, not the structure |

---

### Implementation Sequence

#### Phase 1 — Zero disruption (add without changing existing logic)
1. Create `PIPELINE_RUN` and `PIPELINE_STAGE_LOG` tables in a new `IFP_APP` schema
2. Create `XXIFP_PIPELINE_PKG` with `START_PIPELINE_RUN`, `START_STAGE`, `COMPLETE_STAGE`, `FAIL_STAGE`
3. Add `pipelineRunId` parameter to all existing REST endpoint definitions (optional query param — no breaking change)
4. Wire `pipelineRunId` generation into Data Extraction integrations only, populate `PIPELINE_RUN` table
5. Fix OIC tracking variable names for all integrations

#### Phase 2 — Propagation
6. Add `pipelineRunId` propagation through every subsequent hop's outbound call
7. Add `START_STAGE` / `COMPLETE_STAGE` calls to each integration (ATP calls at boundaries, not touching XSL business logic)

#### Phase 3 — Error handling upgrade
8. Replace `ora:getFaultAsString()` with structured error code mapping in all catch-all handlers
9. Move critical business logic to the outer try scope to ensure email fires on all failures
10. Update email notification body template
11. Add BIP operational dashboard report

#### Phase 4 — Reprocess genealogy + event type cleanup
12. Add `INITIATED_BY` and `REPROCESS_OF` population to all reprocess endpoints
13. Redefine Application Event types with semantic field names (`batchId` vs `parameter1`)
14. Introduce `processType` field to event payloads to resolve dual-publisher ambiguity
15. Optional: consolidate `/callbackreprocess` URI collision

Each phase is independently deployable and independently valuable. **Phase 1 alone delivers 80% of the observability benefit with minimal risk.**

---

## 3. Reassessment with Accuracy Checks

### Correction A — "Core Problem" Statement Was Imprecise

**Original claim:** "There is no single place to ask: *What happened to batch X from start to finish?*"

**Correction:** OIC's Activity Stream DOES allow searching by `pbatchId` across integrations. The problem is more specific: the result is a list of **unlinked instances with no ordering, no duration per stage, and no business-level status**. You are manually reconstructing a timeline. The absence is not visibility altogether — it is the absence of **unified, ordered, causal linkage** between hops.

---

### Correction B — `poicintid` Behaviour (Critical Correction)

**Original claim:** "`poicintid` arrives empty in the Batchwise call to `/ponewbatchsplit`"

**Correction:** ❌ Partially wrong.

- For **PO Inbound** Stage Data Load: the trigger (`/ponewbatchsplit`) accepts only `pbatchId` as an inbound query parameter. `poicintid` does NOT come in from the caller — it is generated/assigned **inside** Stage Data Load (from the ATP staging procedure output) and then flows **forward** to Validation and beyond.
- For **Receipt Inbound** Stage Data Load: the trigger accepts both `pbatchId` AND `poicintid` as inbound parameters — properly wired from the Batchwise call.
- Both Stage Data Load integrations have `poicintid` as `tracking_var_2`.

The real gap: for PO Inbound, `poicintid` has no value **before** Stage Data Load begins. No hop upstream of Stage Load (Data Extraction, Batchwise Processing) can use it as a correlation key. The fix is to generate it at Data Extraction, not Stage Load.

> ✅ Evidence: `callpostagerest_REQUEST.jca` (PO): `QueryParameters=pbatchId, poicintid`. Trigger `.jca` for PO: `pbatchId` only. Receipt trigger: `pbatchId, poicintid`.

---

### Correction C — Inner `ehStop` Severity Understated (Critical Correction)

**Original claim:** "Inner catch-all uses `ehStop` (silent stop). The outer catch-all then fires the email."

**Correction:** ⚠️ The outer catch-all does NOT fire after an inner `ehStop`. Confirmed from both Batchwise Processing `project.xml` files:

```xml
<try id="t0" name="FetchPO_RCPTNewBatch">
    <invoke "Oracle ATP1"/>          ← ATP call
    <invoke "Integration1"/>         ← REST call to /ponewbatchsplit
    <catchAll id="ta0">
        <ehStop id="eh0"/>           ← inner: NO EMAIL. Integration aborts here.
    </catchAll>
</try>
<stop id="st0"/>
<catchAll id="ta1">                  ← only fires for faults OUTSIDE the try block
    <notification id="n0"/>          ← email (never reached for ATP/REST failures)
    <ehStop id="eh1"/>
</catchAll>
```

**Consequence:** All realistic business failures (ATP procedure error, REST call to `/ponewbatchsplit` timeout, XSL fault) are caught by the inner handler → `ehStop` → **no email notification is sent**. The failure is only visible in the OIC Activity Stream. For the Batchwise Processing integrations (`IFP_EAM_PO_BATCH_PROCE_INTEG` and `IFP_EAM_PO_RECE_BATC_PROC_INTE`), **there is no escalation at all for the most common failure modes**. My previous statement that "email is the ONLY error escalation mechanism" was actually **over-optimistic**.

> ✅ Evidence: `IFP_EAM_PO_BATCH_PROCE_INTEG/PROJECT-INF/project.xml` lines 475–481; `IFP_EAM_PO_RECE_BATC_PROC_INTE/PROJECT-INF/project.xml` lines 312–318.

---

### Correction D — `parameter1` in Tracking Cannot Be Fixed by Renaming Alone

**Original claim:** "Assign meaningful secondary/tertiary tracking variable names across all integrations."

**Correction:** For Application Event-triggered integrations (`IFP_EAM_PO_BATCH_PROCE_INTEG` etc.), the OIC tracking variable `name` field is populated with `parameter1` because the **event payload schema** itself uses `parameter1`, `parameter2`, `parameter3` as structural element names. The XPath in the tracking definition is `/execute/request-wrapper/parameter1`. Renaming the `<name>` label in `project.xml` changes the display label in the OIC console but the underlying data is still the raw event parameter value. **Full resolution requires redefining the Application Event type** with semantic field names (`batchId`, `processType`, `runId`).

> ✅ Evidence: `IFP_EAM_PO_BATCH_PROCE_INTEG/PROJECT-INF/project.xml` line 134–135: `<name>parameter1</name>` with `<xpath>/execute/request-wrapper/parameter1</xpath>`.

---

### Missing Finding 1 — `tracking_var_2` and `tracking_var_3` are Completely Unmapped in Data Extraction

**Finding:** In `IFP_EAM_PO_DATA_EXTRA_INTEG`, `tracking_var_2` has `<ns2:name>tracking_var_2</ns2:name>` and **no `<ns2:xpath>` element**. `tracking_var_3` is identical. Both tracking slots are defined but point to nothing — no value is ever populated.

> ✅ Evidence: `IFP_EAM_PO_DATA_EXTRA_INTEG/PROJECT-INF/project.xml` lines 329–346.

---

### Missing Finding 2 — `UPDATEERRORINEAMHEXAGON` Has the Same Dual-Publisher Issue

**Finding:** Previously called out for `CALLPRIORITYWISEPORCPT` and `TRIGGERNEWPOBATCH`, but missed for `UPDATEERRORINEAMHEXAGON`. In the Receipt pipeline:
- Published by `IFP_EAM_PO_IMPORT_INTEGR` (main Import — on error path)
- Published by `IFP_EAM_UPDA_MESS_IN_EAM_HEXA_RE` (reprocess via `/updatemessage`)
- Subscribed to by `IFP_EAM_UPDA_MESS_IN_EAM_HEXA` (EAM Update)

All three event types (`CALLPRIORITYWISEPORCPT`, `TRIGGERNEWPOBATCH`, `UPDATEERRORINEAMHEXAGON`) have this same dual-publisher/single-subscriber ambiguity.

> ✅ Evidence: `info.json` for `IFP_EAM_PO_IMPORT_INTEGR` (Receipt): `eventtypes: ["UPDATEERRORINEAMHEXAGON|1|pub"]`; `IFP_EAM_UPDA_MESS_IN_EAM_HEXA_RE`: `eventtypes: ["UPDATEERRORINEAMHEXAGON|1|pub"]`

---

### Missing Finding 3 — ATP Logging Cascade Risk in Catch Handler

**Finding:** The proposed design of calling `FAIL_STAGE()` inside every catch-all handler has a failure mode: if the root cause is the **ATP connection itself** (pool exhausted, ATP unavailable), the ATP logging call inside the catch handler will also fail, generating a secondary fault inside the fault handler.

**Required pattern:**
```
catchAll {
    try {
        ATP: FAIL_STAGE(...)
    } catch {
        // ATP logging itself failed — fall through to email
    }
    email notification
    ehStop
}
```

---

### Missing Finding 4 — ATP Credential Fragmentation is Worse Than Described

**Finding:** There are **five separate ATP connection names** across the five projects: `IFP_PO_ATP_DB_CONNEC`, `IFP_PO_RECEI_ATP_DB_CONNE`, `IFP_COMMON_ATP_DB_CONNEC`, AP Freight ATP connection, `IFP_APINV_ATP_DB_CONNEC`. If all point to the same physical ATP instance and the credential rotates, every project must be independently updated. In OIC there is no native connection inheritance.

---

### Missing Finding 5 — `poicintid` Value Source is Undocumented and Unguarded

**Finding:** Since `poicintid` is assigned **inside** Stage Data Load for PO Inbound (not from the inbound trigger), its generation is buried in XSL assign steps or ATP procedure outputs. If the ATP procedure changes or the assignment is removed, `poicintid` silently becomes null in all downstream tracking — **undetected**, because no integration validates it on entry.

Any correlation ID implementation must include a validation gate: if null on entry to a stage, either generate a new one (with a logged warning) or fail-fast with a clear error.

---

### Revised Summary: Additional Items for "What I Would Do Differently"

| # | Item | Basis |
|---|---|---|
| A | Redefine Application Event types with semantic field names — tracking var rename alone is insufficient | Confirmed from event payload structure in project.xml |
| B | Replicate `IFP_AP_INVOI_OUTBO_INTEG_SF`'s named `wsdlFault` handler pattern to all integrations | Confirmed — this integration already has the target pattern |
| C | Fix `UPDATEERRORINEAMHEXAGON` dual-publisher ambiguity (same issue as `CALLPRIORITYWISEPORCPT`) | Confirmed from info.json event type entries |
| D | Add null guard on `pipelineRunId` propagation — never forward a null correlation ID | Confirmed by PO Inbound trigger structure |
| E | ATP logging in catch handler must have its own fallback — avoid cascade failure | Architectural risk identified in design |
| F | Rotating ATP credentials requires updating 5 separate connections — consolidate | Confirmed from 5 distinct ATP connection names |
| G | Move all critical business logic to **outer** try scope so email notification fires for all business failures | Confirmed by inner `ehStop` analysis — highest priority fix |
