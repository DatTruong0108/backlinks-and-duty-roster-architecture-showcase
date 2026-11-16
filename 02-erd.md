# 2. ERD — Data Model (23 tables)

One Postgres database, one `public` schema, split by **naming convention**: backlink tables have plain names,
on-call tables are prefixed `duty_`. They share `employee` and `app_access`.

| Group | Tables | Count |
|---|---|---|
| Backlink tracking | `site`, `employee`, `domain`, `keyword`, `output`, `daily_report`, `index_history`, `gsc_link`, `ahrefs_gap`, `setting`, `sync_log`, `snapshot` | 12 |
| Shared access | `app_access` | 1 |
| On-call roster | `duty_setting`, `duty_roster`, `duty_shift`, `duty_swap_log`, `duty_action_log`, `duty_change`, `duty_check_template`, `duty_check_template_history`, `duty_check_result`, `duty_error` | 10 |

All 23 tables have RLS enabled. Anonymous users can read at most `snapshot` (minus the settings page) and `sync_log`.

## 2.1 Backlink module

```mermaid
erDiagram
  site ||--o{ keyword : "has"
  site ||--o{ output : "has"
  site ||--o{ daily_report : "has"
  site ||--o{ gsc_link : "has"
  site ||--o{ ahrefs_gap : "has"
  site ||--o{ sync_log : "per-site runs"
  site ||--o{ snapshot : "per-site pages"
  site ||--o{ setting : "per-site config"
  site |o--o{ site : "parent_site_id"
  employee ||--o{ output : "produced by"
  employee ||--o{ daily_report : "reported by"
  domain |o--o{ output : "published on"
  domain |o--o{ gsc_link : "matched to"
  output ||--o{ index_history : "status changes"
  sync_log ||--o{ gsc_link : "loaded by"
  sync_log ||--o{ ahrefs_gap : "loaded by"
  sync_log ||--o{ snapshot : "computed in"

  site {
    bigint id PK
    varchar domain UK
    varchar code UK
    boolean is_own
    bigint parent_site_id FK
    varchar sheet_id
    varchar gsc_sheet_id
    varchar ahrefs_sheet_id
    date start_date
    boolean is_active
  }
  employee {
    bigint id PK
    varchar name UK
    char short_code UK
    jsonb start_dates
    boolean is_active
    text display_name
  }
  domain {
    bigint id PK
    varchar domain UK
    boolean is_valid
    boolean in_storage
    int dr
    jsonb accounts
    timestamptz updated_at
  }
  keyword {
    bigint id PK
    bigint site_id FK
    varchar phase
    varchar station_code
    text keyword_norm
    bigint volume
    jsonb targets
    jsonb progress
  }
  output {
    bigint id PK
    bigint site_id FK
    bigint employee_id FK
    bigint domain_id FK
    text url_norm
    date done_date
    index_status index_status
    text_arr anchors
    jsonb errors
    int sheet_row
    boolean is_deleted
  }
  daily_report {
    bigint id PK
    bigint site_id FK
    bigint employee_id FK
    date report_date
    int total_output
    int error_total
  }
  index_history {
    bigint id PK
    bigint output_id FK
    index_status index_status
    timestamptz changed_at
  }
  gsc_link {
    bigint id PK
    bigint site_id FK
    bigint sync_log_id FK
    bigint domain_id FK
    text linking_page_norm
    varchar domain_match
    varchar index_status
  }
  ahrefs_gap {
    bigint id PK
    bigint site_id FK
    bigint sync_log_id FK
    varchar station_code
    date month
    jsonb by_site
    boolean is_spike
    boolean is_excluded
  }
  setting {
    bigint id PK
    bigint site_id FK
    varchar key
    jsonb value
  }
  sync_log {
    bigint id PK
    bigint site_id FK
    sync_source source
    varchar trigger
    varchar status
    jsonb stats
    jsonb issues
    timestamptz started_at
    timestamptz finished_at
  }
  snapshot {
    bigint id PK
    bigint site_id FK
    bigint sync_log_id FK
    varchar page
    date as_of_date
    jsonb payload
  }
```

Enums: `index_status` = `indexed | not_indexed | http_404 | submitted | blank | other`;
`sync_source` = `site_sheet | domain_storage | gsc_sheet | ahrefs_sheet`.

### Backlink schema invariants

| Invariant | Mechanism | Why |
|---|---|---|
| One output row per `(site, employee, normalized URL)` | `UNIQUE (site_id, employee_id, url_norm)` | Duplicate URLs in a tab collapse to one row and raise a `duplicate_url` issue instead of double-counting. |
| `sheet_row` is informational only | Not part of any key | Rows move in a sheet; keys must be content-derived. |
| History only when status changes | `index_history` insert-on-change | Bounded growth; the table cannot be rebuilt from any other source. |
| One config row per `(site, key)` including shared keys | `UNIQUE NULLS NOT DISTINCT (site_id, key)` | Allows `ON CONFLICT` upsert for rows where `site_id IS NULL`. |
| Only one running sync per source | Partial unique index on `sync_log (source, site_id) NULLS NOT DISTINCT WHERE status = 'running'` | A DB-level mutex (snippet 4.7). |
| Snapshot is the only thing the UI reads | Grants: `SELECT` on `snapshot`, `sync_log` only | UI cannot depend on raw tables. |

## 2.2 On-call module

