# 4. Sanitized Code Snippets & Pseudocode

Every snippet is a **standalone rewrite** that illustrates a technique; none is copied from the source files, and
all business identifiers are generic. TypeScript snippets are runnable in spirit but omit imports/types that do not
matter to the idea. SQL is PostgreSQL / Supabase flavoured.

| § | Technique | Used by |
|---|---|---|
| 4.1 | Header-anchor spreadsheet column resolver | Sync pipeline |
| 4.2 | Month-group and two-tier headers, tolerant of `#REF!` | Sync pipeline |
| 4.3 | Default-deny Row-Level Security | Whole database |
| 4.4 | Role helper functions (JWT → role) | RLS + RPC |
| 4.5 | Idempotent, audited write function (contract) | On-call writes |
| 4.6 | Lock-window decision + client server-clock estimator | Shift swap |
| 4.7 | Sync mutex via partial unique index | Sync pipeline |
| 4.8 | Fail-closed archive-then-delete | Maintenance |
| 4.9 | Pointer-based rotation algorithm | Roster generation |
| 4.10 | `pg_cron` → HTTP dispatch with Vault secret | Scheduling |
| 4.11 | Log sanitizer (secret redaction, bounded size) | Audit log |

---

## 4.1 Header-anchor column resolver

**Problem.** Source sheets are edited by hand. Columns get inserted, a title row may move down, several logical
tables sit side by side in one tab. Column letters are not a stable contract; *titles* are.

**Idea.** (Helper functions such as `colLetter`, `error`, `warning` are omitted for brevity.)
1. Find the header row = first of the top *N* rows that contains **all** anchor titles of the tab.
2. Split that row into **blocks** at *block-start* titles. A block ends where the next one begins (by position) —
   never "at the first empty column".
3. Inside a block, match each field by normalized title (with aliases). Duplicate title ⇒ error, never a guess.
   Unknown title ⇒ warning, never a failure.

```ts
type FieldSpec = { titles: string[]; required: boolean; occurrence?: number };
type BlockSpec = { start?: string; fields: Record<string, FieldSpec> };
type TabSpec = { headerAnchors: string[]; blocks: Record<string, BlockSpec> };

type Issue = {
  type: "header_not_found" | "missing_required" | "duplicate_header" | "new_column";
  level: "error" | "warning";
  blocking: boolean;          // true ⇒ this tab is skipped, previous data is kept
  block?: string; field?: string; columns?: string[]; message: string;
};

// NFC + collapse whitespace + trim + lower-case: "  Ref  Link " == "ref link"
const norm = (s?: string) => (s ?? "").normalize("NFC").replace(/\s+/g, " ").trim().toLowerCase();

function findHeaderRow(grid: string[][], anchors: string[], maxRows = 10): number {
  const wanted = anchors.map(norm);
  for (let r = 0; r < Math.min(maxRows, grid.length); r++) {
    const cells = new Set(grid[r].map(norm));
    if (wanted.every((a) => cells.has(a))) return r;
  }
  return -1;
}

/** Blocks start at their start-title and end right before the next block (by column position). */
function splitBlocks(row: string[], blocks: Record<string, BlockSpec>) {
  const n = row.map(norm);
  const starts = Object.entries(blocks)
    .map(([name, b]) => ({ name, start: b.start === undefined ? 0 : n.indexOf(norm(b.start)) }))
    .filter((b) => b.start >= 0)
    .sort((a, b) => a.start - b.start);
  return Object.fromEntries(
    starts.map((b, i) => [b.name, { start: b.start, end: starts[i + 1]?.start ?? row.length }]),
  );
}

function resolveFields(row: string[], range: { start: number; end: number }, blockName: string, block: BlockSpec) {
  const n = row.map(norm);
  const columns: Record<string, number> = {};
  const issues: Issue[] = [];
  const known = new Set<number>();

  for (const [field, spec] of Object.entries(block.fields)) {
    const wanted = spec.titles.map(norm);                        // title + aliases
    const hits: number[] = [];
    for (let i = range.start; i < range.end; i++) if (wanted.includes(n[i])) hits.push(i);
    hits.forEach((i) => known.add(i));

    if (spec.occurrence !== undefined) {                          // "the 2nd column titled X"
      const idx = hits[spec.occurrence - 1];
      if (idx !== undefined) columns[field] = idx;
      else if (spec.required) issues.push(missing(blockName, field, spec));
    } else if (hits.length > 1) {                                 // never guess which duplicate is right
      issues.push({ type: "duplicate_header", level: "error", blocking: spec.required, block: blockName, field,
        columns: hits.map(colLetter), message: `"${spec.titles[0]}" appears ${hits.length} times` });
    } else if (hits.length === 1) columns[field] = hits[0];
    else if (spec.required) issues.push(missing(blockName, field, spec));
  }

  // Titles that belong to no field: surface them (maybe a rename), but do not block
  for (let i = range.start; i < range.end; i++)
    if (n[i] && !known.has(i))
      issues.push({ type: "new_column", level: "warning", blocking: false, block: blockName,
        columns: [colLetter(i)], message: `Unknown column "${row[i].trim()}" (${colLetter(i)}) — ignored` });

  return { columns, issues };
}

const missing = (block: string, field: string, s: FieldSpec): Issue => ({
  type: "missing_required", level: "error", blocking: true, block, field,
  message: `Required title "${s.titles[0]}" not found in block ${block}`,
});

function resolveTab(grid: string[][], spec: TabSpec) {
  const h = findHeaderRow(grid, spec.headerAnchors);
  if (h < 0) return { ok: false, issues: [{ type: "header_not_found", level: "error", blocking: true,
    message: "No row contains all anchor titles in the first 10 rows" } as Issue] };

  const row = grid[h];
  const ranges = splitBlocks(row, spec.blocks);
  const out = { blocks: {} as Record<string, Record<string, number>>, issues: [] as Issue[] };
  for (const [name, block] of Object.entries(spec.blocks)) {
    const range = ranges[name];
    if (!range) { /* block-start title missing ⇒ every required field of the block is missing */ continue; }
    const r = resolveFields(row, range, name, block);
    out.blocks[name] = r.columns;
    out.issues.push(...r.issues);
  }
  return { ok: !out.issues.some((i) => i.blocking), headerRow: h, ...out };
}
```

