# 3. Sequence Diagrams

Four flows, hardest first:

| ID | Flow | Why it is hard |
|---|---|---|
| S1 | Shift swap | Idempotency + row locks + **in-transaction server clock** + exactly-one audit row + realtime notify, under concurrent users |
| S2 | Scheduled data sync | Unreliable scheduler → redundant triggers that must never double-run |
| S3 | Realtime refresh | Live UI without ever trusting the channel for data |
| S4 | Fail-closed log archiving | Delete only what is *proven* saved elsewhere |

---

## S1 — Shift swap (two-way, effective immediately)

Actors: a signed-in staff member swaps *their* shift on one day with another person's shift on another day. There is
no invitation/accept step; the swap is atomic and immediate, and is recorded in both a user-visible history table and
a dev-only audit log.

```mermaid
sequenceDiagram
  autonumber
  actor U as User (browser)
  participant UI as UI + server-clock estimator
  participant PX as Proxy (middleware)
  participant RT as Route handler<br/>withAction
  participant FN as Postgres function<br/>swap_twoway (SECURITY DEFINER)
  participant SH as shift rows
  participant SW as swap_log (history)
  participant AL as action_log (audit)
  participant CH as duty_change
  participant RL as Realtime

  Note over UI: Button state comes from an ESTIMATED server time<br/>(monotonic clock + offset from periodic samples).<br/>Display only. Never the authority.

  U->>UI: pick day A and day B, click "Swap"
  UI->>UI: create Idempotency-Key (UUID v4) once per intent<br/>disable button, one in-flight promise (double-click coalesced)
  UI->>PX: POST /api/swap/twoway {dateA, dateB, expectedA, expectedB, reason}<br/>Idempotency-Key header
  PX->>PX: refresh session cookie
  PX->>RT: forward

  rect rgb(240,245,252)
    Note over RT: Edge checks (reject = respond + 1 audit row, except step 1)
    RT->>RT: 1 authenticated? else 401 (no DB write: no identity to attribute)
    RT->>RT: 2 Idempotency-Key is UUID v4 → request_id (missing → generated)
    RT->>RT: 3 Origin matches own origin, 4 JSON content-type, 5 body ≤ 32 KB
    RT->>RT: 6 hand-written input validation → 422 with field details
  end

  RT->>FN: RPC swap_twoway(dateA, dateB, expectedA, expectedB, reason, ctx)<br/>executed with the USER's JWT (never service_role)<br/>ctx = {request_id, api_method, api_path}

  rect rgb(245,245,245)
    Note over FN,AL: ONE database transaction
    FN->>FN: validate ctx shape (bad shape = programming error, raise)
    FN->>FN: role check (from JWT e-mail + provider claim)<br/>no role → return FORBIDDEN_ROLE, no log row

    FN->>AL: replay lookup by request_id (1st)
    alt request_id already has a stored result of THIS caller
      AL-->>FN: stored result
      FN-->>RT: same result + replayed=true (no side effects)
    else new request
      FN->>SH: SELECT … WHERE date IN (A,B) ORDER BY date FOR UPDATE<br/>(fixed order → two crossing swaps cannot deadlock)
      Note right of SH: A concurrent swap touching A or B<br/>blocks here until we commit or roll back
      FN->>AL: replay lookup (2nd)<br/>a duplicate that waited on the lock finds the first commit
      FN->>SW: orphan check: history row exists but audit row was lost?<br/>→ rebuild the original result from the history row

      FN->>FN: now = clock_timestamp()<br/>READ AFTER THE LOCKS ARE HELD<br/>(the decision instant is stored with the audit row)

      FN->>FN: validation ladder (see table below)
      alt any rule rejects
        FN->>AL: INSERT audit row outcome=rejected + error code
        Note over FN,AL: rejection is a RETURN VALUE, not an exception,<br/>so the transaction COMMITS with its audit row
        FN-->>RT: {ok:false, code, message}
      else all rules pass
        FN->>SH: UPDATE shift A ← staff of B, shift B ← staff of A<br/>(updated_at = now, last_request_id)
        FN->>SW: INSERT history row {changes[], reason, request_id}<br/>names SNAPSHOTTED at swap time
        FN->>AL: INSERT audit row outcome=success (sub-block: failure → warning only)
        FN->>CH: INSERT (scope='shift', ref='A,B')
        FN-->>RT: {ok:true, data:{changes, decided_at}}
      end
    end
  end

  RT->>RT: map result code → HTTP status (409 SWAP_LOCKED / SHIFT_CHANGED, 422, 403, 200…)
  RT->>AL: log_finish(request_id, http_status, duration_ms) — best effort, retried once
  RT-->>UI: JSON envelope
  UI->>UI: key kept for retryable failures, dropped after a final outcome
  CH-->>RL: change event (hint only)
  RL-->>U: other users' screens refetch (see S3)
```

### Validation ladder (order matters)

The order was fixed deliberately so the *most meaningful* error wins and cheap checks precede state-dependent ones.

