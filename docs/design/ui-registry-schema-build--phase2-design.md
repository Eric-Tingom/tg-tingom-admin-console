# UI Registry Schema Build -- Phase 2 Design Document

**Build Item:** `ui-registry-schema-build`
**Phase:** Design (Phase 1 of 8-phase SDLC)
**Status:** Blocked -- awaiting SCR approval (DBA Agent + Integration Architect)
**SCR:** `f986ae43-5a70-4bc5-b636-895d54e1d01b`
**Date:** 2026-02-19
**Authored by:** Lane A Development Agent Team (PM Agent + BA Agent)

---

## 1. Locked Architectural Decisions

| Decision | Choice |
|---|---|
| UI-managed scope | Registry-only (tables explicitly listed in `ui_table_registry` only) |
| Audit enforcement | Hybrid: hard trigger on UI paths, detection DQ job on system paths |
| Append-only enforcement | Grants revoked + BEFORE trigger (belt and suspenders) |
| Approval scope | Config-only (registry edits) + field-level patch payload |
| Soft-delete policy | Filtered by default + restore allowed + logging required |
| Roles model | Per-tenant (org_id) + field-level permissions |
| Optimistic locking | `version INT` on `ui_table_registry` and `ui_field_registry` |

---

## 2. Table Specifications

### 2.1 `ui_table_registry`
Purpose: Register UI-governed tables.

```sql
CREATE TABLE ui_table_registry (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id         UUID NOT NULL,
  table_name     TEXT NOT NULL,
  is_ui_managed  BOOLEAN NOT NULL DEFAULT TRUE,
  action_mode    TEXT NOT NULL CHECK (action_mode IN ('direct', 'approval_required')),
  deleted_at     TIMESTAMPTZ NULL,
  deleted_by     UUID NULL,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  created_by     UUID,
  updated_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_by     UUID,
  version        INT NOT NULL DEFAULT 1,
  CONSTRAINT ui_table_registry_org_table_uq UNIQUE (org_id, table_name)
);
CREATE INDEX idx_ui_table_registry_org ON ui_table_registry (org_id);
CREATE INDEX idx_ui_table_registry_table ON ui_table_registry (table_name);
ALTER TABLE ui_table_registry ENABLE ROW LEVEL SECURITY;
```

### 2.2 `ui_field_registry`
Purpose: Field-level configuration and validation rules.

```sql
CREATE TABLE ui_field_registry (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id            UUID NOT NULL,
  table_name        TEXT NOT NULL,
  field_name        TEXT NOT NULL,
  read_only         BOOLEAN NOT NULL DEFAULT FALSE,
  is_computed       BOOLEAN NOT NULL DEFAULT FALSE,
  validation_json   JSONB,
  requires_approval BOOLEAN NOT NULL DEFAULT FALSE,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  created_by        UUID,
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_by        UUID,
  version           INT NOT NULL DEFAULT 1,
  CONSTRAINT ui_field_registry_uq UNIQUE (org_id, table_name, field_name)
);
CREATE INDEX idx_ui_field_registry_org_table ON ui_field_registry (org_id, table_name);
ALTER TABLE ui_field_registry ENABLE ROW LEVEL SECURITY;
```

### 2.3 `ui_views`
Purpose: Saved filter presets per table.

```sql
CREATE TABLE ui_views (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id      UUID NOT NULL,
  table_name  TEXT NOT NULL,
  view_name   TEXT NOT NULL,
  filter_json JSONB,
  sort_json   JSONB,
  is_default  BOOLEAN NOT NULL DEFAULT FALSE,
  created_by  UUID,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  CONSTRAINT ui_views_uq UNIQUE (org_id, table_name, view_name)
);
ALTER TABLE ui_views ENABLE ROW LEVEL SECURITY;
```

### 2.4 `ui_approval_queue`
Purpose: Config-only approval workflow with field-level patch payload.

```sql
CREATE TABLE ui_approval_queue (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id         UUID NOT NULL,
  table_name     TEXT NOT NULL,
  field_name     TEXT,
  change_payload JSONB NOT NULL,
  status         TEXT NOT NULL DEFAULT 'pending'
                   CHECK (status IN ('pending', 'approved', 'rejected')),
  requested_by   UUID,
  requested_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  reviewed_by    UUID,
  reviewed_at    TIMESTAMPTZ
);
CREATE INDEX idx_ui_approval_queue_org_status ON ui_approval_queue (org_id, status);
CREATE INDEX idx_ui_approval_queue_table_field ON ui_approval_queue (table_name, field_name);
ALTER TABLE ui_approval_queue ENABLE ROW LEVEL SECURITY;
```

### 2.5 `ui_change_log`
Purpose: Append-only audit trail.

```sql
CREATE TABLE ui_change_log (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id         UUID NOT NULL,
  table_name     TEXT NOT NULL,
  row_id         UUID NOT NULL,
  operation      TEXT NOT NULL CHECK (operation IN ('insert', 'update', 'delete', 'restore')),
  changed_fields JSONB,
  source         TEXT NOT NULL CHECK (source IN ('ui', 'system')),
  created_by     UUID,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ui_change_log_main ON ui_change_log (org_id, table_name, row_id, created_at);
CREATE INDEX idx_ui_change_log_created ON ui_change_log (created_at);
ALTER TABLE ui_change_log ENABLE ROW LEVEL SECURITY;
```

### 2.6 `ui_roles`
Purpose: Tenant-scoped role definitions.