**Worked example** — one tab holding three tables side by side:

| col | A | B | C | D | E | F | G | H |
|---|---|---|---|---|---|---|---|---|
| header row | `Domain` | `URL` | `Done date` | *(blank)* | `Assignment` | `Target` | `Week` | `Total` |

Anchors `["Domain","Assignment","Week"]`, block starts `Domain` / `Assignment` / `Week`:
the header row is found even if two banner rows sit above it; block *output* spans A–D, *assignment* E–F, *report*
G–H. The blank D belongs to the output block (blocks are cut by the next start title, not by blank columns).
If someone inserts a column before `Total`, the code needs no change.

**Degrade-gracefully policy.** `error` + `blocking` ⇒ that tab/block is skipped and the previous data stays;
`error` non-blocking and `warning` ⇒ read anyway, record the issue. Issues are stored per run, so a single broken
block is visible in the UI instead of failing the whole sync.

---

## 4.2 Month-group headers and two-tier headers

Two real-world layouts that defeat plain title matching:

* **Month groups:** a label like `Month 10/2026` sits one row **above** the header row; the sub-columns underneath
  are matched by *position inside the group* because their own titles are unreliable (a formula may show `#REF!`).
* **Two-tier headers:** the upper tier is a person's name in a **merged** cell (value only in the first cell of the
  merge); the lower tier is a site code. Join by *name*, never by column order.

