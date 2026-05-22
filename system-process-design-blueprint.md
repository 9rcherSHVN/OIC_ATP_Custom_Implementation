# OIC Integration Layer — System Process Design Blueprint

**Date:** May 8, 2026  
**Scope:** All 5 OIC projects — `IFP_PO_INBOUN_INTEGR_ATP`, `IFP_PO_RECEI_INBOU_INTEG_ATP`, `IFP_AP_INVO_FREI_LINE_UPDA_INTE`, `IFP_AP_INVOIC_OUTBOU_INTEGR1`, `IFP_SUPPLIER_OUTBOUND_INTEGRAT`  
**Design stance:** Modular, single-responsibility integrations; shared horizontal infrastructure; evidence-based recommendations only. Every design decision is traceable to a confirmed finding in the extracted `.car` files.

---

## Table of Contents

1. [Guiding Principles](#1-guiding-principles)
2. [Module Architecture](#2-module-architecture)
3. [Module 1 — Shared Connection Registry](#3-module-1--shared-connection-registry)
4. [Module 2 — Pipeline Observability (ATP Schema)](#4-module-2--pipeline-observability-atp-schema)
5. [Module 3 — Shared Error Handler Library](#5-module-3--shared-error-handler-library)
6. [System Process ID: `pipelineRunId` Correlation Design](#6-system-process-id-pipelinerunid-correlation-design)
7. [Error Handling Architecture](#7-error-handling-architecture)
8. [Event Design (Decoupled)](#8-event-design-decoupled)
9. [Reprocess Architecture](#9-reprocess-architecture)
10. [Security Design](#10-security-design)
11. [AP Freight — ESS Job Polling Replacement](#11-ap-freight--ess-job-polling-replacement)
12. [Versioning Strategy](#12-versioning-strategy)
13. [Implementation Phases](#13-implementation-phases-risk-ordered)
14. [Accuracy Checks and Corrections](#14-accuracy-checks-and-corrections)

---

## 1. Guiding Principles

| Principle | Applied as |
|---|---|
| **Modular, single-responsibility** | Each integration owns one pipeline stage. Shared logic is extracted into library integrations — never duplicated across projects. |
| **Fail-loud, never fail-silent** | No business logic inside a catch handler that produces only `ehStop`. Every failure produces a logged ATP record AND an email notification. Both paths always execute. |
| **Observable by design** | Every integration hop writes structured stage entries to ATP before and after its logic. Traceability is built-in, not bolted-on. |
| **Correlation-first** | A `pipelineRunId` UUID is generated at pipeline entry via ATP and propagated as a non-optional parameter through every subsequent hop. |
| **Semantic event contracts** | OIC Application Event types use named payload fields (`batchId`, `processType`, `runId`) — not positional `parameter1/2/3`. |
| **Version-controlled releases** | Breaking interface changes increment the major version. Backward-compatible additions increment minor. No in-place replacement at `01.00.0000`. |

---

## 2. Module Architecture

The current implementation has one monolithic project per pipeline with no shared infrastructure. The recommended model introduces three **horizontal shared modules** consumed by five **vertical pipeline modules**, plus one **reprocess dispatcher** per pipeline.

```
┌─────────────────────────────────────────────────────────────────────┐
│                  Shared Infrastructure Layer                         │
│                                                                     │
│  Module 1                Module 2                 Module 3          │
│  Shared Connection       Pipeline Observability   Shared Error      │
│  Registry                (ATP IFP_APP schema)     Handler Library   │
│  (OIC Connections)       PIPELINE_RUN             (Library Integ.)  │
│                          PIPELINE_STAGE_LOG                         │
└───────────┬──────────────────────┬────────────────────┬────────────┘
            │                      │                    │
            ▼                      ▼                    ▼
┌───────────────────────────────────────────────────────────────────┐
│                     Pipeline Modules (OIC Projects)               │
│                                                                   │
│  Module 4            Module 5           Module 6                  │
│  PO Inbound          Receipt Inbound    AP Freight                │
│  IFP_PO_INBOUN       IFP_PO_RECEI      IFP_AP_INVO_FREI          │
│  _INTEGR_ATP         _INBOU_INTEG_ATP  _LINE_UPDA_INTE            │
│                                                                   │
│  Module 7                    Module 8                             │
│  AP Invoice Outbound         Supplier Outbound                    │
│  IFP_AP_INVOIC_OUTBOU        IFP_SUPPLIER_OUTBOUND                │
│  _INTEGR1                    _INTEGRAT                            │
└───────────────────────────────────────────────────────────────────┘
            │
            ▼
┌───────────────────────────────────────────────────────────────────┐
│           Module 9 — Unified Reprocess Dispatcher                 │
│           (one per pipeline: PO and Receipt)                      │
│           POST /reprocess — stage-aware CBR routing               │
└───────────────────────────────────────────────────────────────────┘
```

**Accuracy check:** The five pipeline projects are confirmed from the five extracted `.car` files. The three shared modules and the Reprocess Dispatcher are new design components — they do not currently exist. The Reprocess Dispatcher replaces seven fragmented reprocess integrations confirmed from `info.json` entries across both PO and Receipt projects.

---

## 3. Module 1 — Shared Connection Registry

### Current Problem (Confirmed)

Five separate ATP connection names — `IFP_PO_ATP_DB_CONNEC`, `IFP_PO_RECEI_ATP_DB_CONNE`, `IFP_COMMON_ATP_DB_CONNEC`, one AP Freight ATP connection, `IFP_APINV_ATP_DB_CONNEC` — all pointing to the same physical ATP instance (confirmed from `SchemaName=ADMIN` across all JCA files). If the credential rotates, every project must be independently updated. OIC has no native connection inheritance.

### Recommended Connection Consolidation

| Connection Name | Purpose | Used by |
|---|---|---|
| `IFP_SHARED_ATP_DB_CONN` | All ATP calls (application tables, staging, observability) | All 5 pipelines |
| `IFP_ERP_SOAP_CONN` | ERP Integration Service SOAP (FBDI import, ESS status) | PO, Receipt, AP Freight |
| `IFP_ERP_REST_CONN` | ERP REST APIs (PO, invoice, supplier) | PO, Receipt, AP Invoice Outbound |
| `IFP_EAM_REST_CONN` | EAM/Hexagon REST (header/line fetch, writeback) | PO, Receipt, AP Freight, Supplier Outbound |
| `IFP_BIP_SOAP_CONN` | BIP SOAP (validation reports, invoice extract) | AP Invoice Outbound, PO/Receipt Validation |

**Rules:**
- Connections are defined once and referenced by name. Credential rotation touches one connection, not five.
- `privateEndpoint=true` on `IFP_SHARED_ATP_DB_CONN`. ATP traffic routes through the OCI VCN private subnet. **Currently confirmed `false` from all ATP JCA files.**
- All application tables and packages migrate from schema `ADMIN` to `IFP_APP`. `ADMIN` has database-elevated privileges that are inappropriate for application data. Confirmed: `SchemaName=ADMIN` in all existing ATP JCA files.

---

## 4. Module 2 — Pipeline Observability (ATP Schema)

ATP is already the staging layer. This module extends it as the observability store. No external tooling is required.

### Schema: `IFP_APP`

```sql
-- Table 1: One row per pipeline run (first-run or reprocess)
CREATE TABLE IFP_APP.PIPELINE_RUN (
  PIPELINE_RUN_ID   VARCHAR2(32)    NOT NULL,   -- RAWTOHEX(SYS_GUID()), 32-char hex UUID
  PIPELINE_TYPE     VARCHAR2(30)    NOT NULL,   -- 'PO_INBOUND', 'RECEIPT_INBOUND',
                                               --  'AP_FREIGHT', 'AP_INVOICE_OUTBOUND',
                                               --  'SUPPLIER_OUTBOUND'
  BATCH_ID          VARCHAR2(200),             -- pbatchId — the EAM business key
  INITIATED_BY      VARCHAR2(30)    NOT NULL,  -- 'SCHEDULER', 'MANUAL_REPROCESS',
                                               --  'PRIORITY_REPROCESS', 'CALLBACK_REPROCESS'
  REPROCESS_OF      VARCHAR2(32),             -- FK to original PIPELINE_RUN_ID (NULL = first run)
  STATUS            VARCHAR2(20)    NOT NULL,  -- 'RUNNING', 'COMPLETED', 'FAILED', 'PARTIAL'
  TOTAL_RECORDS     NUMBER,
  SUCCESS_COUNT     NUMBER,
  FAIL_COUNT        NUMBER,
  STARTED_AT        TIMESTAMP       DEFAULT SYSTIMESTAMP NOT NULL,
  COMPLETED_AT      TIMESTAMP,
  CONSTRAINT PK_PIPELINE_RUN      PRIMARY KEY (PIPELINE_RUN_ID),
  CONSTRAINT FK_REPROCESS_OF      FOREIGN KEY (REPROCESS_OF)
    REFERENCES IFP_APP.PIPELINE_RUN (PIPELINE_RUN_ID)
);

-- Table 2: One row per stage entry/exit per run
CREATE TABLE IFP_APP.PIPELINE_STAGE_LOG (
  LOG_ID            NUMBER          GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  PIPELINE_RUN_ID   VARCHAR2(32)    NOT NULL,
  STAGE_CODE        VARCHAR2(50)    NOT NULL,  -- 'DATA_EXTRACTION', 'BATCHWISE',
                                               --  'STAGE_LOAD', 'VALIDATION',
                                               --  'IMPORT', 'CALLBACK', 'EAM_WRITEBACK'
  INTEGRATION_CODE  VARCHAR2(100)   NOT NULL,  -- e.g. 'IFP_EAM_PO_IMPORT_INTEGR'
  OIC_INSTANCE_ID   VARCHAR2(200),             -- OIC integration instance ID for cross-reference
  STATUS            VARCHAR2(20)    NOT NULL,  -- 'STARTED', 'COMPLETED', 'FAILED'
  ERROR_CODE        VARCHAR2(50),              -- structured: 'ATP_DB_ERROR', 'ERP_SOAP_FAULT', etc.
  ERROR_MESSAGE     VARCHAR2(4000),            -- human-readable — never raw getFaultAsString() XML
  INPUT_SUMMARY     VARCHAR2(4000),            -- key inputs as JSON string
  OUTPUT_SUMMARY    VARCHAR2(4000),            -- key outputs (load request IDs, ESS job IDs) as JSON
  STARTED_AT        TIMESTAMP       DEFAULT SYSTIMESTAMP NOT NULL,
  COMPLETED_AT      TIMESTAMP,
  CONSTRAINT FK_PSL_RUN   FOREIGN KEY (PIPELINE_RUN_ID)
    REFERENCES IFP_APP.PIPELINE_RUN (PIPELINE_RUN_ID)
);

CREATE INDEX IDX_PSL_RUN_ID   ON IFP_APP.PIPELINE_STAGE_LOG (PIPELINE_RUN_ID);
CREATE INDEX IDX_PSL_STATUS   ON IFP_APP.PIPELINE_STAGE_LOG (STATUS, STARTED_AT);
CREATE INDEX IDX_PR_BATCH_ID  ON IFP_APP.PIPELINE_RUN (BATCH_ID, PIPELINE_TYPE);
```

### Package: `IFP_APP.XXIFP_PIPELINE_PKG`

```sql
CREATE OR REPLACE PACKAGE IFP_APP.XXIFP_PIPELINE_PKG AS

  -- Call at pipeline entry (scheduler trigger / REST trigger).
  -- Generates pipelineRunId and writes the PIPELINE_RUN header row.
  -- Returns pipelineRunId to propagate through all subsequent hops.
  FUNCTION START_PIPELINE_RUN(
    p_pipeline_type  VARCHAR2,
    p_batch_id       VARCHAR2,
    p_initiated_by   VARCHAR2,
    p_reprocess_of   VARCHAR2 DEFAULT NULL   -- original pipelineRunId for reprocess chains
  ) RETURN VARCHAR2;  -- returns RAWTOHEX(SYS_GUID())

  -- Call at the START of each integration's business logic.
  PROCEDURE START_STAGE(
    p_run_id           VARCHAR2,
    p_stage_code       VARCHAR2,
    p_integration_code VARCHAR2,
    p_oic_instance_id  VARCHAR2 DEFAULT NULL,
    p_input_summary    VARCHAR2 DEFAULT NULL  -- key inputs as JSON string
  );

  -- Call at the END of each integration on the success path.
  PROCEDURE COMPLETE_STAGE(
    p_run_id         VARCHAR2,
    p_stage_code     VARCHAR2,
    p_output_summary VARCHAR2 DEFAULT NULL   -- key outputs as JSON string
  );

  -- Call inside catch handler on the failure path (BEFORE email notification).
  PROCEDURE FAIL_STAGE(
    p_run_id        VARCHAR2,
    p_stage_code    VARCHAR2,
    p_error_code    VARCHAR2,
    p_error_message VARCHAR2
  );

  -- Call when the entire pipeline run concludes.
  PROCEDURE COMPLETE_RUN(
    p_run_id        VARCHAR2,
    p_success_count NUMBER,
    p_fail_count    NUMBER
  );

  PROCEDURE FAIL_RUN(p_run_id VARCHAR2);

END XXIFP_PIPELINE_PKG;
```

### ATP Logging Cascade Guard

> **Critical constraint (identified from error handling analysis):** If ATP itself is the root cause of failure (connection pool exhausted, ATP service unavailable), calling `FAIL_STAGE` inside a catch handler will itself throw a secondary fault. The ATP logging call must be wrapped in its own inner try so the email notification always fires regardless.

```
OUTER catchAll:
  ┌─ inner try ────────────────────────────────────────┐
  │   ATP: XXIFP_PIPELINE_PKG.FAIL_STAGE(...)          │
  │   ATP: XXIFP_PIPELINE_PKG.FAIL_RUN(...)            │
  └─ inner catchAll ───────────────────────────────────┘
      (ATP logging failed — do not rethrow, fall through)
  email notification   ← ALWAYS executes regardless
  ehStop
```

### Operational Support Queries

```sql
-- Q1: Full timeline for a batch from start to finish
SELECT r.pipeline_type, r.batch_id, r.status AS run_status,
       l.stage_code, l.integration_code, l.status AS stage_status,
       l.error_code, l.error_message,
       l.started_at, l.completed_at
FROM IFP_APP.PIPELINE_RUN r
JOIN IFP_APP.PIPELINE_STAGE_LOG l ON l.pipeline_run_id = r.pipeline_run_id
WHERE r.batch_id = :batch_id
ORDER BY l.started_at;

-- Q2: Reprocess genealogy — all attempts for a batch
SELECT pipeline_run_id, initiated_by, status, started_at,
       reprocess_of
FROM IFP_APP.PIPELINE_RUN
WHERE batch_id = :batch_id
ORDER BY started_at;

-- Q3: Batches currently stuck (RUNNING for more than 30 minutes)
SELECT pipeline_run_id, pipeline_type, batch_id, started_at,
       ROUND((SYSTIMESTAMP - started_at) * 24 * 60) AS minutes_running
FROM IFP_APP.PIPELINE_RUN
WHERE status = 'RUNNING'
  AND started_at < SYSTIMESTAMP - INTERVAL '30' MINUTE
ORDER BY started_at;

-- Q4: Error frequency by code in last 7 days
SELECT error_code, COUNT(*) AS occurrences, MAX(started_at) AS last_seen
FROM IFP_APP.PIPELINE_STAGE_LOG
WHERE status = 'FAILED'
  AND started_at > SYSTIMESTAMP - INTERVAL '7' DAY
GROUP BY error_code
ORDER BY occurrences DESC;
```

---

## 5. Module 3 — Shared Error Handler Library

### Current Problem (Confirmed)

Seventeen or more integration instances use `ora:getFaultAsString()` as the email body content — confirmed from `Detailed_Error_Parameterexpr.properties` files across both pipelines. This dumps raw OIC fault XML into the email, which is not actionable for a support engineer.

### Fault Code Taxonomy

All integrations map every caught fault to a structured error code before logging to ATP or sending email. The named `wsdlFault` handler pattern already implemented in `IFP_AP_INVOI_OUTBO_INTEG_SF` (BIP `AccessDeniedException`, `OperationFailedException`, `InvalidParametersException`) is the reference and is extended to all adapter types.

| Fault origin | OIC fault subRole | Error Code | Human message |
|---|---|---|---|
| ATP DB adapter | `serviceInvocationError` | `ATP_DB_ERROR` | ATP stored procedure failed — check connection pool and procedure name |
| ERP SOAP adapter | `ServiceException` | `ERP_SOAP_FAULT` | ERP Integration Service returned a SOAP fault |
| ERP REST adapter | `APIInvocationError` | `ERP_REST_ERROR` | ERP REST API call failed |
| BIP SOAP | `AccessDeniedException` | `BIP_ACCESS_DENIED` | BIP report access denied — check BIP service account |
| BIP SOAP | `OperationFailedException` | `BIP_OP_FAILED` | BIP report execution failed — check report path |
| BIP SOAP | `InvalidParametersException` | `BIP_INVALID_PARAMS` | BIP report received invalid parameters |
| EAM REST | `APIInvocationError` | `EAM_REST_ERROR` | EAM REST call failed — check EAM system availability |
| Collocated ICS call | `APIInvocationError` | `COLLOCATED_CALL_ERROR` | Child integration call failed — check activation state |
| BIP validation rejection | catch-all | `VALIDATION_REJECTED` | BIP validation returned rejection for this batch |
| ESS job failure | `ServiceException` | `ESS_JOB_FAILED` | ERP ESS job failed — check ESS job log |
| Uncategorised | catch-all (residual) | `UNKNOWN_FAULT` | Unexpected fault — see OIC Activity Stream |

### Standard Email Notification Body

Replace `getFaultAsString()` with a curated template, populated from OIC local variables set at the start of each integration:

```
=== IFP Pipeline Failure Notification ===
Pipeline   : {pipelineType}
Batch ID   : {pbatchId}
Run ID     : {pipelineRunId}
Stage      : {stageCode}
Integration: {integrationCode}
Error Code : {errorCode}
Message    : {errorMessage}
Time       : {timestamp}

Audit query:
  SELECT * FROM IFP_APP.PIPELINE_STAGE_LOG
  WHERE PIPELINE_RUN_ID = '{pipelineRunId}'
  ORDER BY STARTED_AT;

OIC Activity Stream: search tracking_var_1 = '{pipelineRunId}'
==========================================
```

Store the template in `IFP_Common_Utility_Lookup` DVM under key `EMAIL_BODY_TEMPLATE`. All integrations reference the DVM for the template text — one change propagates to all.

---

## 6. System Process ID: `pipelineRunId` Correlation Design

### Current Gaps (Confirmed from File Reads)

| Integration | tracking_var_1 | tracking_var_2 | tracking_var_3 | Gap |
|---|---|---|---|---|
| `IFP_EAM_PO_DATA_EXTRA_INTEG` | `startTime` (schedule payload) | No XPath | No XPath | No business correlation at all — only timestamp |
| `IFP_EAM_PO_BATCH_PROCE_INTEG` | `parameter1` (TRIGGERNEWPOBATCH event field) | `parameter2` | No XPath | Positional field names — no semantic meaning |
| `IFP_EAM_PO_STAG_DATA_LOAD_INTE` | `pbatchId` | `poicintid` | No XPath | `poicintid` generated internally — not propagated from upstream |
| `IFP_EAM_PO_RECE_BATC_PROC_INTE` | `p_new_batch_id` (CALLPRIORITYWISEPORCPT field) | `p_new_priority_col` | No XPath | `tracking_var_3` unmapped; no cross-hop correlation key |

> **Accuracy note:** `IFP_EAM_PO_RECE_BATC_PROC_INTE` (Receipt Batchwise) uses named event payload fields `p_new_batch_id` and `p_new_priority_col` from the `CALLPRIORITYWISEPORCPT` event type — these are already semantically meaningful. This is different from `IFP_EAM_PO_BATCH_PROCE_INTEG` (PO Batchwise) which uses the positional `parameter1`/`parameter2` fields from `TRIGGERNEWPOBATCH`. The event type redefinition requirement for semantic field names applies specifically to `TRIGGERNEWPOBATCH`, not to `CALLPRIORITYWISEPORCPT`.

### Design: `pipelineRunId` Generation

**Why ATP `SYS_GUID()`, not `ora:getInstanceId()`:**  
`ora:getInstanceId()` returns the OIC integration instance ID. Each integration hop in the pipeline gets a different instance ID — cross-hop correlation is impossible with it. `RAWTOHEX(SYS_GUID())` called inside `START_PIPELINE_RUN` via the ATP adapter returns a stable 32-character hex UUID that belongs to the entire pipeline run, not a single hop. It is generated once and propagated forward to all subsequent hops.

### Generation Point

`pipelineRunId` is generated at the **pipeline entry integration** — the first integration that fires in the pipeline:

| Pipeline | Entry integration | Trigger type |
|---|---|---|
| PO Inbound | `IFP_EAM_PO_DATA_EXTRA_INTEG` | OIC Scheduler |
| Receipt Inbound | `UPDA_IFP_EAM_PO_RECE_DATA_EXTR_I` | OIC Scheduler |
| AP Freight | `IFP_AP_INVO_FREI_LINE_UPDA_INTE` main | OIC Scheduler |
| AP Invoice Outbound | `IFP_AP_INVOI_OUTBO_INTEG_SF` | OIC Scheduler |
| Supplier Outbound | `TEST_IFP_SCHED_SUPPL_OUTBO_INTEG` | OIC Scheduler |

**First action in every entry integration:**
```
invoke ATP: pipelineRunId := XXIFP_PIPELINE_PKG.START_PIPELINE_RUN(
  p_pipeline_type => 'PO_INBOUND',
  p_batch_id      => null,           -- may be null at extraction; populated after first ATP fetch
  p_initiated_by  => 'SCHEDULER'
);
-- pipelineRunId is now a local OIC string variable
-- Immediately assign to tracking_var_1
```

### Propagation Model (PO Inbound — confirmed 7-hop pipeline)

```
Scheduler
  │
  ▼
[1] Data Extraction (IFP_EAM_PO_DATA_EXTRA_INTEG)
    ATP: START_PIPELINE_RUN → pipelineRunId
    ATP: START_STAGE('DATA_EXTRACTION')
    EAM REST: FetchHdrGridDetails, FetchLineGridDetails
    ATP: EAMHdrGridInsert, EAMLineGridInsert (staging)
    ATP: COMPLETE_STAGE / FAIL_STAGE
    Publish event TRIGGERNEWPOBATCH_V2 {batchId, pipelineRunId, processType='NORMAL'}
    │
    ▼ (Application Event, async fan-out)
[2] Batchwise Processing (IFP_EAM_PO_BATCH_PROCE_INTEG)
    Subscribe: TRIGGERNEWPOBATCH_V2 → extract pipelineRunId
    ATP: START_STAGE('BATCHWISE')
    ATP: ponewbatchfetch (fetch batch list per loop iteration)
    ATP: updatebatchstatusasDraft
    Collocated call → Stage Data Load + pipelineRunId
    ATP: COMPLETE_STAGE / FAIL_STAGE
    │
    ▼ (Collocated ICS call, synchronous)
[3] Stage Data Load (IFP_EAM_PO_STAG_DATA_LOAD_INTE)
    Receive pipelineRunId as query parameter
    ATP: START_STAGE('STAGE_LOAD')
    ATP: mainstageload (staging proc)
    Collocated call → Validation + pipelineRunId
    ATP: COMPLETE_STAGE / FAIL_STAGE
    │
    ▼ (Collocated ICS call, synchronous)
[4] Validation (IFP_EAM_PO_VALIDA_INTEGR)
    ATP: START_STAGE('VALIDATION')
    BIP SOAP: validation report
    ATP: COMPLETE_STAGE / FAIL_STAGE
    │
    ▼
[5] Import (IFP_EAM_PO_IMPORT_INTEGR)
    ATP: START_STAGE('IMPORT')
    ERP SOAP: importBulkData / changePurchaseOrder / cancelPurchaseOrder
    ATP: COMPLETE_STAGE / FAIL_STAGE
    │
    ▼
[6] Callback (IFP_PO_IMPOR_CALLB)
    ATP: START_STAGE('CALLBACK')
    ERP REST: callback results
    ATP: COMPLETE_STAGE / FAIL_STAGE
    │
    ▼
[7] EAM Writeback (IFP_EAM_STATU_UPDAT_INTEG_M)
    ATP: START_STAGE('EAM_WRITEBACK')
    EAM REST: status update
    ATP: COMPLETE_STAGE / FAIL_STAGE
    ATP: COMPLETE_RUN / FAIL_RUN
```

### OIC Tracking Variable Standard (All Integrations, All Pipelines)

| Slot | Value | Constraint |
|---|---|---|
| `tracking_var_1` (primary) | `pipelineRunId` | Always populated; never null; searchable across all hops in OIC Activity Stream |
| `tracking_var_2` | `pbatchId` | The EAM business key |
| `tracking_var_3` | Literal stage name | Static assign at integration start: `'DATA_EXTRACTION'`, `'BATCHWISE'`, etc. |

**Constraint for event-triggered hops:** For `TRIGGERNEWPOBATCH` (PO Batchwise), the event payload fields are named `parameter1`/`parameter2`. Renaming the tracking variable display label in `project.xml` changes only what appears in the OIC console — the underlying XPath `/execute/request-wrapper/parameter1` still resolves to that field value. Full resolution requires redefining the `TRIGGERNEWPOBATCH` event type with named fields. This does not apply to `CALLPRIORITYWISEPORCPT` (Receipt Batchwise), which already uses named fields.

### Propagation Mechanism by Hop Type

| Hop type | Mechanism | Notes |
|---|---|---|
| Application Event publish → subscribe | Add `runId` field to event payload | Requires event type redefinition for `TRIGGERNEWPOBATCH` |
| Collocated ICS call | Add `pipelineRunId` as query/body parameter in collocated WSDL contract | `InvokePOStageloadintegration_REQUEST.wsdl`, `callPORCPTstagedataloading_REQUEST.wsdl` |
| REST inbound trigger (reprocess) | Add `pipelineRunId` as optional query parameter | Non-breaking addition |
| REST outbound to EAM | Pass as HTTP header `X-Pipeline-Run-Id` | EAM ignores unknown headers; does not break EAM contract |

---

## 7. Error Handling Architecture

### The Core Problem (Confirmed from `project.xml` Orchestration Blocks)

Both Batchwise Processing integrations — PO (`IFP_EAM_PO_BATCH_PROCE_INTEG`) and Receipt (`IFP_EAM_PO_RECE_BATC_PROC_INTE`) — share the same fatal orchestration pattern:

```xml
<!-- CONFIRMED: PO Batchwise (IFP_EAM_PO_BATCH_PROCE_INTEG/PROJECT-INF/project.xml) -->
<!-- CONFIRMED: Receipt Batchwise (IFP_EAM_PO_RECE_BATC_PROC_INTE/PROJECT-INF/project.xml) -->
<globalTry>
  <subscribe ... />                         ← event received

  <try name="FetchPOnewbatch">              ← INNER try
    <invoke "Oracle ATP1"/>                 ← ponewbatchfetch / BatchstatusDRAFTupd
    <for>
      <invoke "Oracle ATP2"/>               ← updatebatchstatusasDraft
      <invoke "Integration1"/>              ← collocated Stage Load call
    </for>
    <catchAll>
      <ehStop/>                             ← CONFIRMED: silent abort. No ATP log, no email.
    </catchAll>
  </try>

  <stop/>

  <catchAll>                                ← CONFIRMED: never reached after inner ehStop fires
    <notification/>                         ← email (dead code for all real business failures)
    <ehStop/>
  </catchAll>
</globalTry>
```

**Consequence:** All realistic failures — ATP stored procedure error, REST timeout to Stage Load, for-loop fault — are swallowed by the inner `ehStop`. The outer email handler is architecturally unreachable for all business-level failures in these two integrations.

### Redesigned Orchestration Pattern (Single Outer Try)

```xml
<globalTry>
  <subscribe ... />

  <!-- Step 1: Generate pipelineRunId at pipeline entry -->
  <invoke "ATP: START_PIPELINE_RUN()" → assign local var pipelineRunId />

  <!-- Step 2: Log stage start -->
  <invoke "ATP: START_STAGE('BATCHWISE')" />

  <!-- Step 3: ALL business logic in ONE outer scope — no nesting -->
  <invoke "ATP: ponewbatchfetch" />
  <for>
    <invoke "ATP: updatebatchstatusasDraft" />
    <invoke "Collocated Stage Load + pipelineRunId" />
  </for>

  <!-- Step 4: Log stage completion -->
  <invoke "ATP: COMPLETE_STAGE('BATCHWISE')" />
  <stop/>

  <!-- Single catch — fires for ALL failures in the outer scope -->
  <catchAll>
    <try>
      <invoke "ATP: FAIL_STAGE(errorCode, humanMessage)" />
      <invoke "ATP: FAIL_RUN()" />
    </try>
    <catchAll>
      <!-- ATP logging itself failed (cascade guard) — fall through to email -->
    </catchAll>
    <notification/>        ← email ALWAYS fires
    <ehStop/>
  </catchAll>

</globalTry>
```

**Rule:** One catch handler for all business logic. Inner try blocks are used **only** within the outer catch to guard the ATP logging calls against cascade failure. Never nest business logic inside inner try/catch.

### Named Fault Handler Pattern (Extending the Reference Pattern)

`IFP_AP_INVOI_OUTBO_INTEG_SF` is the only integration in the suite that already implements named `wsdlFault` handlers — for BIP SOAP faults (`AccessDeniedException`, `OperationFailedException`, `InvalidParametersException`) and for EAM REST `APIInvocationError`. This is the reference pattern to extend to all pipeline integrations:

```
Per adapter invocation — register specific wsdlFault handlers:

  invoke "ERP SOAP importBulkData"
    fault: ServiceException          → errorCode = 'ERP_SOAP_FAULT'

  invoke "ATP DB proc"
    fault: serviceInvocationError    → errorCode = 'ATP_DB_ERROR'

  invoke "BIP SOAP runReport"
    fault: AccessDeniedException     → errorCode = 'BIP_ACCESS_DENIED'
    fault: OperationFailedException  → errorCode = 'BIP_OP_FAILED'
    fault: InvalidParametersException → errorCode = 'BIP_INVALID_PARAMS'

  invoke "EAM REST"
    fault: APIInvocationError        → errorCode = 'EAM_REST_ERROR'

  invoke "Collocated ICS"
    fault: APIInvocationError        → errorCode = 'COLLOCATED_CALL_ERROR'

  catchAll (residual):               → errorCode = 'UNKNOWN_FAULT'
```

Each named fault handler sets the local `errorCode` variable. The outer catch handler reads this variable when calling `FAIL_STAGE`.

---

## 8. Event Design (Decoupled)

### Current Problem (Confirmed from `info.json` Event Type Entries)

Three event types with dual-publisher / single-subscriber ambiguity — no subtype distinguishes a normal first-run publication from a reprocess publication:

| Event type | Publishers | Subscriber | Ambiguity |
|---|---|---|---|
| `TRIGGERNEWPOBATCH` | `IFP_EAM_PO_DATA_EXTRA_INTEG` (normal) + `IFP_EAM_PO_BATCH_REPRO_INTEG` (reprocess) | `IFP_EAM_PO_BATCH_PROCE_INTEG` | No differentiation between first-run and reprocess |
| `CALLPRIORITYWISEPORCPT` | `UPDA_IFP_EAM_PO_RECE_DATA_EXTR_I` (normal) + `IFP_EAM_PRIORITY_REPROCES` (priority reprocess) + `IFP_PO_RECEI_IMPOR_CALLB_REPRO` (callback reprocess) | `IFP_EAM_PO_RECE_BATC_PROC_INTE` | Three publishers, no routing distinction |
| `UPDATEERRORINEAMHEXAGON` | `IFP_EAM_PO_IMPORT_INTEGR` (import error path) + `IFP_EAM_UPDA_MESS_IN_EAM_HEXA_RE` (reprocess) | `IFP_EAM_UPDA_MESS_IN_EAM_HEXA` | Import errors and reprocess attempts are indistinguishable |

### Recommended Design: Separate Event Types per Run Type

Introduce distinct event types per run intent. This eliminates ambiguity at the event level — subscriber intent is explicit:

| Replace | With (first-run) | With (reprocess) |
|---|---|---|
| `TRIGGERNEWPOBATCH` | `IFP_PO_BATCH_SCHEDULED_V2` | `IFP_PO_BATCH_REPROCESS_V2` |
| `CALLPRIORITYWISEPORCPT` | `IFP_RCPT_BATCH_SCHEDULED_V2` | `IFP_RCPT_BATCH_REPROCESS_V2` |
| `UPDATEERRORINEAMHEXAGON` | `IFP_EAM_ERROR_UPDATE_V2` (single publisher: Import only) | *(reprocess calls EAM writeback directly via collocated call)* |

### Semantic Event Type Schema

Replace positional field names with named fields in all event type definitions:

```json
{
  "eventType": "IFP_PO_BATCH_SCHEDULED_V2",
  "version": 2,
  "fields": [
    { "name": "batchId",       "type": "string",  "required": true  },
    { "name": "pipelineRunId", "type": "string",  "required": true  },
    { "name": "processType",   "type": "string",  "required": true  }
  ]
}
```

> **Accuracy note:** For `CALLPRIORITYWISEPORCPT`, the current event payload already uses named fields (`p_new_batch_id`, `p_new_priority_col`). The new V2 schema adds `pipelineRunId` and `processType` to these, which is an additive change. For `TRIGGERNEWPOBATCH`, the current event payload uses `parameter1`/`parameter2` — replacing these with named fields is a breaking change requiring coordinated redeployment of all publishers and subscribers.

**Deployment note:** Redefining event type schemas is a breaking change. Coordinated deployment is required: new event types activated, publishers and subscribers migrated, old event types deactivated. This is a Phase 5 change.

---

## 9. Reprocess Architecture

### Current Problem (Confirmed)

Seven reprocess integrations across PO and Receipt, two of which share the URI `/callbackreprocess` with incompatible parameter contracts:

| Project | Integration | Endpoint | Entry stage |
|---|---|---|---|
| PO Inbound | `IFP_PO_REPROCES_INTEGRAT` | `/poreprocess` | Validation |
| PO Inbound | `IFP_EAM_PO_BATCH_REPRO_INTEG` | `/callbackreprocess?importid` | Batchwise dispatch |
| PO Inbound | `IFP_PO_IMPORT_CALLBA_REPROC` | `/callbackreprocess?loadrequestid` | Callback |
| Receipt | `IFP_EAM_PRIORITY_REPROCES` | `/poprioritywisereprocess` | Batchwise dispatch |
| Receipt | `IFP_PO_RECEIP_REPROC_INTEGR` | `/porcptreprocess` | Validation |
| Receipt | `IFP_PO_RECEI_IMPOR_CALLB_REPRO` | `/callbackreprocess?loadrequestid` | Callback |
| Receipt | `IFP_EAM_UPDA_MESS_IN_EAM_HEXA_RE` | `/updatemessage` | EAM writeback |

Two PO integrations and one Receipt integration share `/callbackreprocess` with different parameter contracts — IT support must know which to call and with which parameters based on the failure stage.

### Unified Reprocess Dispatcher

One integration per pipeline: `IFP_PO_REPROCESS_DISPATCHER` and `IFP_RCPT_REPROCESS_DISPATCHER`.

**Endpoint:** `POST /reprocess`

**Request body:**
```json
{
  "originalPipelineRunId": "a3f9b2c1...",
  "entryStage":            "VALIDATION",
  "pbatchId":              "BT-20260508-001",
  "importId":              "12345",
  "loadRequestId":         "67890"
}
```

`entryStage` valid values: `VALIDATION` | `IMPORT` | `CALLBACK` | `EAM_WRITEBACK`

**Internal routing via CBR on `entryStage`:**

```
POST /reprocess
  │
  ├─ Generate new pipelineRunId
  │    ATP: START_PIPELINE_RUN(
  │           p_initiated_by  = 'MANUAL_REPROCESS',
  │           p_reprocess_of  = originalPipelineRunId
  │         )
  │
  ├─ CBR: entryStage = 'VALIDATION'   → Collocated call: Validation integration
  ├─ CBR: entryStage = 'IMPORT'       → Collocated call: Import integration
  ├─ CBR: entryStage = 'CALLBACK'     → Collocated call: Callback integration
  └─ CBR: entryStage = 'EAM_WRITEBACK' → Collocated call: EAM Writeback integration
```

### Reprocess Genealogy in ATP

```
PIPELINE_RUN: a3f9b2c1  PO_INBOUND  BT-001  FAILED    SCHEDULER         (original run)
  └── PIPELINE_RUN: d7e1a4f9  PO_INBOUND  BT-001  FAILED    MANUAL_REPROCESS  (reprocess 1)
        └── PIPELINE_RUN: 8c2b3d5e  PO_INBOUND  BT-001  COMPLETED MANUAL_REPROCESS  (reprocess 2)
```

Query for reprocess genealogy:
```sql
SELECT pipeline_run_id, initiated_by, status, started_at, reprocess_of
FROM IFP_APP.PIPELINE_RUN
WHERE batch_id = :batch_id
  AND pipeline_type = 'PO_INBOUND'
ORDER BY started_at;
```

---

## 10. Security Design

### External REST Trigger Endpoints

**Current state (confirmed):** All external endpoints have `securityPolicy=BASIC_AUTH`, `privateEndpoint=false`.

| Endpoint | Current | Recommended |
|---|---|---|
| `/poreprocess`, `/porcptreprocess` | `BASIC_AUTH` | OAuth 2.0 client credentials |
| `/poprioritywisereprocess` | `BASIC_AUTH` | OAuth 2.0 client credentials |
| `/supplieroutbound` | `BASIC_AUTH` | OAuth 2.0 client credentials |
| `/reprocess` (new unified) | n/a | OAuth 2.0 client credentials |

**OAuth 2.0 implementation:** Configure OIC as an OAuth Resource Server via the OCI Identity domain. External callers (EAM trigger, operational reprocess tools) obtain access tokens from the OCI Identity domain and pass as `Authorization: Bearer <token>`. OIC validates the token signature and scope before routing.

### ATP Database Connection

Set `privateEndpoint=true` on `IFP_SHARED_ATP_DB_CONN`. Routes ATP traffic through the OCI VCN private endpoint — currently confirmed `false` on all ATP JCA files.

### Collocated Adapter `securityPolicy=NONE`

Confirmed across 11 collocated `.jca` files. This is the correct and expected setting for OIC collocated adapters (intra-OIC-instance calls, no external network traversal). **Document explicitly in runbooks** — this is intentional, not a security gap.

### Schema Privilege Separation

| What | Current | Recommended |
|---|---|---|
| Application tables schema | `ADMIN` (elevated privileges) | `IFP_APP` (application-only privileges) |
| Packages | `ADMIN.XXIFP_POINBOUND_INT_PKG`, `ADMIN.XXIFP_PORCPT_INB_INT_PKG` | `IFP_APP.XXIFP_PIPELINE_PKG` |
| OIC ATP connection user | `ADMIN` | Dedicated `IFP_APP` user with SELECT/INSERT/UPDATE/EXECUTE on `IFP_APP` objects only |

---

## 11. AP Freight — ESS Job Polling Replacement

### Current Problem (Confirmed)

`IFP_SUBMI_PAYAB_IMPOR_INVOI_INTE` uses a `while` loop polling ESS job status:

- Adapter: `application_132` / `GetESSJobStatus_InWhile` — confirmed from `GetESSJobStatus_InWhile_REQUEST.jca`
- While condition: `$GetESSJobStatus_InWhile/types:getESSJobStatusResponse/types:result` — confirmed from `expr.properties`
- Pattern: OIC integration instance remains open and holds the ERP SOAP connection for the full duration of the ESS job
- Risk: OIC instance timeout on long-running Payables import jobs; connection pool starvation under concurrent runs

### Recommended Pattern: ERP Business Event Callback

```
Step 1: Submit ESS job
  IFP_SUBMI_PAYAB_IMPOR_INVOI_INTE submits Payables import ESS job
  → receives loadRequestId
  → ATP: START_STAGE('IMPORT', output_summary = {loadRequestId})
  → integration instance completes immediately (fire-and-forget)

Step 2: ERP Business Event fires when ESS job completes
  ERP Cloud publishes: oracle/apps/ess/core/async/JobStatus

Step 3: New integration subscribes
  IFP_ESS_JOB_CALLBACK_INTEG subscribes to ERP Business Event
  → reads loadRequestId from event payload
  → queries ATP: find PIPELINE_STAGE_LOG row matching loadRequestId
  → calls COMPLETE_STAGE or FAIL_STAGE based on ESS result
  → continues pipeline via collocated call to next stage
```

**Benefit:** OIC integration instance is released immediately after ESS job submission. No polling, no OIC instance lock, no timeout risk. The ERP Business Event (`ErpIntegrationService`) adapter is a confirmed available adapter type in OIC, already used in the PO Import integration.

---

## 12. Versioning Strategy

### Current State (Confirmed)

All 20+ integrations across all 5 projects remain at version `01.00.0000`, spanning OIC platform updates from `25.10-EC` to `26.04.0.7-EC`. Any change is an in-place replacement with no rollback path other than re-importing the `.car` file.

### Recommended Versioning Scheme

| Version pattern | Trigger |
|---|---|
| `02.00.0000` | Breaking interface change — new event type fields (e.g. `pipelineRunId` added to event payload), endpoint parameter contract changes |
| `01.01.0000` | Non-breaking addition — new optional query parameter, new ATP logging calls added without changing existing interface |
| `01.00.0001` | Patch — bug fix with no interface change (e.g. fixing inner `ehStop` pattern) |

**Blue/green deployment:** Activate `02.00.0000` alongside `01.00.0000`. External callers are migrated to the new version. Once traffic is confirmed on v2, deactivate v1. OIC supports multiple simultaneously active versions of the same integration code.

**Additional naming fix (confirmed):**

| Current (confirmed from `info.json`) | Recommended |
|---|---|
| `TEST_IFP_SUPPL_OUTBO_INTEG_17702` (`persistedState=ACTIVATED`) | `IFP_SUPPL_OUTBO_INTEG` |
| `TEST_IFP_SCHED_SUPPL_OUTBO_INTEG` (`persistedState=ACTIVATED`) | `IFP_SCHED_SUPPL_OUTBO_INTEG` |
| `IFP_PO_IMPOR_CALLB_16_FEB` (date stamp in code name from `originalICSVersion=26.01.0.16-EC`) | `IFP_PO_IMPOR_CALLBACK` |

---

## 13. Implementation Phases (Risk-Ordered)

### Phase 0 — Immediate (No New Infrastructure Required)

Fix the inner `ehStop` pattern in both Batchwise Processing integrations. Move all business logic to the outer try scope. This is the only change that requires no prerequisite work and eliminates the production operational blind spot.

**Integrations:** `IFP_EAM_PO_BATCH_PROCE_INTEG`, `IFP_EAM_PO_RECE_BATC_PROC_INTE`  
**Risk:** Low  
**Value:** Critical — restores email escalation for the most common failure modes

### Phase 1 — Zero Disruption (Add Without Changing Existing Logic)

1. Create `IFP_APP` schema with `PIPELINE_RUN` and `PIPELINE_STAGE_LOG` tables
2. Create `XXIFP_PIPELINE_PKG` with `START_PIPELINE_RUN`, `START_STAGE`, `COMPLETE_STAGE`, `FAIL_STAGE`
3. Wire `START_PIPELINE_RUN` into the entry integration of each pipeline only (`IFP_EAM_PO_DATA_EXTRA_INTEG` etc.)
4. Add `pipelineRunId` as an optional query parameter to all existing REST endpoint definitions (non-breaking)
5. Fix OIC tracking variable labels: assign `pipelineRunId` to `tracking_var_1`, `pbatchId` to `tracking_var_2`, stage literal to `tracking_var_3`

**Delivers:** 60% of observability benefit. Entry-point pipeline runs are now logged and searchable by `pipelineRunId` in both ATP and OIC Activity Stream.

### Phase 2 — Propagation

6. Propagate `pipelineRunId` through every subsequent hop (collocated WSDL contracts, event payloads, REST outbound headers)
7. Add `START_STAGE` / `COMPLETE_STAGE` calls at every integration boundary (ATP calls at entry/exit, not touching XSL business mapping logic)
8. Add `FAIL_STAGE` in all catch handlers (with ATP cascade guard)

**Delivers:** Full end-to-end traceability. Any batch is fully queryable from one ATP query.

### Phase 3 — Error Handling Upgrade

9. Replace `getFaultAsString()` with structured error code mapping in all `Detailed_Error_Parameterexpr.properties` files
10. Implement named `wsdlFault` handlers per adapter type across all pipeline integrations (using `IFP_AP_INVOI_OUTBO_INTEG_SF` as the template)
11. Update email notification body to structured template

**Delivers:** Actionable, human-readable error notifications.

### Phase 4 — Architecture Improvements

12. Replace ESS polling while-loop with ERP Business Event callback integration (`IFP_ESS_JOB_CALLBACK_INTEG`)
13. Deploy Unified Reprocess Dispatcher (`IFP_PO_REPROCESS_DISPATCHER`, `IFP_RCPT_REPROCESS_DISPATCHER`)
14. Consolidate five ATP connections to `IFP_SHARED_ATP_DB_CONN`
15. Upgrade external REST endpoints from Basic Auth to OAuth 2.0

**Delivers:** Security posture improvement, performance improvement, simplified operations.

### Phase 5 — Event and Naming Cleanup

16. Redefine `TRIGGERNEWPOBATCH` event type with semantic field names (`batchId`, `pipelineRunId`, `processType`)
17. Separate dual-publisher event types: `IFP_PO_BATCH_SCHEDULED_V2` / `IFP_PO_BATCH_REPROCESS_V2`, etc.
18. Rename `TEST_` integrations and `IFP_PO_IMPOR_CALLB_16_FEB`
19. Migrate ATP tables and packages from `ADMIN` schema to `IFP_APP`

**Delivers:** Clean architecture. High coordination cost — requires simultaneous publisher and subscriber deployment.

---

## 14. Accuracy Checks and Corrections

All design decisions are based on direct reads of the extracted files. Key facts confirmed and corrections applied before this document was written:

### Confirmed Facts

| Fact | Evidence source |
|---|---|
| Inner `ehStop` in PO Batchwise swallows all business failures | `IFP_EAM_PO_BATCH_PROCE_INTEG/PROJECT-INF/project.xml` — `orchestration` block |
| Inner `ehStop` in Receipt Batchwise — identical pattern | `IFP_EAM_PO_RECE_BATC_PROC_INTE/PROJECT-INF/project.xml` — `orchestration` block |
| Data Extraction `tracking_var_1 = startTime` (schedule), `tracking_var_2` and `tracking_var_3` have no XPath | `IFP_EAM_PO_DATA_EXTRA_INTEG/PROJECT-INF/project.xml` — `messageTracker` processor |
| PO Batchwise `tracking_var_1 = parameter1`, `tracking_var_2 = parameter2` (TRIGGERNEWPOBATCH event positional fields) | `IFP_EAM_PO_BATCH_PROCE_INTEG/PROJECT-INF/project.xml` — `messageTracker` processor |
| Receipt Batchwise `tracking_var_1 = p_new_batch_id`, `tracking_var_2 = p_new_priority_col` (CALLPRIORITYWISEPORCPT named fields) | `IFP_EAM_PO_RECE_BATC_PROC_INTE/PROJECT-INF/project.xml` — `messageTracker` processor |
| `poicintid` is NOT an inbound trigger parameter for PO Stage Data Load; it is generated internally | PO Stage Data Load trigger JCA: `QueryParameters=pbatchId, poicintid`; PO Batchwise collocated call WSDL structure |
| `poicintid` IS an inbound parameter for Receipt Stage Data Load | Receipt Batchwise collocated call to `callPORCPTstagedataloading` — confirmed from project.xml |
| ESS job polling: while loop with `GetESSJobStatus_InWhile` | `GetESSJobStatus_InWhile_REQUEST.jca` + `expr.properties` XPath condition `$GetESSJobStatus_InWhile/types:getESSJobStatusResponse/types:result` |
| Both `TEST_` Supplier Outbound integrations are `persistedState=ACTIVATED` | `TEST_IFP_SUPPL_OUTBO_INTEG_17702/info.json`, `TEST_IFP_SCHED_SUPPL_OUTBO_INTEG/info.json` |
| `IFP_AP_INVOI_OUTBO_INTEG_SF` is the only integration with named `wsdlFault` handlers | Confirmed from `project.xml` — BIP: `AccessDeniedException`, `OperationFailedException`, `InvalidParametersException`; EAM REST: `APIInvocationError` |
| All ATP JCA files use `SchemaName=ADMIN` | Confirmed across all five project ATP JCA files |
| Five distinct ATP connection names across five projects | `IFP_PO_ATP_DB_CONNEC`, `IFP_PO_RECEI_ATP_DB_CONNE`, `IFP_COMMON_ATP_DB_CONNEC`, AP Freight ATP, `IFP_APINV_ATP_DB_CONNEC` |
| All external REST triggers use `securityPolicy=BASIC_AUTH`, `privateEndpoint=false` | All external trigger JCA files |
| All collocated calls use `securityPolicy=NONE` | All collocated ICS JCA files — 11 confirmed instances |

### Applied Corrections

| Correction | Source of imprecision | Corrected statement |
|---|---|---|
| Event type semantic naming requirement | Earlier draft implied both `TRIGGERNEWPOBATCH` and `CALLPRIORITYWISEPORCPT` use positional `parameter1/2/3` names | `CALLPRIORITYWISEPORCPT` already uses named fields (`p_new_batch_id`, `p_new_priority_col`). Only `TRIGGERNEWPOBATCH` uses positional names. Event type redefinition is required only for `TRIGGERNEWPOBATCH`. |
| `poicintid` generation point | Earlier draft stated `poicintid` "arrives empty" in the Batchwise call | PO Stage Data Load: `poicintid` is NOT an inbound parameter — it is assigned internally (from ATP proc output). Receipt Stage Data Load: `poicintid` IS an inbound parameter from the Batchwise collocated call. The upstream gap is that neither Data Extraction nor Batchwise Processing has `poicintid` — only Stage Data Load onwards. |
| Inner `ehStop` scope | Earlier draft stated outer email "fires for outer-level faults" implying some failures reach it | Confirmed: for both Batchwise Processing integrations, ALL realistic business failures (ATP proc error, REST timeout, for-loop fault) are caught by the inner `ehStop`. The outer email handler is never reached for any of these. It only fires for a fault that occurs outside the try block entirely — which has no realistic trigger path in the current flow. |

---

## 15. Architecture Design Audit

**Audit Date:** May 8, 2026  
**Scope:** Independent audit of this blueprint's factual accuracy, design soundness, internal consistency, and completeness — all findings cross-referenced against the extracted `.car` file contents.

---

### Audit Finding 1 — FACTUAL ERROR: "OIC Has No Native Connection Inheritance" Is Incorrect

**Blueprint claim (Section 3, Module 1):**
> "OIC has no native connection inheritance."

**Evidence from extracted files:**  
Every connection XML file across all 5 projects contains `<hasOverride>true</hasOverride>` and a `<remoteResource>` element referencing a shared project:

```xml
<hasOverride>true</hasOverride>
<remoteResource>
  <code>IFP_COMMON_ATP_DB_CONNEC</code>
  <projectCode>IFP_COMMON_CONNECTION</projectCode>
</remoteResource>
```

All ATP connections (`IFP_PO_ATP_DB_CONNEC`, `IFP_PO_RECEI_ATP_DB_CONNE`, `IFP_APINV_ATP_DB_CONNEC`, `IFP_SUPP_OUTBOU_INTEGR_ATP`) reference the same remote connection `IFP_COMMON_ATP_DB_CONNEC` from a shared `IFP_COMMON_CONNECTION` project. Similarly:

| Local connection | Remote source in `IFP_COMMON_CONNECTION` |
|---|---|
| `IFP_PO_EAM_REST_CONNEC`, `IFP_APINV_EAM_REST_CONNEC`, etc. | `IFP_COMMON_EAM_REST_CONNEC` |
| `IFP_PO_OIC_REST_CONNEC`, `IFP_APFR_OIC_REST_CONNEC`, etc. | `IFP_COMMON_OIC_REST_CONNEC` |
| `IFP_PO_ORACL_ERP_REST_CONNE`, etc. | `IFP_COMMO_ORACL_ERP_REST_CONNE` |
| `IFP_PO_ORACL_BIP_SOAP_CONNE`, etc. | `ORACLE_COMMON_BIP_SOAP_CONNEC` |
| `IFP_PO_ORACL_ERP_CLOUD_CONNE`, etc. | `IFP_COMMO_ORACL_ERP_CLOUD_CONNE` |

**Impact on Module 1 design:**  
OIC's project-level connection override mechanism IS a form of connection inheritance. A shared `IFP_COMMON_CONNECTION` project already exists and serves as the connection registry. The five project-local connection names are overrides of the shared definitions.

- If the remote connection's credential is updated in `IFP_COMMON_CONNECTION` and the local overrides do not specify their own credential values, the update propagates automatically.
- The Module 1 recommendation to consolidate to `IFP_SHARED_ATP_DB_CONN` is partially redundant — the consolidation already exists at the `IFP_COMMON_CONNECTION` level.
- **However:** The local override names still differ per project (creating confusion), and whether the local overrides store independent credential copies vs. inherit from the remote requires runtime verification on the OIC instance. The `<percentageComplete>13</percentageComplete>` and `<status>IN_PROGRESS</status>` on the ATP connections suggest the connection values (host, wallet, etc.) may not be fully configured in the local overrides, meaning they likely inherit from the remote — but this is a runtime state, not definitively confirmable from `.car` exports alone.

**Required correction:** Replace "OIC has no native connection inheritance" with an accurate description of the existing `IFP_COMMON_CONNECTION` project and the override mechanism. The recommendation should focus on standardising local override naming and verifying credential delegation, not on creating a new shared connection registry from scratch.

---

### Audit Finding 2 — FACTUAL ERROR: AP Freight Does Not Have a Dedicated ATP Connection

**Blueprint claim (Section 3):**
> "Five separate ATP connection names — `IFP_PO_ATP_DB_CONNEC`, `IFP_PO_RECEI_ATP_DB_CONNE`, `IFP_COMMON_ATP_DB_CONNEC`, one AP Freight ATP connection, `IFP_APINV_ATP_DB_CONNEC`"

**Evidence from extracted files:**  
The AP Freight project (`IFP_AP_INVO_FREI_LINE_UPDA_INTE`) contains exactly 5 connection files:
- `IFP_APFR_OIC_REST_CONNEC.xml` (REST to OIC)
- `IFP_APFR_ORAC_ERP_INTE_SERV_SOAP.xml` (ERP SOAP)
- `IFP_APFR_ORACL_BIP_SOAP_CONNE.xml` (BIP SOAP)
- `IFP_APFR_ORACLE_REST_CONNEC.xml` (ERP REST)
- `PRESEEDED_COLLOCATED_CONN_1741.xml` (Collocated ICS)

**No ATP database connection file exists in the AP Freight project.** The `SchemaName=ADMIN` grep across AP Freight JCA files returned only `integrationSchemaNamespace` references (REST adapter namespaces), not ATP database schema references. AP Freight accesses ERP Fusion via REST and SOAP exclusively — it does not directly interact with ATP.

**Impact:** The count of "five separate ATP connection names" should be corrected to **four** (PO, Receipt, Supplier Outbound, AP Invoice Outbound). The `IFP_COMMON_ATP_DB_CONNEC` is the remote source in `IFP_COMMON_CONNECTION`, not a project-local connection. The AP Freight pipeline observability design (Module 2 wiring) needs to account for the fact that AP Freight has no existing ATP adapter — adding one requires a new connection override to be created in that project.

---

### Audit Finding 3 — FACTUAL ERROR: `privateEndpoint` on ATP Connections

**Blueprint claim (Section 3):**
> "`privateEndpoint=true` on `IFP_SHARED_ATP_DB_CONN`. ATP traffic routes through the OCI VCN private subnet. **Currently confirmed `false` from all ATP JCA files.**"

**Evidence from extracted files:**  
The ATP connection XML files (`IFP_PO_ATP_DB_CONNEC.xml`, `IFP_PO_RECEI_ATP_DB_CONNE.xml`, `IFP_APINV_ATP_DB_CONNEC.xml`) do **not contain a `<privateEndpoint>` element at all**. Only `IFP_SUPP_OUTBOU_INTEGR_ATP.xml` explicitly has `<privateEndpoint>false</privateEndpoint>`. The REST/EAM connections have `<privateEndpoint>false</privateEndpoint>` explicitly.

**Impact:** The claim "confirmed `false` from all ATP JCA files" is imprecise. Three of four ATP connection files omit the element entirely. In OIC, absence of the element defaults to `false`, so the functional conclusion is the same, but the evidence statement overstates what was confirmed.

---

### Audit Finding 4 — INTERNAL INCONSISTENCY: `PIPELINE_RUN_ID` Column Size

**Blueprint (Section 4):**
```sql
PIPELINE_RUN_ID   VARCHAR2(32)    NOT NULL,   -- RAWTOHEX(SYS_GUID()), 32-char hex UUID
```

**OIC Technical Assessment (Section 2, Design Component 2):**
```sql
PIPELINE_RUN_ID   VARCHAR2(36)  PRIMARY KEY,   -- RAWTOHEX(SYS_GUID())
```

`RAWTOHEX(SYS_GUID())` returns exactly 32 hexadecimal characters. The blueprint's `VARCHAR2(32)` is technically correct. The assessment document's `VARCHAR2(36)` appears to be sized for a standard RFC 4122 UUID format (32 hex + 4 hyphens = 36), but `RAWTOHEX()` does not insert hyphens. These two documents are inconsistent.

**Required action:** Standardise on `VARCHAR2(32)` if using `RAWTOHEX(SYS_GUID())` directly. If hyphenated UUIDs are preferred for readability (e.g., `a3f9b2c1-d7e1-4a4f-8c2b-3d5e00000000`), then insert hyphens in the package function and use `VARCHAR2(36)`. Pick one — do not leave both documents with different sizes.

---

### Audit Finding 5 — DESIGN GAP: `PIPELINE_RUN` Table Missing `PIPELINE_TYPE` CHECK Constraint

The `PIPELINE_TYPE` column accepts freeform `VARCHAR2(30)` values. Five allowed values are listed in comments (`'PO_INBOUND'`, `'RECEIPT_INBOUND'`, `'AP_FREIGHT'`, `'AP_INVOICE_OUTBOUND'`, `'SUPPLIER_OUTBOUND'`), but no `CHECK` constraint enforces them:

```sql
-- Missing from DDL:
CONSTRAINT CHK_PIPELINE_TYPE CHECK (
  PIPELINE_TYPE IN ('PO_INBOUND', 'RECEIPT_INBOUND', 'AP_FREIGHT',
                     'AP_INVOICE_OUTBOUND', 'SUPPLIER_OUTBOUND')
)
```

Similarly, `INITIATED_BY` and `STATUS` columns lack CHECK constraints despite having documented valid values. Adding constraints prevents silent data corruption from misspelled values (e.g., `'PO_INBOOUND'` or `'SCHEDULAR'`).

---

### Audit Finding 6 — DESIGN GAP: No Data Retention / Purge Strategy for Observability Tables

The blueprint introduces two new tables (`PIPELINE_RUN`, `PIPELINE_STAGE_LOG`) that will accumulate rows indefinitely. With 5 pipelines potentially running multiple times daily, each with 5–7 stage log entries, the tables will grow significantly.

**Missing from the blueprint:**
- Retention period definition (e.g., 90 days, 1 year)
- Purge mechanism (scheduled DBMS_SCHEDULER job or OIC-triggered procedure)
- Archival strategy (move to a `_ARCHIVE` table before purge)
- Index maintenance plan for the three indexes defined

This should be addressed in Phase 1 or Phase 2 to prevent the observability tables from becoming a performance liability.

---

### Audit Finding 7 — DESIGN GAP: Reprocess Dispatcher Missing Input Validation

The unified reprocess dispatcher (Section 9) accepts a `POST /reprocess` body with `entryStage` as a free-text string routed via CBR. There is no documented validation for:

1. **Invalid `entryStage` values** — What happens if someone posts `"entryStage": "EXTRACTION"` (not in the valid set)? The CBR should have an explicit `otherwise` branch that returns an HTTP 400 with a message listing valid values.
2. **Missing required fields per stage** — `CALLBACK` requires `loadRequestId`, `IMPORT` may require `importId`. No validation matrix is defined for which fields are required for which `entryStage`.
3. **`originalPipelineRunId` existence check** — If the provided `originalPipelineRunId` does not exist in `PIPELINE_RUN`, the FK constraint on `REPROCESS_OF` will fail. The dispatcher should validate existence before calling `START_PIPELINE_RUN` and return a meaningful error.

**Recommended addition:** A validation matrix in the blueprint:

| `entryStage` | Required fields | Optional fields |
|---|---|---|
| `VALIDATION` | `pbatchId` | `originalPipelineRunId` |
| `IMPORT` | `pbatchId`, `importId` | `originalPipelineRunId` |
| `CALLBACK` | `pbatchId`, `loadRequestId` | `originalPipelineRunId`, `importId` |
| `EAM_WRITEBACK` | `pbatchId` | `originalPipelineRunId` |

---

### Audit Finding 8 — DESIGN GAP: `COMPLETE_RUN` Placement Ambiguity

The propagation model (Section 6) shows `COMPLETE_RUN` / `FAIL_RUN` called only at the final stage (EAM Writeback, step 7). However:

1. **If an intermediate stage fails and sends email + `ehStop`**, the `PIPELINE_RUN.STATUS` remains `'RUNNING'` forever. The `FAIL_STAGE` call writes the stage failure, but `FAIL_RUN` must also be called to update the run status to `'FAILED'`.
2. The redesigned orchestration pattern (Section 7) shows `FAIL_RUN()` inside the catch handler of the Batchwise Processing integration — which is an intermediate stage, not the final stage.
3. **Contradiction:** If every intermediate catch handler calls `FAIL_RUN()`, the run status is set to `'FAILED'` even when downstream stages haven't been attempted. If only the final stage calls it, intermediate failures leave the run stuck in `'RUNNING'`.

**Resolution needed:** Define explicitly whether `FAIL_RUN` is called by every integration's catch handler (marking the run as failed at the first failure) or only by the final stage. The "stuck batch" detection query (Q3) assumes `STATUS = 'RUNNING'` means genuinely stuck — but if intermediate failures never call `FAIL_RUN`, it will also flag legitimately failed batches.

**Recommendation:** Every integration's catch handler should call `FAIL_RUN()`. A failed intermediate stage means the pipeline run has failed (it won't proceed to downstream stages after `ehStop`). This is consistent with the cascade guard pattern already shown in Section 7.

---

### Audit Finding 9 — DESIGN RISK: ESS Callback Pattern Requires Additional Infrastructure

The ESS job polling replacement (Section 11) recommends subscribing to ERP Business Events for ESS job completion. This pattern requires:

1. **ERP Cloud Business Event configuration** — The ERP instance must have the relevant business event (`oracle/apps/ess/core/async/JobStatus` or equivalent) enabled and published. This is an ERP Cloud admin task, not an OIC-side change.
2. **Event correlation** — The ERP Business Event payload must contain the `loadRequestId` or ESS job ID to correlate back to the OIC pipeline run. The blueprint assumes this but does not verify it from the ERP event schema.
3. **Missed event recovery** — If OIC is down or the event subscription fails when the ESS job completes, the completion event is lost. The blueprint does not address this failure mode. A safety-net polling integration (running on a longer interval, e.g., every 30 minutes) may be needed to catch missed callbacks.

**Impact:** This is listed as a Phase 4 change, which is appropriate, but the prerequisite ERP Cloud configuration should be explicitly documented as a dependency.

---

### Audit Finding 10 — DESIGN CONSIDERATION: Event Type Separation May Cause Operational Complexity

Section 8 recommends splitting each event type into separate first-run and reprocess variants:
- `IFP_PO_BATCH_SCHEDULED_V2` + `IFP_PO_BATCH_REPROCESS_V2`
- `IFP_RCPT_BATCH_SCHEDULED_V2` + `IFP_RCPT_BATCH_REPROCESS_V2`

This doubles the number of event type definitions and requires separate subscriber integrations (or a single subscriber with dual subscriptions). An alternative approach — adding a `processType` field to the existing event payload and routing inside the subscriber via CBR — achieves the same disambiguation with fewer moving parts and no coordinated redeployment.

The blueprint already mentions the `processType` field approach (in the Technical Assessment, Section 2.5) but then overrides it with the full event type separation in Section 8. These two approaches should be reconciled. The `processType` field approach is lower risk and should be the primary recommendation, with full event type separation as a future optimization if needed.

---

### Audit Finding 11 — COMPLETENESS: Missing AP Freight and Supplier Outbound Pipeline Details

Sections 6 and 7 provide detailed propagation models and error handling redesigns for PO Inbound (7-hop) and implicitly Receipt Inbound (structurally identical). The following pipelines have no equivalent detail:

| Pipeline | Hop count documented? | Error handling pattern documented? | Tracking variables documented? |
|---|---|---|---|
| PO Inbound | Yes (7 hops) | Yes | Yes |
| Receipt Inbound | Implicit (mirror of PO) | Yes (Batchwise only) | Yes |
| AP Freight | No | No | No |
| AP Invoice Outbound | No (referenced as positive pattern only) | Partial (named fault reference) | No |
| Supplier Outbound | No | No | No |

The blueprint should include at minimum the pipeline hop sequence and `pipelineRunId` generation point for all 5 pipelines, not just PO and Receipt.

---

### Audit Finding 12 — PHASE DEPENDENCY: Phase 0 and Phase 1 Have an Implicit Ordering Conflict

Phase 0 says: "Fix the inner `ehStop` pattern — move all business logic to the outer try scope."

Phase 1, step 3 says: "Wire `START_PIPELINE_RUN` into the entry integration of each pipeline."

Phase 2 says: "Add `START_STAGE` / `COMPLETE_STAGE` calls at every integration boundary."

The redesigned orchestration pattern in Section 7 shows `START_STAGE`, `COMPLETE_STAGE`, and `FAIL_STAGE` calls embedded in the restructured try/catch — but these ATP procedures don't exist until Phase 1. If Phase 0 is deployed before Phase 1, the restructured orchestration cannot include ATP logging calls.

**Resolution:** Phase 0 should restructure the try/catch to eliminate the inner `ehStop` and ensure email fires, but should NOT yet include ATP logging calls. The ATP logging is added in Phase 2 after the schema and package exist from Phase 1. The Section 7 redesigned pattern combines both Phase 0 and Phase 2 changes into one diagram without distinguishing them, which will cause confusion during implementation.

---

### Audit Summary

| # | Finding | Severity | Section Affected |
|---|---|---|---|
| 1 | "No native connection inheritance" is incorrect — `IFP_COMMON_CONNECTION` project already exists | **Factual Error** | 3 (Module 1) |
| 2 | AP Freight has no ATP connection — count is 4, not 5 | **Factual Error** | 3 (Module 1) |
| 3 | `privateEndpoint=false` not explicitly present in all ATP JCA files | Minor Imprecision | 3 (Module 1), 10 |
| 4 | `PIPELINE_RUN_ID` size inconsistent between blueprint (32) and assessment (36) | Internal Inconsistency | 4 (Module 2) |
| 5 | No CHECK constraints on enumerated columns | Design Gap | 4 (Module 2) |
| 6 | No data retention / purge strategy for observability tables | Design Gap | 4 (Module 2) |
| 7 | Reprocess dispatcher has no input validation design | Design Gap | 9 |
| 8 | `COMPLETE_RUN` / `FAIL_RUN` placement ambiguity across pipeline stages | Design Ambiguity | 4, 6, 7 |
| 9 | ESS callback pattern prerequisites not documented | Design Risk | 11 |
| 10 | Event type separation vs. `processType` field — approaches not reconciled | Design Inconsistency | 8 |
| 11 | AP Freight, AP Invoice Outbound, Supplier Outbound pipeline details missing | Incomplete | 6, 7 |
| 12 | Phase 0 / Phase 1 / Phase 2 dependency conflict in orchestration redesign | Phase Ordering Issue | 7, 13 |

---

*Audit performed against extracted `.car` file contents (connection XML, JCA, `info.json`, `project.xml`, WSDL). All findings are evidence-based.*