```sql
CREATE TABLE ui_roles (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id     UUID NOT NULL,
  role_name  TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  CONSTRAINT ui_roles_uq UNIQUE (org_id, role_name)
);
ALTER TABLE ui_roles ENABLE ROW LEVEL SECURITY;
```

### 2.7 `ui_role_permissions`
Purpose: Table and field-level permissions per tenant role.

```sql
CREATE TABLE ui_role_permissions (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id     UUID NOT NULL,
  role_id    UUID NOT NULL REFERENCES ui_roles(id) ON DELETE CASCADE,
  table_name TEXT NOT NULL,
  field_name TEXT,
  can_create BOOLEAN NOT NULL DEFAULT FALSE,
  can_read   BOOLEAN NOT NULL DEFAULT TRUE,
  can_update BOOLEAN NOT NULL DEFAULT FALSE,
  can_delete BOOLEAN NOT NULL DEFAULT FALSE,
  CONSTRAINT ui_role_permissions_uq UNIQUE (org_id, role_id, table_name, field_name)
);
ALTER TABLE ui_role_permissions ENABLE ROW LEVEL SECURITY;
```

---

## 3. Soft-Delete Policy (Existing Tables)

Tables: `work_items`, `output_templates`, `connected_systems`

```sql
ALTER TABLE work_items       ADD COLUMN IF NOT EXISTS deleted_at TIMESTAMPTZ NULL;
ALTER TABLE work_items       ADD COLUMN IF NOT EXISTS deleted_by UUID NULL;
ALTER TABLE output_templates ADD COLUMN IF NOT EXISTS deleted_at TIMESTAMPTZ NULL;
ALTER TABLE output_templates ADD COLUMN IF NOT EXISTS deleted_by UUID NULL;
ALTER TABLE connected_systems ADD COLUMN IF NOT EXISTS deleted_at TIMESTAMPTZ NULL;
ALTER TABLE connected_systems ADD COLUMN IF NOT EXISTS deleted_by UUID NULL;
```

Policy: Default SELECT filters `deleted_at IS NULL`. Admin only can soft-delete. Restore clears `deleted_at` + writes `ui_change_log` restore entry. Hard deletes not permitted.

---

## 4. RLS Matrix

| Table | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| `ui_table_registry` | tenant-scoped | role-gated | role-gated + version check | admin only |
| `ui_field_registry` | tenant-scoped | approval required | approval required + version check | admin only |
| `ui_views` | tenant-scoped | role-gated | owner or admin | owner or admin |
| `ui_approval_queue` | tenant-scoped | authenticated | reviewer only | DENY |
| `ui_change_log` | tenant-scoped | system/ui only | DENY | DENY |
| `ui_roles` | tenant-scoped | admin | admin | admin |
| `ui_role_permissions` | tenant-scoped | admin | admin | admin |

Tenant isolation: `USING (org_id = current_setting('app.org_id')::uuid)`

`ui_change_log` append-only: REVOKE UPDATE/DELETE from PUBLIC + BEFORE trigger raises exception.

---

## 5. Hybrid Enforcement Pattern

UI path: `app.write_source = 'ui'` -- BEFORE trigger verifies change log entry in same transaction.
System path: `app.write_source = 'system'` -- allowlisted, nightly DQ job detects missing log entries.

---

## 6. Optimistic Locking

`version INT` on `ui_table_registry` and `ui_field_registry`. RPCs accept `expected_version`, reject on mismatch with OPTIMISTIC_LOCK_CONFLICT error, increment on success.

---

## 7. Migration Order

1. Enums/check constraints
2. `ui_table_registry`, `ui_field_registry` (with version)
3. `ui_roles`, `ui_role_permissions`
4. `ui_approval_queue`
5. `ui_change_log`
6. `ui_views`
7. ALTER TABLE soft-delete columns
8. RLS policies
9. REVOKE UPDATE/DELETE on `ui_change_log`
10. `fn_ui_change_log_immutable` + trigger
11. Hybrid enforcement triggers
12. DBA advisory check
13. Insert DQ rule

---

## 8. Rollback Strategy

New tables: DROP TABLE CASCADE (no data). Soft-delete columns: DROP COLUMN (nullable, no data loss). Triggers/RLS: DROP TRIGGER, DROP POLICY. Feature disable: set `is_ui_managed=FALSE` globally.

---

## 9. Phase Definition of Done

Phase 1 (Design): [x] Decisions locked [x] SCR filed [x] Status=blocked enforced [x] phase_review_log written [x] Design doc in GitHub
Phase 2 (PeerReview): DBA Agent + Integration Architect sign-off on SCR f986ae43
Phase 3 (Build): Full migration SQL, triggers, RLS, DQ rule
Phase 4 (Test): Advisory check clean, all 10 ACs verified, RLS matrix row-by-row
Phase 5 (ReviewResults): QA report + AC verification notes
Phase 6 (CleanUp): dev_changelog entry, no open SCRs
Phase 8 (CloseOut): ui-registry-rpc-suite unblocked, build_queue.status=done

---

## 10. Open Gates

1. Eric clears admin-console-governance-agent blocker
2. SCR f986ae43 approved by DBA Agent
3. SCR f986ae43 approved by Integration Architect
4. Hybrid enforcement trigger spec finalized (DBA Agent)
5. System write allowlist confirmed (Eric)
6. DQ rule query finalized (BA Agent + DBA Agent)