```ts
const REF_ERROR = "#ref!";

/** Month-group block: labels in the row ABOVE the header, sub-columns taken positionally. */
function resolveMonthGroups(headerRow: string[], upperRow: string[], range: {start:number; end:number},
                            expectedSubColumns: string[], labelPattern: RegExp) {
  const labels: { month: string; col: number }[] = [];
  for (let i = range.start; i < range.end; i++) {
    const m = (upperRow[i] ?? "").normalize("NFC").trim().match(labelPattern);   // e.g. /^Month (\d{2})\/(\d{4})$/
    if (m) labels.push({ month: `${m[2]}-${m[1]}`, col: i });
  }
  const groups: { month: string; columns: Record<string, number> }[] = [];
  const issues: Issue[] = [];

  labels.forEach(({ month, col }, idx) => {
    const end = labels[idx + 1]?.col ?? range.end;                   // a group runs to the next label
    const columns: Record<string, number> = {};
    let broken = false;
    expectedSubColumns.forEach((expected, k) => {
      const i = col + k;
      const title = norm(headerRow[i]);
      if (i >= end || (title !== norm(expected) && title !== REF_ERROR)) {
        broken = true;                                               // wrong title ⇒ do not guess
        issues.push(error(`Month ${month}: column ${colLetter(i)} is "${headerRow[i]}", expected "${expected}"`));
        return;
      }
      if (title === REF_ERROR)                                       // tolerate: accept by POSITION, but warn
        issues.push(warning(`Month ${month}: "#REF!" at ${colLetter(i)}, treated as "${expected}" by position`));
      columns[expected] = i;
    });
    if (!broken) groups.push({ month, columns });                    // one broken month never hides the others
  });
  return { groups, issues };
}

/** Two-tier header: upper = merged name cell (forward-filled), lower = site code. Join by name. */
function resolveTwoTier(upper: string[], lower: string[], range: {start:number; end:number},
                        knownNames: string[], siteCodes: string[]) {
  const found: Record<string, Record<string, number>> = {};         // name → site → column
  let currentName = "";
  for (let i = range.start; i < range.end; i++) {
    if ((upper[i] ?? "").trim()) currentName = upper[i].trim();      // merged cell: value only in the first cell
    const site = siteCodes.find((c) => norm(c) === norm(lower[i]));
    if (!site) continue;
    const name = knownNames.find((e) => norm(e) === norm(currentName));
    if (name) (found[name] ??= {})[site] = i;                        // unknown name ⇒ warning elsewhere, not a crash
  }
  return found;   // data rows start 2 rows below the header when a lower tier exists
}
```

Note on the design: the Sheets API **truncates trailing empty cells** of a row, so the header row may be shorter than
the tier below it. The resolver pads the header row to the widest of (row above, header, row below) before
splitting blocks, otherwise the last block silently loses its last columns.

---

## 4.3 Default-deny Row-Level Security

**Goal.** A table created tomorrow, by anyone, is invisible to browsers until someone consciously opens it.
Denied access must fail **loudly** (`42501 permission denied`), not return an empty list that looks like "no data".

```sql
-- 1. RLS on every table (rows are invisible without a matching policy)
alter table site          enable row level security;
alter table output        enable row level security;
alter table snapshot      enable row level security;
alter table sync_log      enable row level security;
alter table setting       enable row level security;
-- … every table in the schema

-- 2. Take away what the platform grants by default to browser-facing roles
revoke all on all tables    in schema public from anon, authenticated;
revoke all on all sequences in schema public from anon, authenticated;
revoke all on all functions in schema public from anon, authenticated;

-- 3. …and make that the default for everything created later
alter default privileges in schema public revoke all on tables    from anon, authenticated;
alter default privileges in schema public revoke all on sequences from anon, authenticated;
alter default privileges in schema public revoke all on functions from anon, authenticated;

-- 4. Re-open the minimum, in two independent layers (privilege AND policy)
grant select on snapshot, sync_log to anon;
create policy anon_read_snapshot on snapshot for select to anon using (page <> 'settings');
create policy anon_read_sync_log on sync_log for select to anon using (true);

-- 5. Signed-in users: same public reads, plus the settings page only for entitled roles
grant select on snapshot, sync_log to authenticated;
create policy auth_read_snapshot on snapshot for select to authenticated
  using (page <> 'settings' or public.is_settings_user());
```

Why two layers? With *only* policies, a missing policy hides rows (empty result) and a mistaken `GRANT` can still
allow writes. With the privilege layer revoked, `SELECT output` fails with `42501` even before a policy is evaluated,
and write access never depends on whether someone remembered a policy.

**Write protection for configuration** — a policy that checks both the old and the new row, so a key cannot be
renamed into a protected one:

```sql
create policy auth_update_setting on setting for update to authenticated
  using      (public.is_settings_user() and public.is_editable_setting_key(key))
  with check (public.is_settings_user() and public.is_editable_setting_key(key));
```

**Realtime exposure, deny by default as well:**

```sql
-- Only a content-free "something changed" table is published
alter publication supabase_realtime add table public.change_hint;   -- scope, ref, at — no business data
```

---