| # | Check | Result code |
|---|---|---|
| 1 | Caller has an on-call role | `FORBIDDEN_ROLE` (no log) |
| 2 | Replay: same `request_id` already done by this caller | stored result, `replayed: true` |
| 3 | *(lock the two rows, then read the DB clock)* | — |
| 4 | Lock-window config readable and valid (broken config ⇒ **fail safe: locked**) | `SWAP_LOCKED` |
| 5 | Input: both dates present and well-formed, `expected*` present, reason ≤ 500 chars | `VALIDATION_ERROR` |
| 6 | Dates differ | `SWAP_SAME_DAY` |
| 7 | Both are weekdays | `SWAP_WEEKEND` |
| 8 | Neither date is in the past | `SWAP_PAST_DATE` |
| 9 | If either date is **today**: inside the lock window → `SWAP_LOCKED`; past the window start → `SWAP_DAY_ENDED` | `SWAP_LOCKED` / `SWAP_DAY_ENDED` |
| 10 | Both days have a person assigned | `SWAP_NO_STAFF` |
| 11 | The two people differ | `SWAP_SAME_STAFF` |
| 12 | Caller is one of the two people (managers may swap on behalf) | `SWAP_NOT_PARTICIPANT` |
| 13 | `expected*` equals the current assignee (the screen was stale?) | `SHIFT_CHANGED` |

### Properties and how they are achieved

| Property | Mechanism |
|---|---|
| **Exactly-once effect** under double-click / retry / flaky network | Client sends one `Idempotency-Key` per intent. DB: `UNIQUE (request_id)` on history and audit tables + replay lookup before *and after* taking locks. |
| **No time-of-check/time-of-use race** | Locks first, clock second, validation third. A request that waited for a lock is judged at the moment it actually runs. |
| **Clock authority** | `clock_timestamp()` in the DB. Client/app-server clocks are never consulted for the decision. |
| **Client clock cannot be forged or drift** | The UI estimator uses only a monotonic clock plus an offset from server samples (S3, snippet 4.6); suspected sleep ⇒ the button dims until re-synced. |
| **No deadlock** | Rows are locked `ORDER BY date`. |
| **Boundary exactness** | Lock iff `from ≤ time-of-day < to`. Tested at 15:29:59.999 (allowed), 15:30:00.000 (blocked), 18:59:59.999 (blocked), 19:00:00.000 (allowed) *and* with the test process in a different timezone. |
| **Audit completeness** | One audit row per request (`UNIQUE`), written by the function in the same transaction; a reconciliation job finds business rows with no audit row. |
| **Logging cannot break the action** | Audit insert is in a sub-block (savepoint); on failure only a warning is raised. |
| **Authorization cannot be spoofed** | Identity read from the JWT inside the function. No "actor" parameter exists. |
| **Fail-safe configuration** | Missing / malformed / `from ≥ to` lock config is treated as *locked*. Losing swaps temporarily is better than unlocking during a live broadcast. |

### Concurrency scenario

```mermaid
sequenceDiagram
  autonumber
  participant A as Request A (swap day 1 ↔ day 2)
  participant B as Request B (swap day 2 ↔ day 3)
  participant DB as shift rows

  A->>DB: lock days 1,2 (ordered) → acquired
  B->>DB: lock days 2,3 (ordered) → BLOCKED on day 2
  A->>DB: read clock, validate, update, insert history + audit
  A->>DB: COMMIT (locks released)
  DB-->>B: lock acquired, rows re-read
  B->>B: expected assignee for day 2 no longer matches
  B->>DB: audit row rejected SHIFT_CHANGED, COMMIT
  Note over B: user sees "The roster just changed, reload and choose again"
```

### Where `pg_cron` appears in this system

`pg_cron` is **not** part of the swap transaction. It drives the *sync* pipeline (S2). The swap lock window does
influence scheduling indirectly: the manual "run maintenance" action is refused while the swap lock window is
active, and the scheduled maintenance job is placed outside the busy window.

---

## S2 — Scheduled data sync (redundant triggers, no double run)

```mermaid
sequenceDiagram
  autonumber
  participant PC as pg_cron (DB)
  participant FN as dispatch_sync_workflow()<br/>SECURITY DEFINER
  participant VT as Vault
  participant PN as pg_net
  participant GH as GitHub API / Actions
  participant JOB as sync job (runner)
  participant DB as Postgres
  participant GS as Google Sheets API

  PC->>FN: at the exact configured time (cron expression, UTC)
  FN->>VT: read decrypted secret "dispatch token"
  FN->>PN: http_post(workflow_dispatch, inputs.trigger='cron')
  PN->>GH: POST …/dispatches (timeout 10 s)
  GH-->>PN: 204 No Content (response row kept for inspection)
  GH->>JOB: start run (CI concurrency group: one run at a time)

  JOB->>DB: expire stale 'running' rows (older than 30 min → failed)
  JOB->>DB: any 'running' row left?
  alt another run is active
    JOB-->>GH: exit — "skipped: another sync is running"
  else free
    JOB->>DB: ONE multi-row INSERT of 'running' rows for every planned source
    Note right of DB: partial unique index (source, site) WHERE status='running'<br/>→ two simultaneous starters: the later INSERT fails with 23505 → skip
    loop each source in order (domain pool first, then per project: sheet, search-console, backlink-tool)
      JOB->>DB: mark started
      JOB->>GS: read tabs (read-only)
      JOB->>JOB: resolve headers → parse → validate → typed issues
      JOB->>DB: upsert base rows
      JOB->>DB: close sync_log row (success / partial / failed + stats + issues)
      Note over JOB: one failing source never stops the others
    end
    JOB->>DB: load dataset (including data from sources that failed this time)
    JOB->>JOB: core computes every page (pure functions)
    JOB->>DB: write snapshots (+ prune)
    JOB-->>GH: exit code reflects failures
  end
```

