# Design Document: Admin Console Governance Agent

**Build Item:** `admin-console-governance-agent`
**Phase:** Design
**Priority:** 1 (Critical Path) | **Complexity:** Medium
**HubSpot Company:** 3971192462 (Tingom Group LLC)
**HubSpot Deal:** 56558749706
**Template Reference:** `b4a1f880-9487-4cce-8d1f-a453800ea8ac` (Design Document Template)
**Unblocks:** `ui-registry-schema-build` (P1), `admin-console-wave1-onboarding` (P2), `admin-console-builder-specialization` (P2)
**Gate Lane Mode:** Lane A + Lane B (all 11 reviewer roles)

---

## 1. Agent Purpose & Scope

The Admin Console Governance Agent is a governance-layer agent that owns the UI registry system within the COO Brain. It is the single authority for deciding which database tables are CRUD-enabled in the admin console, which fields within those tables are editable versus read-only, and which field edits require approval workflows before persisting.

This agent operates purely at the specification and governance level. It produces structured artifacts (onboarding specs, registry change proposals, approval gate definitions) that downstream agents consume. It never generates UI code, never modifies database schema directly, and never writes to operational (non-ui_*) tables.

### 1.1 Registration Details

| Field | Value |
|-------|-------|
| agent_name | `admin_console_governance` |
| platform | `skill` |
| scope | `flowops360` |
| registered_by | `design-phase-thread` |

---

## 2. Domain Boundaries

### 2.1 Owns (Exclusive Write Access)

| Table | Purpose |
|-------|--------|
| `ui_table_registry` | Master registry of tables exposed in admin console |
| `ui_field_registry` | Per-field governance: editability, visibility, validation rules |
| `ui_views` | View configurations: default, filtered, role-scoped views per table |
| `ui_approval_queue` | Approval workflow definitions for sensitive field edits |

### 2.2 Does NOT

- Build or generate UI code (React, Tailwind, shadcn/ui) - Frontend Builder Agent domain
- Execute DDL or schema changes directly - all proposals route through DBA Agent via SCR
- Write to any non-ui_* operational table
- Define or modify data quality rules - DQ Steward domain
- Monitor agent fleet health or detect conflicts - Meta-Ops domain
- Manage deployment approval gates - Release & Deploy Agent domain
- Directly call HubSpot MCP or any external API

### 2.3 Boundary Agent Map

| Agent | Type | Their Domain | Boundary Rule |
|-------|------|-------------|---------------|
| DBA Agent | governance | DDL execution | Governance proposes via SCR; DBA executes |
| Meta-Ops | governance | Fleet monitoring | Governance is monitored subject, not competitor |
| Frontend Builder | skill | UI code generation | Governance defines specs; Frontend implements |
| DQ Steward | governance | Data quality rules | Governance owns UI-input validation; DQ owns data integrity |
| Release & Deploy | governance | Deployment gates | Governance owns UI edit approval gates (different domain) |

---

## 3. IO Contracts

| Contract | Direction | Target | Description |
|----------|-----------|--------|-------------|
| `ui_registry_change` | OUTPUT | - | Structured change proposal for ui_* registry tables |
| `ui_onboarding_spec` | OUTPUT | - | Per-table artifact: intent, action_mode, form_mode, fields, validation, views, approval rules |
| `schema_change_request` | OUTPUT | DBA Agent | When UI registry needs new columns/tables via SCR |
| `conflict_check_request` | OUTPUT | Meta-Ops | Conflict detection before defining approval gates |
| `build_item_assignment` | INPUT | PM Agent | Work items assigned via build_queue |
| `ui_code_generation_request` | OUTPUT | Frontend Builder | After onboarding spec approved, hand off for code gen |

---

## 4. Hard Rules

### 4.1 Table Write Allowlist
Only: `ui_table_registry`, `ui_field_registry`, `ui_views`, `ui_approval_queue`.

### 4.2 Schema Change Protocol
All schema proposals route through SCR workflow. Direct DDL prohibited.

### 4.3 Approval Gate Rules
Must request Meta-Ops conflict check before defining any new approval gate.

### 4.4 current_rules jsonb

```json
{
  "table_write_allowlist": ["ui_table_registry", "ui_field_registry", "ui_views", "ui_approval_queue"],
  "schema_change_protocol": "scr_workflow_required",
  "approval_gate_rules": {
    "requires_meta_ops_conflict_check": true,
    "conflict_check_contract": "conflict_check_request"
  },
  "io_contracts": {
    "outputs": ["ui_registry_change", "ui_onboarding_spec", "schema_change_request", "conflict_check_request", "ui_code_generation_request"],
    "inputs": ["build_item_assignment"]
  }
}
```

---

## 5. Capability Inventory