## 4.4 Role helpers: JWT → role

```sql
-- SECURITY DEFINER: can read the access table even though the caller cannot.
-- Empty search_path: every reference is schema-qualified, so nobody can shadow a function or table.
create function public.current_actor()
returns table (email text, role text, duty_roles text[], employee_id bigint)
language sql stable security definer set search_path = ''
as $$
  select a.email, a.role, a.duty_roles, a.employee_id
  from public.app_access a
  where a.email = lower(auth.jwt() ->> 'email')
    -- app_metadata is writable only by the service role; user_metadata is user-editable and is NEVER consulted.
    -- The provider claim proves the e-mail was verified by the identity provider.
    and coalesce(auth.jwt() -> 'app_metadata' -> 'providers', '[]'::jsonb) @> '"google"'::jsonb
$$;

create function public.can_view() returns boolean
language sql stable security definer set search_path = ''
as $$ select coalesce((select a.role in ('admin','dev') or a.duty_roles && array['staff','po']
                       from public.current_actor() a), false) $$;

-- Postgres grants EXECUTE to PUBLIC by default — including anonymous callers
revoke execute on function public.current_actor(), public.can_view() from public;
grant  execute on function public.can_view() to authenticated;      -- current_actor stays internal
```

**Gotcha worth documenting (found by the parallel TS↔SQL tests).** A user with *only* `duty_roles` has `role = NULL`.
In SQL, `NULL IN ('admin','dev')` is `NULL`, not `false`, so `not is_manager` became `NULL` and the "caller must be a
participant" check was **silently skipped**. Fix: wrap every role predicate in `coalesce(…, false)`.

---

## 4.5 Idempotent, audited write function (contract)

Every write function in the on-call module follows the same five-step contract, so correctness is a property of the
*shape*, not of each author's memory.

```sql
create function public.do_something(p_params …, p_ctx jsonb) returns jsonb
language plpgsql security definer set search_path = '' as $$
declare v_ctx record; v_now timestamptz; v_result jsonb; v_replay jsonb;
begin
  -- 1. Context: {request_id uuid, api_method, api_path}. Malformed ⇒ RAISE (programming error, not user error)
  select * into v_ctx from public.ctx(p_ctx);

  -- 2. Authorization. No role ⇒ return a value and write NO log row (no trusted identity to attribute it to)
  if not public.can_view() then return public.result(false, 'FORBIDDEN_ROLE', 'Not allowed'); end if;

  -- 3. Replay: this caller already completed this request_id ⇒ return the stored result unchanged
  v_replay := public.replay(v_ctx.request_id);
  if v_replay is not null then return v_replay; end if;

  -- 3b. Lock the rows this action will touch, in a FIXED order (no deadlocks)
  perform 1 from public.shift s where s.date in (…) order by s.date for update;
  -- 3c. Replay again: a duplicate that waited for the lock now finds the first request's commit
  v_replay := public.replay(v_ctx.request_id);
  if v_replay is not null then return v_replay; end if;

  -- 4. Decision instant: read AFTER the locks, then validate. A rejection is a RETURN VALUE so the
  --    transaction commits together with its 'rejected' audit row.
  v_now := clock_timestamp();
  if <rule fails> then
    return public.complete(p_ctx, 'domain.action', 'do_something',
             public.result(false, 'SOME_CODE', 'Human message'), v_params, v_now);
  end if;

  -- … do the work …

  -- 5. Exactly one audit row; notify realtime on success; return the result
  v_result := public.result(true, 'OK', 'Done', jsonb_build_object('decided_at', v_now));
  return public.complete(p_ctx, 'domain.action', 'do_something', v_result, v_params, v_now,
                         p_scope => 'shift', p_ref => '2026-01-05');
end $$;

-- The completion helper enforces the contract itself:
--   * a successful result WITHOUT a notify scope RAISES (so a test catches a function that forgot realtime)
--   * the audit insert is wrapped in a sub-block: failure ⇒ RAISE WARNING, never fails the action
--   * a stored result larger than 4 KB drops its `data` (replay then returns only ok/code/message)
```

---

## 4.6 Lock-window decision and the client-side server clock

### Decision (single source of truth is a config row, evaluated with the DB clock)