```mermaid
erDiagram
  employee ||--o{ duty_roster : "rotation member"
  employee ||--o{ duty_shift : "staff_id (on duty)"
  employee ||--o{ duty_shift : "base_staff_id (by rule)"
  employee |o--o{ duty_swap_log : "actor_staff_id"
  employee |o--o{ duty_check_result : "checked_by_staff_id"
  employee |o--o{ duty_error : "reported_by_staff_id"
  duty_check_template ||--o{ duty_check_template_history : "version snapshots"
  duty_check_template ||--o{ duty_check_result : "item_id"
  duty_check_template ||--o{ duty_error : "item_id"

  duty_setting {
    text key PK
    jsonb value
    timestamptz updated_at
    text updated_by
  }
  duty_roster {
    bigint id PK
    bigint employee_id FK
    numeric position
    date active_from
    date active_to
  }
  duty_shift {
    date date PK
    bigint staff_id FK
    bigint base_staff_id FK
    uuid last_request_id
    timestamptz updated_at
    text updated_by
  }
  duty_swap_log {
    bigint id PK
    text kind
    text actor_email
    bigint actor_staff_id FK
    text actor_name
    jsonb changes
    text reason
    uuid request_id UK
    timestamptz created_at
  }
  duty_action_log {
    bigint id PK
    timestamptz created_at
    uuid actor_user_id
    text actor_email
    text_arr actor_roles
    text action
    text rpc_name
    jsonb params
    text outcome
    int http_status
    text error_code
    jsonb result
    uuid request_id UK
    timestamptz decided_at
  }
  duty_change {
    bigint id PK
    text scope
    text ref
    timestamptz at
  }
  duty_check_template {
    text id PK
    text region
    text slot_label
    int start_min
    int end_min
    jsonb content
    jsonb station_order
    int sort_order
    boolean is_active
    int version
  }
  duty_check_template_history {
    bigint id PK
    text template_id FK
    int version
    text changed_by
    uuid request_id
  }
  duty_check_result {
    bigint id PK
    date date
    text item_id FK
    text status
    text checked_by_name
    timestamptz checked_at
    int template_version
    uuid request_id UK
  }
  duty_error {
    bigint id PK
    date date
    text region
    text item_id FK
    text status
    text reported_by_name
    timestamptz reported_at
    timestamptz resolved_at
    uuid request_id UK
  }
```

Intentionally **no foreign key** from `duty_action_log.actor_user_id` to any user table: deleting an account must
never delete the audit trail.

### On-call schema invariants

| Invariant | Mechanism | Why |
|---|---|---|
| One shift per working day, Mon–Fri only | `date` is the PK; `CHECK (isodow BETWEEN 1 AND 5)` | The date *is* the identity of a shift; weekends cannot exist. |
| `staff_id` vs `base_staff_id` | Two columns | `staff_id` = who is on duty now; `base_staff_id` = who the rotation rule assigns. They differ ⇔ the shift was swapped, which makes "reset to schedule" trivial. |
| Rotation members: same person ⇒ same `position`; two people never share a position; no overlapping periods for one person | Trigger `duty_roster_guard` under `pg_advisory_xact_lock` | Serializes concurrent roster edits; `position` is `numeric` so a newcomer can be inserted between 2 and 3 as 2.5. |
| `active_to` is exclusive | Half-open interval `[active_from, active_to)` | Removes "is the last day included?" ambiguity. |
| **Idempotency** | `request_id uuid UNIQUE` on every write-record table | A retried request can never create a second row. |
| **Exactly one audit row per action** | `UNIQUE (request_id)` on `duty_action_log` | One-to-one between actions and log rows; gaps are detectable by reconciliation. |
| Names are snapshotted | `actor_name`, `checked_by_name`, `item_name`, `slot_label`, `template_version` copied at write time | History stays correct after staff leave or templates are edited. |
| Error state machine | `CHECK ((status='open' AND resolved_* IS NULL) OR (status<>'open' AND resolved_at/by NOT NULL))` | An error cannot be "resolved" without a resolver and timestamp. |
| Check content shape | `CHECK` with `jsonb_path_query_array(... type())` | Arrays of strings only, validated in the DB, not just in the API. |
| Realtime exposes one table | Publication contains only `duty_change` | See ADR-4. Rows older than one day are purged. |
| Action log is dev-only | RLS: only role `dev` can `SELECT`; not in the publication | The log may contain operational detail; admins are deliberately excluded. |

## 2.3 Access control tables

```mermaid
erDiagram
  employee |o--o| app_access : "employee_id (unique)"
  app_access {
    text email PK "lower-case, CHECK email = lower(email)"
    text role "admin | editor | dev | NULL"
    text_arr duty_roles "staff | po"
    bigint employee_id FK
    timestamptz created_at
    text created_by
  }
```

| Constraint | Meaning |
|---|---|
| `role IS NOT NULL OR cardinality(duty_roles) > 0` | A row with no role at all is meaningless. |
| `'staff' ∈ duty_roles ⇒ employee_id IS NOT NULL` | A staff member must be linked to the person whose shifts they own. |
| `UNIQUE (employee_id)` | One account per person. |
| Only a `dev` may insert/update/delete a row whose `role = 'dev'` | Prevents an admin from self-escalating to the only role that can read the audit log. |

### Role matrix

| Capability | staff | po | admin | dev | editor |
|---|---|---|---|---|---|
| View on-call data | ✔ | ✔ | ✔ | ✔ | ✘ |
| Mark checks OK / Error | ✔ | ✔ | ✔ | ✔ | ✘ |
| Swap own shifts | ✔ | ✔ | ✔ | ✔ | ✘ |
| Swap anyone's shifts, single-sided swap, reset | ✘ | ✔ | ✔ | ✔ | ✘ |
| Edit config (backlink "level-1" keys) | ✘ | ✘ | ✔ | ✔ | ✔ |
| Read the action log page | ✘ | ✘ | ✘ | ✔ | ✘ |

Identity rule shared by every helper: the caller's **e-mail claim** must match `app_access` **and** the token's
`app_metadata.providers` must contain the OAuth provider. `app_metadata` can only be changed by the service role;
`user_metadata` is user-editable and is **never** used.