| Capability Tag | Name | Description |
|---------------|------|-------------|
| `ui_table_onboarding` | UI Table Onboarding | Evaluate table, produce onboarding spec |
| `ui_field_governance` | UI Field Governance | Define field editability, visibility, computed status |
| `ui_approval_gate_definition` | UI Approval Gate Definition | Define approval workflows for sensitive edits |
| `ui_view_configuration` | UI View Configuration | Define default, filtered, role-scoped views |
| `ui_registry_change_review` | UI Registry Change Review | Review proposed changes to ui_* tables |
| `ui_action_mode_classification` | UI Action Mode Classification | Classify tables as CRUD, action-only, or view-only |

---

## 6. Trigger Conditions

| Trigger | Description |
|---------|-------------|
| `build_item_assignment_pm` | PM assigns build_queue item with component = admin_console |
| `manual_user_request` | Eric requests table onboarding or governance review |
| `ui_registry_change_proposal` | Another agent proposes ui_* registry change |

---

## 7. Dependency Map

| Dependency | Purpose |
|-----------|--------|
| `ui_table_registry` | Read existing table registrations |
| `ui_field_registry` | Read existing field governance |
| `ui_views` | Read existing view configurations |
| `ui_approval_queue` | Read existing approval gates |
| `agent_capability_tags` | Read for conflict detection verification |
| `schema_change_requests` | Write SCR artifacts |
| `build_queue` | Read assigned work items |

---

## 8. Meta-Ops Conflict Analysis

| Existing Agent | Capability | Overlap | Verdict |
|---------------|-----------|---------|--------|
| DBA Agent | cascade_review_initiation | None | CLEAR |
| DBA Agent | cascade_staleness_check | None | CLEAR |
| Meta-Ops | conflict_detection | None | CLEAR |
| Meta-Ops | domain_coverage_analysis | None | CLEAR |
| Frontend Builder | frontend_change_generation | None | CLEAR |
| Release & Deploy | approval_gate_management | Low (different domain) | CLEAR |

**Conclusion:** No domain overlaps. All 6 capabilities are unique.

---

## 9. SQL Statements (Ready to Execute)

### 9.1 agent_configurations INSERT

```sql
INSERT INTO agent_configurations (
  agent_name, platform, scope, current_rules,
  capabilities, dependencies, work_accepted_from, registered_by
) VALUES (
  'admin_console_governance', 'skill', 'flowops360',
  '{"table_write_allowlist":["ui_table_registry","ui_field_registry","ui_views","ui_approval_queue"],"schema_change_protocol":"scr_workflow_required","approval_gate_rules":{"requires_meta_ops_conflict_check":true,"conflict_check_contract":"conflict_check_request"},"io_contracts":{"outputs":["ui_registry_change","ui_onboarding_spec","schema_change_request","conflict_check_request","ui_code_generation_request"],"inputs":["build_item_assignment"]}}'::jsonb,
  ARRAY['ui_table_onboarding','ui_field_governance','ui_approval_gate_definition','ui_view_configuration','ui_registry_change_review','ui_action_mode_classification'],
  ARRAY['ui_table_registry','ui_field_registry','ui_views','ui_approval_queue','agent_capability_tags','schema_change_requests','build_queue'],
  ARRAY['build_item_assignment_pm','manual_user_request','ui_registry_change_proposal'],
  'design-phase-thread'
);
```

### 9.2 agent_capability_tags INSERTs

```sql
INSERT INTO agent_capability_tags (assistant_name, agent_type, is_active, capability_tag, capability_name, category, capability_description) VALUES
  ('Admin Console Governance Agent','governance',true,'ui_table_onboarding','UI Table Onboarding','agent_capability','Evaluate table, produce onboarding spec'),
  ('Admin Console Governance Agent','governance',true,'ui_field_governance','UI Field Governance','agent_capability','Define field editability, visibility, computed status'),
  ('Admin Console Governance Agent','governance',true,'ui_approval_gate_definition','UI Approval Gate Definition','agent_capability','Define approval workflows for sensitive edits'),
  ('Admin Console Governance Agent','governance',true,'ui_view_configuration','UI View Configuration','agent_capability','Define default, filtered, role-scoped views'),
  ('Admin Console Governance Agent','governance',true,'ui_registry_change_review','UI Registry Change Review','agent_capability','Review proposed changes to ui_* tables'),
  ('Admin Console Governance Agent','governance',true,'ui_action_mode_classification','UI Action Mode Classification','agent_capability','Classify tables as CRUD, action-only, or view-only');
```

---

## 10. Acceptance Criteria Verification

| # | Criterion | Status |
|---|-----------|--------|
| 1 | Agent registered in agent_configurations | SQL Ready |
| 2 | IO contracts declared | Documented (Sec 3) |
| 3 | Domain boundaries with does-NOT list | Documented (Sec 2) |
| 4 | Hard rules in current_rules jsonb | Documented (Sec 4) |
| 5 | Meta-Ops conflict detection, no overlaps | Verified (Sec 8) |
| 6 | Trigger conditions defined | Documented (Sec 6) |