```ts
type LockConfig = { from: string; to: string; enabled?: boolean };   // e.g. {"from":"15:30:00","to":"19:00:00"}
type Decision = { locked: boolean; configError: null | "missing" | "bad_format" | "not_ascending"; nextChangeMs: number | null };

function decideLock(cfg: unknown, msOfDay: number): Decision {
  const parsed = parseConfig(cfg);                              // strict "HH:MM[:SS[.mmm]]" → ms, from < to required
  // FAIL SAFE: a broken config means LOCKED. Temporarily losing swaps beats unlocking during a live event.
  if (!parsed.ok) return { locked: true, configError: parsed.error, nextChangeMs: null };

  const { fromMs, toMs, enabled } = parsed;
  const locked = enabled && msOfDay >= fromMs && msOfDay < toMs;     // half-open: [from, to)
  const next = locked ? toMs : msOfDay < fromMs ? fromMs : fromMs + 86_400_000;
  return { locked, configError: null, nextChangeMs: next };
}
```

Boundary table (all four are automated tests, executed also with the process in another timezone):

| Instant (local) | Expected |
|---|---|
| 15:29:59.999 | swap allowed |
| 15:30:00.000 | **blocked** |
| 18:59:59.999 | **blocked** |
| 19:00:00.000 | allowed |

The same function exists in SQL (`lock_decision(timestamptz)`); it takes the instant as a **parameter** rather than
reading the clock itself, so the swap function can read the clock *after* taking row locks and pass it in.

Rule evolution worth knowing: the window originally blocked every swap; it was later narrowed to block only swaps
that involve **today's** shift (other days stay swappable inside the window). Today's shift additionally becomes
"started" at `from` and is never reopened at `to`.

### Estimating server time in the browser (display only)

```ts
// offset = serverTime - midpoint(sentAt, receivedAt); estimate = monotonicNow + offset.
// Only a MONOTONIC clock is used for the estimate, so changing the OS clock cannot move displayed time.
// The wall clock is used as a SIGNAL only — to detect sleep or a changed system clock.
function createServerClock(monotonic = () => performance.now(), wall = () => Date.now(),
                           { maxRttMs = 3000, staleAfterMs = 600_000, suspendGapMs = 120_000 } = {}) {
  let best: { offset: number; at: number; rtt: number } | null = null;
  let suspected = false, lastMono: number | null = null, lastWall: number | null = null;

  const tick = () => {                                           // call at least once per second
    const m = monotonic(), w = wall();
    if (lastMono !== null && Math.abs((w - lastWall!) - (m - lastMono)) > suspendGapMs) suspected = true;
    lastMono = m; lastWall = w;
    return m;
  };

  return {
    accept({ serverNowMs, sentAt, receivedAt }: { serverNowMs: number; sentAt: number; receivedAt: number }) {
      const rtt = receivedAt - sentAt;
      if (rtt < 0 || rtt > maxRttMs) return false;               // too noisy to trust
      if (best && rtt > best.rtt && receivedAt - best.at < 60_000) return false;   // keep a tighter sample while it is under a minute old
      best = { offset: serverNowMs - (sentAt + rtt / 2), at: receivedAt, rtt };
      suspected = false; lastWall = wall(); lastMono = monotonic();
      return true;
    },
    read() {
      const m = tick();
      if (!best) return { nowMs: null, stale: true, reason: "no_sample" as const };
      const age = m - best.at;
      const reason = suspected ? "suspected_sleep" : age > staleAfterMs ? "too_old" : null;
      return { nowMs: m + best.offset, stale: reason !== null, reason, uncertaintyMs: best.rtt / 2 };
    },
  };
}
```

UI policy: **no safety margin** is added around the lock boundary. The button works until the last millisecond; a
request that arrives late is rejected by the database with a clear message. The UI trusts the DB as the only judge,
and dims the button (then re-syncs) whenever `stale` is true.

---

## 4.7 Sync mutex via a partial unique index

A "check then insert" lock is racy: two starters both see "nothing running". Let the database arbitrate.

```sql
create unique index sync_log_one_running_per_source
  on sync_log (source, site_id) nulls not distinct   -- the shared source has site_id NULL
  where status = 'running';
```