### Backup path (the native scheduler is known to run hours late)

```mermaid
sequenceDiagram
  autonumber
  participant GH as GitHub native schedule (late)
  participant G as guard step
  participant DB as Postgres (REST)
  participant JOB as sync job

  GH->>G: fires (possibly hours after target)
  G->>DB: any sync_log with status in (success, partial) in the last 12 h?
  alt yes
    G-->>GH: skip all later steps ("primary already ran")
  else no, or the query itself failed
    Note over G: a failed check does NOT skip:<br/>an extra run is better than a missed day
    G->>JOB: run sync
  end
```

Two independent anti-overlap layers: **CI `concurrency` group** (cheap, in the orchestrator) and the **DB mutex**
(authoritative, survives a different orchestrator). Staleness expiry (30 min) prevents a crashed run from holding the
lock forever.

---

## S3 — Realtime refresh (hints, never data)

```mermaid
sequenceDiagram
  autonumber
  participant B as Browser
  participant API as Next.js server
  participant AUTH as Auth (token introspection)
  participant RT as Realtime channel
  participant DB as Postgres

  B->>API: GET /api/realtime-token (session cookie, httpOnly)
  API->>AUTH: getUser() — verify signature and that the account still exists
  AUTH-->>API: ok
  API-->>B: short-lived access token (≥30 s left, ≤1 h) + url + anon key<br/>(refresh token never leaves the server)
  B->>RT: subscribe to duty_change only

  Note over DB: any successful write function ends with<br/>duty_notify(scope, ref) → INSERT duty_change

  DB-->>RT: row event {scope, ref, at}
  RT-->>B: hint
  B->>B: coalesce hints within 200 ms into ONE refetch
  B->>API: refetch page data through the normal server path
  API->>DB: user session → RLS + role checks run again
  DB-->>API: rows
  API-->>B: data
  B->>B: compare state fingerprint — re-render only if changed
```

Degraded modes (state machine shown in the header as a connection badge):

```mermaid
stateDiagram-v2
  [*] --> connecting
  connecting --> live: subscribed
  connecting --> reconnecting: channel lost or token fetch failed
  live --> reconnecting: channel lost
  reconnecting --> live: subscribed (resync once)
  reconnecting --> polling: still down after 3 s grace
  polling --> live: subscribed
  polling --> offline: consecutive poll failures
  offline --> polling: a poll succeeds
  offline --> reconnecting: browser reports online
  live --> offline: browser reports offline
  polling --> offline: browser reports offline
  live --> live: reconcile every 60 s (1 GET + fingerprint compare)
```

Extra resync triggers: tab becomes visible, browser goes back online, one settle-check 5 s after each subscribe
(the realtime server may not be ready to emit yet). Polling interval is 5 s.

Hint parsing is defensive: only a whitelisted `scope` and a short `ref` are accepted; anything unexpected (empty
payload, wrong type, unknown scope) degrades to "refetch everything" — when in doubt, reload rather than miss a change.

Why this is safe: the channel carries `{scope, ref, timestamp}` only. A leaked token or replayed event reveals
nothing, and a missed event is repaired by reconcile/poll — correctness never depends on the channel.

---

## S4 — Fail-closed log archiving

```mermaid
sequenceDiagram
  autonumber
  participant J as maintenance job
  participant DB as Postgres (service role)
  participant SH as Archive sheet (write access)

  J->>DB: batch = oldest ≤ N audit rows older than cutoff (e.g. 90 days)
  loop until no rows or batch cap reached
    J->>SH: append rows whose request_id is NOT already in the sheet (re-runnable, no duplicates)
    J->>SH: RE-READ the sheet now (never reuse an earlier read)
    J->>J: every request_id of this batch present in the sheet?
    alt any id missing
      J-->>J: STOP — delete nothing, fail the job loudly
    else all confirmed
      J->>DB: DELETE WHERE id IN (exactly this batch's ids)
      Note over J,DB: delete by id, never by timestamp:<br/>no window between "saved" and "deleted"
    end
  end
  J->>DB: purge realtime hint rows older than 1 day
  J->>DB: reconcile: business rows missing an audit row (last 2 days)
  J->>DB: measure sizes (log table, whole DB) vs thresholds
  J-->>J: exit non-zero if gaps, thresholds exceeded, archive failure or any step could not run
```

Crash-safety: if the delete fails after the append succeeded, the next run finds the rows already archived, skips the
append, and only deletes. Archived columns exclude parameters, e-mail and raw results.