```ts
async function acquire(db: Db, plan: PlannedSource[]): Promise<number[] | null> {
  await expireStale(db, 30 /* minutes */);                      // a crashed run must not hold the lock forever
  // ONE multi-row INSERT opens a 'running' row for every source of this run.
  const { data, error } = await db.from("sync_log")
    .insert(plan.map((p) => ({ source: p.source, site_id: p.siteId, status: "running" }))).select("id");
  if (error?.code === "23505") return null;                      // unique_violation ⇒ someone else holds it ⇒ skip
  if (error) throw error;
  return data.map((r) => r.id);                                  // each row is closed (success|partial|failed) as its source finishes
}
```

Finalization uses `try/finally`: if the process stops mid-way, rows not yet reached are closed as `failed`.

---

## 4.8 Fail-closed archive-then-delete

```ts
async function archiveAndDelete(store: LogStore, sheet: SheetApi, cutoffIso: string, batchSize = 1000, maxBatches = 200) {
  for (let b = 0; b < maxBatches; b++) {
    const batch = await store.listOlderThan(cutoffIso, batchSize);          // oldest first
    if (batch.length === 0) break;

    // 1. Append only rows not already present (keyed by request_id) ⇒ safe to re-run
    const present = await sheet.readRequestIds();
    const fresh = batch.filter((r) => !present.has(r.requestId));
    if (fresh.length) await sheet.append(fresh.map(toSheetRow));            // 7 columns; NO params, e-mail or raw result

    // 2. RE-READ immediately before deleting. Never reuse the earlier read.
    const confirmed = await sheet.readRequestIds();
    const missingIds = batch.filter((r) => !confirmed.has(r.requestId));
    if (missingIds.length) throw new ArchiveError("not_confirmed", `${missingIds.length} rows`);   // delete NOTHING

    // 3. Delete exactly these ids — never "older than T". No gap between "saved" and "deleted".
    await store.deleteByIds(batch.map((r) => r.id));
  }
}
```

Failure semantics: any exception stops the run before the next delete; rows already appended but not deleted are
skipped at append time and deleted next run. The job exits non-zero so CI alerts by e-mail.

---

## 4.9 Pointer-based rotation algorithm

**Rule.** One person per working day, fixed order, wrapping. Compute the next person from *the previous person's
position*, not from "day number modulo headcount".

Why not modulo? A holiday, a departure or a newcomer would reshuffle everybody. With a pointer:

* **Holiday:** no shift and **no turn consumed** — the person due that day works the next working day.
* **Departure:** skipped from the effective date; everyone else keeps their order.
* **Newcomer:** inserted between two people by a fractional `position` (e.g. `2.5`); future turns are correct, past days are untouched.

```ts
type Member = { staffId: number; position: number; activeFrom: string; activeTo?: string | null };   // [activeFrom, activeTo)

function generate(anchorDate: string, anchorStaffId: number, members: Member[], skipDates: Set<string>,
                  from: string, to: string) {
  const shifts: { date: string; staffId: number }[] = [];
  let prev: { staffId: number; position: number } | null = null;

  for (const day of workingDays(anchorDate, to)) {                       // Mon–Fri
    if (skipDates.has(day)) continue;                                    // holiday: no shift, no turn used
    const active = distinctByStaff(members.filter((m) => m.activeFrom <= day && (!m.activeTo || day < m.activeTo)))
      .sort((a, b) => a.position - b.position);

    const staffId = prev === null
      ? anchorStaffId                                                    // first working day belongs to the anchor
      : (active.find((m) => m.position > prev!.position) ?? active[0]).staffId;   // next higher position, else wrap

    prev = { staffId, position: members.find((m) => m.staffId === staffId)!.position };
    if (day >= from) shifts.push({ date: day, staffId });
  }
  return shifts;
}
```

Worked example: members `A(1) B(2) C(3) D(4)`, anchor `A` on working day 1, day 3 is a holiday, `C` leaves
effective day 5 (`activeTo = day 5`):

| Working day | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
|---|---|---|---|---|---|---|---|---|---|
| On duty | A | B | *holiday* | C | D | A | B | D | A |

`C` was due on the holiday and works the next working day (no turn consumed). From day 5 `C` is inactive, so the
pointer after `C` (position 3) lands on `D`; on day 8 the next higher position after `B` skips the departed `C`.

Persistence uses two columns: `staff_id` (current) and `base_staff_id` (by rule). A swap changes only `staff_id`;
"reset to schedule" copies `base_staff_id` back. The same algorithm exists in SQL; a parallel test runs one scenario
table against both implementations.

---

## 4.10 `pg_cron` → HTTP dispatch with a Vault secret

```sql
-- One-time, run by hand on the cloud project (not a migration: contains a secret)
select vault.create_secret('<GITHUB_TOKEN>', 'ci_dispatch_token', 'fine-grained token, Actions: write');

create or replace function public.dispatch_workflow() returns bigint
language plpgsql security definer set search_path = public, extensions
as $$
declare token text; req bigint;
begin
  select decrypted_secret into token from vault.decrypted_secrets where name = 'ci_dispatch_token';
  if token is null then raise exception 'secret ci_dispatch_token missing'; end if;
  select net.http_post(
    url     := 'https://api.github.com/repos/<ORG>/<REPO>/actions/workflows/sync.yml/dispatches',
    headers := jsonb_build_object('Authorization', 'Bearer ' || token,
                                  'Accept', 'application/vnd.github+json',
                                  'Content-Type', 'application/json'),
    body    := jsonb_build_object('ref', 'main', 'inputs', jsonb_build_object('trigger', 'cron')),
    timeout_milliseconds := 10000) into req;
  return req;                                                 -- response lands in net._http_response (204 = accepted)
end $$;

revoke all on function public.dispatch_workflow() from public, anon, authenticated;   -- DB owner / cron only

-- Idempotent scheduling: remove any job with this name, then create it. 12:30 UTC.
select cron.unschedule(jobid) from cron.job where jobname = 'daily-sync';
select cron.schedule('daily-sync', '30 12 * * *', $$select public.dispatch_workflow()$$);
```

Design notes: the secret never appears in the job definition (it is read from the vault at call time); the function
is the only place that knows it; rerunning the file is safe; the workflow input `trigger = 'cron'` is stored in the
sync log so runs triggered by schedule and by hand are distinguishable.

---

## 4.11 Log sanitizer (secret redaction, bounded size)

Audit rows must never become a secret store. Sanitization happens **in the database**, so no code path can bypass it.

```sql
-- Mask token-like strings, then truncate
create function clean_text(p text, p_max int) returns text language sql immutable as $$
  select left(
    regexp_replace(regexp_replace(regexp_replace(regexp_replace(p,
      'eyJ[A-Za-z0-9_-]+(\.[A-Za-z0-9_-]+)*', '[redacted]', 'g'),   -- JWTs
      'bearer\s+\S+',                         '[redacted]', 'gi'),  -- Authorization headers
      'sb-[A-Za-z0-9_-]+',                    '[redacted]', 'g'),   -- vendor key prefixes
      '[A-Za-z0-9+/=_-]{40,}',                '[redacted]', 'g'),   -- long base64/hex blobs
    p_max) $$;

-- Recursive JSON cleaner: bounded depth (3), keys (20), array length (20), strings (200 chars);
-- unknown key shapes are dropped; values under sensitive-looking keys are masked.
create function clean_json(v jsonb, depth int) returns jsonb language plpgsql immutable as $$
declare k text; r jsonb := '{}'; n int := 0; i int;
begin
  case jsonb_typeof(v)
    when 'string' then return to_jsonb(clean_text(v #>> '{}', 200));
    when 'object' then
      if depth >= 3 then return '"[truncated]"'; end if;
      for k in select jsonb_object_keys(v) order by 1 loop
        continue when k !~ '^[a-z][a-z0-9_]{0,39}$';                    -- only snake_case keys survive
        exit when n >= 20; n := n + 1;
        r := r || jsonb_build_object(k,
               case when k ~ '(password|token|secret|cookie|authorization|api_?key|jwt|service_?role|session|credential)'
                    then '"[redacted]"'::jsonb else clean_json(v -> k, depth + 1) end);
      end loop;
      return r;
    when 'array' then
      if depth >= 3 then return '"[truncated]"'; end if;
      r := '[]';
      for i in 0 .. least(jsonb_array_length(v), 20) - 1 loop r := r || jsonb_build_array(clean_json(v -> i, depth + 1)); end loop;
      return r;
    else return v;
  end case;
end $$;
```

Final guard: if the cleaned parameters still exceed 4 KB, they are replaced by `{"truncated": true, "bytes": n}`.
