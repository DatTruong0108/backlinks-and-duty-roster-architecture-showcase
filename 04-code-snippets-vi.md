# 4. Các đoạn mã & Mã giả đã được làm sạch

Mỗi đoạn mã là một **bản viết lại độc lập** để minh họa một kỹ thuật; không đoạn nào được copy từ file mã nguồn thực sự, và mọi định danh nghiệp vụ đều là tên chung chung. Mã TypeScript chạy được về mặt tinh thần nhưng bỏ qua việc import/các kiểu dữ liệu không liên quan đến ý tưởng chính. Mã SQL mang hương vị của PostgreSQL / Supabase.

| § | Kỹ thuật | Được dùng bởi |
|---|---|---|
| 4.1 | Bộ phân giải cột bảng tính bằng header-anchor | Pipeline Đồng bộ |
| 4.2 | Header nhóm theo tháng và header phân hai tầng, chấp nhận `#REF!` | Pipeline Đồng bộ |
| 4.3 | Row-Level Security từ chối mặc định | Toàn bộ cơ sở dữ liệu |
| 4.4 | Các hàm hỗ trợ Role (JWT → role) | RLS + RPC |
| 4.5 | Hàm ghi lũy đẳng, được audit (hợp đồng) | Thao tác ghi trực ban |
| 4.6 | Quyết định khóa khung giờ + ước lượng giờ server client | Đổi ca trực |
| 4.7 | Mutex đồng bộ thông qua partial unique index | Pipeline Đồng bộ |
| 4.8 | Lưu trữ rồi xóa một cách an toàn fail-closed | Bảo trì |
| 4.9 | Thuật toán luân phiên ca dựa trên con trỏ | Tạo lịch |
| 4.10 | `pg_cron` → HTTP dispatch với secret Vault | Lên lịch |
| 4.11 | Bộ lọc Log (che giấu mật khẩu, giới hạn kích thước) | Audit log |

---

## 4.1 Bộ phân giải cột bảng tính bằng header-anchor

**Vấn đề.** Sheet nguồn được con người chỉnh sửa thủ công. Các cột có thể bị chèn thêm, hàng tiêu đề bị đẩy xuống dưới, nhiều bảng logic nằm sát cạnh nhau trong cùng một tab. Ký tự cột (ví dụ A, B, C) không phải là một hợp đồng ổn định; *tiêu đề* mới là chuẩn.

**Ý tưởng.** (Các hàm phụ như `colLetter`, `error`, `warning` được lược bỏ cho gọn.)
1. Tìm hàng tiêu đề (header row) = hàng đầu tiên trong top *N* hàng chứa **tất cả** các tiêu đề neo (anchor) của tab.
2. Chia hàng đó thành **các khối (blocks)** bằng các tiêu đề bắt-đầu-khối (block-start). Một khối kết thúc ở nơi khối tiếp theo bắt đầu (dựa vào vị trí) — tuyệt đối không kết thúc tại "cột trống đầu tiên".
3. Bên trong một khối, khớp từng trường theo tiêu đề đã chuẩn hóa (kèm theo bí danh - alias). Tiêu đề bị lặp ⇒ báo lỗi, không bao giờ tự đoán. Tiêu đề lạ ⇒ cảnh báo, không làm hỏng đồng bộ.

```ts
type FieldSpec = { titles: string[]; required: boolean; occurrence?: number };
type BlockSpec = { start?: string; fields: Record<string, FieldSpec> };
type TabSpec = { headerAnchors: string[]; blocks: Record<string, BlockSpec> };

type Issue = {
  type: "header_not_found" | "missing_required" | "duplicate_header" | "new_column";
  level: "error" | "warning";
  blocking: boolean;          // true ⇒ tab này bị bỏ qua, dữ liệu cũ giữ nguyên
  block?: string; field?: string; columns?: string[]; message: string;
};

// NFC + xóa khoảng trắng thừa + trim + chữ thường: "  Ref  Link " == "ref link"
const norm = (s?: string) => (s ?? "").normalize("NFC").replace(/\s+/g, " ").trim().toLowerCase();

function findHeaderRow(grid: string[][], anchors: string[], maxRows = 10): number {
  const wanted = anchors.map(norm);
  for (let r = 0; r < Math.min(maxRows, grid.length); r++) {
    const cells = new Set(grid[r].map(norm));
    if (wanted.every((a) => cells.has(a))) return r;
  }
  return -1;
}

/** Khối bắt đầu tại start-title và kết thúc ngay trước khối tiếp theo (theo vị trí cột). */
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
    const wanted = spec.titles.map(norm);                        // tiêu đề + bí danh
    const hits: number[] = [];
    for (let i = range.start; i < range.end; i++) if (wanted.includes(n[i])) hits.push(i);
    hits.forEach((i) => known.add(i));

    if (spec.occurrence !== undefined) {                          // "cột thứ 2 có tên là X"
      const idx = hits[spec.occurrence - 1];
      if (idx !== undefined) columns[field] = idx;
      else if (spec.required) issues.push(missing(blockName, field, spec));
    } else if (hits.length > 1) {                                 // không bao giờ đoán cột trùng lặp
      issues.push({ type: "duplicate_header", level: "error", blocking: spec.required, block: blockName, field,
        columns: hits.map(colLetter), message: `"${spec.titles[0]}" xuất hiện ${hits.length} lần` });
    } else if (hits.length === 1) columns[field] = hits[0];
    else if (spec.required) issues.push(missing(blockName, field, spec));
  }

  // Cột lạ không khớp với field nào: ghi nhận lại (đề phòng việc đổi tên), nhưng không chặn
  for (let i = range.start; i < range.end; i++)
    if (n[i] && !known.has(i))
      issues.push({ type: "new_column", level: "warning", blocking: false, block: blockName,
        columns: [colLetter(i)], message: `Cột lạ "${row[i].trim()}" (${colLetter(i)}) — bỏ qua` });

  return { columns, issues };
}

const missing = (block: string, field: string, s: FieldSpec): Issue => ({
  type: "missing_required", level: "error", blocking: true, block, field,
  message: `Tiêu đề bắt buộc "${s.titles[0]}" không tìm thấy ở khối ${block}`,
});

function resolveTab(grid: string[][], spec: TabSpec) {
  const h = findHeaderRow(grid, spec.headerAnchors);
  if (h < 0) return { ok: false, issues: [{ type: "header_not_found", level: "error", blocking: true,
    message: "Không có hàng nào chứa đủ tất cả anchor ở 10 hàng đầu tiên" } as Issue] };

  const row = grid[h];
  const ranges = splitBlocks(row, spec.blocks);
  const out = { blocks: {} as Record<string, Record<string, number>>, issues: [] as Issue[] };
  for (const [name, block] of Object.entries(spec.blocks)) {
    const range = ranges[name];
    if (!range) { /* start-title bị thiếu ⇒ mọi field bắt buộc trong khối bị báo lỗi */ continue; }
    const r = resolveFields(row, range, name, block);
    out.blocks[name] = r.columns;
    out.issues.push(...r.issues);
  }
  return { ok: !out.issues.some((i) => i.blocking), headerRow: h, ...out };
}
```

**Ví dụ thực tế** — một tab chứa ba bảng cạnh nhau:

| col | A | B | C | D | E | F | G | H |
|---|---|---|---|---|---|---|---|---|
| header row | `Domain` | `URL` | `Ngày xong` | *(trống)* | `Giao việc` | `Target` | `Tuần` | `Tổng` |

Anchor `["Domain","Giao việc","Tuần"]`, khối bắt đầu `Domain` / `Giao việc` / `Tuần`:
hàng header sẽ được tìm thấy kể cả khi có 2 hàng chữ to nằm bên trên nó; khối *output* bao từ A–D, khối *assignment* bao E–F, khối *report* bao G–H. Cột D trống thuộc về khối output (các khối bị cắt bởi tiêu đề start của khối kế tiếp, không phải bị cắt bởi cột rỗng).
Nếu ai đó chèn 1 cột trước chữ `Tổng`, code này không cần sửa.

**Quy tắc xuống cấp uyển chuyển (Degrade-gracefully).** `error` + `blocking` ⇒ tab/khối đó bị bỏ qua và dữ liệu cũ giữ nguyên;
`error` non-blocking và `warning` ⇒ đọc bình thường, ghi nhận sự cố. Issue được lưu theo từng lần chạy nên một khối lỗi sẽ hiển thị cho người dùng thay vì làm sập cả đợt sync.

---

## 4.2 Header nhóm theo tháng và header phân hai tầng

Hai cách bài trí ngoài đời thật đánh bại cách dò tiêu đề thông thường:

* **Nhóm tháng:** một nhãn kiểu `Month 10/2026` đứng phía trên một hàng so với header; các cột phụ bên dưới được lấy theo *vị trí tương đối bên trong nhóm* bởi vì tựa cột của nó không đáng tin cậy (một công thức lỗi có thể ra `#REF!`).
* **Header hai tầng:** tầng trên là tên của một người nằm ở một ô **gộp (merged)** (dữ liệu nằm ở ô đầu tiên của cụm gộp); tầng dưới là mã dự án. Nối (Join) bằng *tên*, không bao giờ theo thứ tự cột.

```ts
const REF_ERROR = "#ref!";

/** Khối nhóm tháng: nhãn nằm bên TRÊN header, lấy cột phụ theo vị trí. */
function resolveMonthGroups(headerRow: string[], upperRow: string[], range: {start:number; end:number},
                            expectedSubColumns: string[], labelPattern: RegExp) {
  const labels: { month: string; col: number }[] = [];
  for (let i = range.start; i < range.end; i++) {
    const m = (upperRow[i] ?? "").normalize("NFC").trim().match(labelPattern);   // vd: /^Month (\d{2})\/(\d{4})$/
    if (m) labels.push({ month: `${m[2]}-${m[1]}`, col: i });
  }
  const groups: { month: string; columns: Record<string, number> }[] = [];
  const issues: Issue[] = [];

  labels.forEach(({ month, col }, idx) => {
    const end = labels[idx + 1]?.col ?? range.end;                   // một nhóm chạy tới nhãn tiếp theo
    const columns: Record<string, number> = {};
    let broken = false;
    expectedSubColumns.forEach((expected, k) => {
      const i = col + k;
      const title = norm(headerRow[i]);
      if (i >= end || (title !== norm(expected) && title !== REF_ERROR)) {
        broken = true;                                               // sai tựa ⇒ không đoán
        issues.push(error(`Tháng ${month}: cột ${colLetter(i)} là "${headerRow[i]}", đáng lẽ là "${expected}"`));
        return;
      }
      if (title === REF_ERROR)                                       // dung nhượng: lấy theo VỊ TRÍ, báo cảnh báo
        issues.push(warning(`Tháng ${month}: "#REF!" tại ${colLetter(i)}, lấy là "${expected}" theo vị trí`));
      columns[expected] = i;
    });
    if (!broken) groups.push({ month, columns });                    // một tháng hỏng không che khuất tháng khác
  });
  return { groups, issues };
}

/** Header 2 tầng: tầng trên = ô tên gộp, tầng dưới = site. Nối theo tên. */
function resolveTwoTier(upper: string[], lower: string[], range: {start:number; end:number},
                        knownNames: string[], siteCodes: string[]) {
  const found: Record<string, Record<string, number>> = {};         // tên → site → cột
  let currentName = "";
  for (let i = range.start; i < range.end; i++) {
    if ((upper[i] ?? "").trim()) currentName = upper[i].trim();      // ô gộp: giá trị chỉ có ở ô đầu tiên
    const site = siteCodes.find((c) => norm(c) === norm(lower[i]));
    if (!site) continue;
    const name = knownNames.find((e) => norm(e) === norm(currentName));
    if (name) (found[name] ??= {})[site] = i;                        // tên lạ ⇒ cảnh báo ở nơi khác, không sập
  }
  return found;   // dòng dữ liệu bắt đầu 2 dòng bên dưới header khi có tầng thấp hơn
}
```

Lưu ý về thiết kế: Sheets API **tự cắt bỏ các ô rỗng ở cuối hàng**, cho nên hàng header có thể ngắn hơn hàng trên của nó. Bộ phân giải sẽ tự pad thêm độ dài cho hàng header sao cho bằng hàng dài nhất, nếu không khối cuối sẽ vô tình bị mất cột.

---

## 4.3 Row-Level Security mặc định-từ chối

**Mục tiêu.** Một bảng được tạo vào ngày mai, bởi bất kỳ ai, sẽ tự động bị trình duyệt không nhìn thấy cho đến khi có người mở cửa cho nó. Quyền truy cập bị từ chối phải báo lỗi **lớn** (`42501 permission denied`), chứ không trả về một list rỗng vờ như "không có dữ liệu".

```sql
-- 1. RLS trên mọi bảng (hàng sẽ vô hình nếu không có policy tương ứng)
alter table site          enable row level security;
alter table output        enable row level security;
alter table snapshot      enable row level security;
alter table sync_log      enable row level security;
alter table setting       enable row level security;
-- … tất cả bảng trong schema

-- 2. Tước đi thứ mà hệ thống mặc định cấp cho các role ở client-side
revoke all on all tables    in schema public from anon, authenticated;
revoke all on all sequences in schema public from anon, authenticated;
revoke all on all functions in schema public from anon, authenticated;

-- 3. …và biến điều đó làm mặc định cho tất cả những gì được tạo sau này
alter default privileges in schema public revoke all on tables    from anon, authenticated;
alter default privileges in schema public revoke all on sequences from anon, authenticated;
alter default privileges in schema public revoke all on functions from anon, authenticated;

-- 4. Mở lại với cấu hình tối thiểu, với 2 lớp (privilege VÀ policy)
grant select on snapshot, sync_log to anon;
create policy anon_read_snapshot on snapshot for select to anon using (page <> 'settings');
create policy anon_read_sync_log on sync_log for select to anon using (true);

-- 5. User đã đăng nhập: quyền tương tự public, cộng thêm đọc settings page với các quyền đặc thù
grant select on snapshot, sync_log to authenticated;
create policy auth_read_snapshot on snapshot for select to authenticated
  using (page <> 'settings' or public.is_settings_user());
```

Tại sao phải 2 lớp? Nếu *chỉ có* policy, quên một policy có thể làm hàng biến mất (trả list rỗng) và nếu lỡ GRANT nhầm, thao tác ghi vẫn lọt qua. Với lớp privilege đã bị tước, lệnh `SELECT output` sẽ sập với lỗi `42501` trước cả khi policy chạy. Và quyền Ghi không bao giờ phụ thuộc vào việc ai đó quên tạo policy.

**Bảo vệ việc ghi cho file cấu hình** — policy kiểm tra đồng thời cả hàng cũ và mới, nên không ai đổi tên khóa thành tên khóa được bảo vệ:

```sql
create policy auth_update_setting on setting for update to authenticated
  using      (public.is_settings_user() and public.is_editable_setting_key(key))
  with check (public.is_settings_user() and public.is_editable_setting_key(key));
```

**Mở quyền thời gian thực (Realtime), cũng từ chối mặc định:**

```sql
-- Chỉ bảng nào vô hại như "có cái gì thay đổi" mới được đăng
alter publication supabase_realtime add table public.change_hint;   -- scope, ref, at — không có data
```

---

## 4.4 Các hàm hỗ trợ Role: JWT → role

```sql
-- SECURITY DEFINER: đọc bảng access dù cho người gọi không có quyền truy cập bảng đó.
-- search_path rỗng: mọi tham chiếu phải ghi rõ schema, nên không ai qua mặt bằng cách tạo shadow function/table.
create function public.current_actor()
returns table (email text, role text, duty_roles text[], employee_id bigint)
language sql stable security definer set search_path = ''
as $$
  select a.email, a.role, a.duty_roles, a.employee_id
  from public.app_access a
  where a.email = lower(auth.jwt() ->> 'email')
    -- app_metadata chỉ có thể ghi bởi quyền service role; user_metadata do người dùng tự sửa KHÔNG BAO GIỜ DÙNG.
    -- Provider claim xác thực e-mail đã được hệ thống OAuth chứng minh.
    and coalesce(auth.jwt() -> 'app_metadata' -> 'providers', '[]'::jsonb) @> '"google"'::jsonb
$$;

create function public.can_view() returns boolean
language sql stable security definer set search_path = ''
as $$ select coalesce((select a.role in ('admin','dev') or a.duty_roles && array['staff','po']
                       from public.current_actor() a), false) $$;

-- Postgres cấp quyền EXECUTE cho PUBLIC mặc định — kể cả người gọi nặc danh (anon)
revoke execute on function public.current_actor(), public.can_view() from public;
grant  execute on function public.can_view() to authenticated;      -- current_actor chỉ dùng nội bộ
```

**Bẫy lỗi cần nhớ (được phát hiện bởi test song song TS↔SQL).** User *chỉ có* `duty_roles` thì `role = NULL`.
Trong SQL, `NULL IN ('admin','dev')` sẽ ra `NULL`, chứ không phải `false`, nên mệnh đề `not is_manager` sẽ thành `NULL` và kiểm tra "người gọi phải là người tham gia" đã bị **bỏ qua một cách âm thầm**. Cách sửa: luôn luôn bọc predicate bằng `coalesce(…, false)`.

---

## 4.5 Hàm ghi lũy đẳng, được audit (hợp đồng)

Mọi hàm ghi ở module trực ban đều tuân theo bản hợp đồng năm bước giống hệt nhau, nên tính đúng đắn sẽ phụ thuộc vào *cấu trúc hình thù*, chứ không phải trí nhớ lập trình viên.

```sql
create function public.do_something(p_params …, p_ctx jsonb) returns jsonb
language plpgsql security definer set search_path = '' as $$
declare v_ctx record; v_now timestamptz; v_result jsonb; v_replay jsonb;
begin
  -- 1. Ngữ cảnh: {request_id uuid, api_method, api_path}. Hỏng format ⇒ RAISE (lỗi code, không phải lỗi user)
  select * into v_ctx from public.ctx(p_ctx);

  -- 2. Ủy quyền. Không có quyền ⇒ trả về mã và KHÔNG ghi log (không có danh tính đáng tin nào cả)
  if not public.can_view() then return public.result(false, 'FORBIDDEN_ROLE', 'Không cho phép'); end if;

  -- 3. Phát lại: người này đã từng hoàn thành request_id này ⇒ trả về kết quả cũ
  v_replay := public.replay(v_ctx.request_id);
  if v_replay is not null then return v_replay; end if;

  -- 3b. Khóa các hàng theo trình tự CỐ ĐỊNH (không deadlocks)
  perform 1 from public.shift s where s.date in (…) order by s.date for update;
  -- 3c. Phát lại thêm lần nữa: thằng duplicate phải đợi khóa ở trên giờ sẽ nhìn thấy bản commit đầu tiên
  v_replay := public.replay(v_ctx.request_id);
  if v_replay is not null then return v_replay; end if;

  -- 4. Thời điểm ra quyết định: đọc SAU KHI đã lấy khóa, sau đó mới xác thực. Bị từ chối là GIÁ TRỊ TRẢ VỀ nên
  --    transaction vẫn commit cùng với dòng audit 'rejected'.
  v_now := clock_timestamp();
  if <quy tắc bị lỗi> then
    return public.complete(p_ctx, 'domain.action', 'do_something',
             public.result(false, 'SOME_CODE', 'Tin nhắn con người'), v_params, v_now);
  end if;

  -- … làm công việc …

  -- 5. Đúng duy nhất một dòng audit log; notify realtime khi xong; trả kết quả
  v_result := public.result(true, 'OK', 'Xong', jsonb_build_object('decided_at', v_now));
  return public.complete(p_ctx, 'domain.action', 'do_something', v_result, v_params, v_now,
                         p_scope => 'shift', p_ref => '2026-01-05');
end $$;

-- Bộ helper complete() sẽ tự ép buộc các chuẩn mực:
--   * thành công mà KHÔNG CÓ notify scope ⇒ RAISE (nên test sẽ bắt được hàm nào quên realtime)
--   * insert audit được bọc trong sub-block: lỗi ⇒ RAISE WARNING, không bao giờ đánh hỏng hành động
--   * kết quả quá lớn (trên 4 KB) sẽ tự drop phần data (replay lúc này chỉ trả ok/code/message)
```

---

## 4.6 Quyết định khóa khung giờ + ước lượng giờ server client

### Quyết định (nguồn chân lý duy nhất là ở DB, đánh giá bằng giờ DB)

```ts
type LockConfig = { from: string; to: string; enabled?: boolean };   // vd: {"from":"15:30:00","to":"19:00:00"}
type Decision = { locked: boolean; configError: null | "missing" | "bad_format" | "not_ascending"; nextChangeMs: number | null };

function decideLock(cfg: unknown, msOfDay: number): Decision {
  const parsed = parseConfig(cfg);                              // nghiêm ngặt "HH:MM[:SS[.mmm]]" → ms, from < to
  // KHÓA AN TOÀN (FAIL SAFE): cấu hình hỏng nghĩa là KHÓA. Mất đổi ca một lúc vẫn tốt hơn mở cửa giữa trận live.
  if (!parsed.ok) return { locked: true, configError: parsed.error, nextChangeMs: null };

  const { fromMs, toMs, enabled } = parsed;
  const locked = enabled && msOfDay >= fromMs && msOfDay < toMs;     // nửa mở: [from, to)
  const next = locked ? toMs : msOfDay < fromMs ? fromMs : fromMs + 86_400_000;
  return { locked, configError: null, nextChangeMs: next };
}
```

Bảng ranh giới (có 4 test chạy cùng múi giờ ảo để khẳng định sự phân ly với hệ thống):

| Mốc giờ (local) | Kỳ vọng |
|---|---|
| 15:29:59.999 | cho phép đổi |
| 15:30:00.000 | **chặn** |
| 18:59:59.999 | **chặn** |
| 19:00:00.000 | cho phép |

Hàm tương tự cũng ở trong SQL (`lock_decision(timestamptz)`); nó nhận mốc giờ ở **tham số** chứ không tự đọc đồng hồ, nên hàm swap ở 4.5 có thể đọc đồng hồ *sau khi* lấy khóa và truyền nó vào cho hàm kiểm tra.

Sự tiến hóa của quy tắc: Cửa sổ lúc đầu chặn toàn bộ lệnh đổi ca; sau đó được thu hẹp để chỉ chặn ca của **ngày hôm nay** (các ngày sau vẫn có thể bị đổi). Ca hôm nay khi đến mốc `from` sẽ coi như "đã bắt đầu" và qua mốc `to` sẽ không mở lại nữa.

### Ước tính giờ server từ phía trình duyệt (chỉ để hiển thị)

```ts
// offset = serverTime - midpoint(sentAt, receivedAt); estimate = monotonicNow + offset.
// Chỉ dùng MONOTONIC CLOCK (đồng hồ đơn điệu) để ước lượng, nên đổi giờ OS cũng không làm đổi giờ hiển thị.
// Wall clock (đồng hồ tường) chỉ đóng vai trò làm tín hiệu TÍN HIỆU — để biết khi máy ngủ hoặc đổi giờ hệ thống.
function createServerClock(monotonic = () => performance.now(), wall = () => Date.now(),
                           { maxRttMs = 3000, staleAfterMs = 600_000, suspendGapMs = 120_000 } = {}) {
  let best: { offset: number; at: number; rtt: number } | null = null;
  let suspected = false, lastMono: number | null = null, lastWall: number | null = null;

  const tick = () => {                                           // gọi liên tục mỗi giây
    const m = monotonic(), w = wall();
    if (lastMono !== null && Math.abs((w - lastWall!) - (m - lastMono)) > suspendGapMs) suspected = true;
    lastMono = m; lastWall = w;
    return m;
  };

  return {
    accept({ serverNowMs, sentAt, receivedAt }: { serverNowMs: number; sentAt: number; receivedAt: number }) {
      const rtt = receivedAt - sentAt;
      if (rtt < 0 || rtt > maxRttMs) return false;               // quá nhiễu để tin tưởng
      if (best && rtt > best.rtt && receivedAt - best.at < 60_000) return false;   // giữ lại mẫu xịn hơn nếu nó chưa quá 1 phút
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

Chính sách UI: **không thêm margin an toàn** quanh ranh giới. Nút bấm hoạt động cho tới mili-giây cuối; một yêu cầu đến muộn sẽ bị từ chối rõ ràng bằng chữ do Database trả về. UI tin tưởng DB là thẩm phán duy nhất, làm mờ (dim) ngay khi `stale` bằng true.

---

## 4.7 Mutex đồng bộ thông qua partial unique index

Chỉ báo khóa "đọc rồi ghi" quá rủi ro: hai bản chạy trùng đều nghĩ "mình là duy nhất". Hãy để DB phân định.

```sql
create unique index sync_log_one_running_per_source
  on sync_log (source, site_id) nulls not distinct   -- mãng dùng chung sẽ có site_id NULL
  where status = 'running';
```

```ts
async function acquire(db: Db, plan: PlannedSource[]): Promise<number[] | null> {
  await expireStale(db, 30 /* phút */);                         // rớt mạng thì không được khóa mãi
  // MỘT câu lệnh INSERT nhiều dòng sinh ra các hàng 'running' cho mỗi nguồn.
  const { data, error } = await db.from("sync_log")
    .insert(plan.map((p) => ({ source: p.source, site_id: p.siteId, status: "running" }))).select("id");
  if (error?.code === "23505") return null;                      // unique_violation ⇒ người khác đã cầm ⇒ skip
  if (error) throw error;
  return data.map((r) => r.id);                                  // mỗi hàng sẽ được đóng (success|partial|failed) khi xong
}
```

Bước chốt dọn (Finalization) sử dụng `try/finally`: nếu tiến trình bị dừng giữa đường, các hàng nào chưa đến đích sẽ được đóng lại bằng `failed`.

---

## 4.8 Lưu trữ rồi xóa một cách an toàn fail-closed

```ts
async function archiveAndDelete(store: LogStore, sheet: SheetApi, cutoffIso: string, batchSize = 1000, maxBatches = 200) {
  for (let b = 0; b < maxBatches; b++) {
    const batch = await store.listOlderThan(cutoffIso, batchSize);          // cũ nhất lên đầu
    if (batch.length === 0) break;

    // 1. Chỉ Append hàng CHƯA CÓ TRONG ĐÓ (key là request_id) ⇒ tái chạy vô tư
    const present = await sheet.readRequestIds();
    const fresh = batch.filter((r) => !present.has(r.requestId));
    if (fresh.length) await sheet.append(fresh.map(toSheetRow));            // 7 cột; KHÔNG có parameter, email hay kết quả thô

    // 2. ĐỌC LẠI ngay lập tức trước khi xóa. Tuyệt đối không xài lại bộ read ở bước 1.
    const confirmed = await sheet.readRequestIds();
    const missingIds = batch.filter((r) => !confirmed.has(r.requestId));
    if (missingIds.length) throw new ArchiveError("not_confirmed", `${missingIds.length} rows`);   // KHÔNG xóa gì hết

    // 3. Xóa chính xác các id này — không bao giờ dùng "nhỏ hơn thời gian T". Không có khoảng trống giữa "lưu" và "xóa".
    await store.deleteByIds(batch.map((r) => r.id));
  }
}
```

Ngữ nghĩa Thất bại: bất kỳ exception nào cũng làm dừng trước lần xóa kế tiếp; những hàng nào đã ở trong sheet mà chưa xóa thì lần chạy sau sẽ bỏ qua append và thực hiện lệnh xóa. Công việc trả mã lỗi khác 0 để CI nhắn mail báo động.

---

## 4.9 Thuật toán luân phiên ca dựa trên con trỏ

**Quy tắc.** Một người mỗi ngày làm việc, thứ tự không đổi, vòng lặp. Tính người kế tiếp bằng *vị trí của người đứng ngay trước đó*, không bao giờ là "lấy số ngày chia modulo cho số nhân viên".

Vì sao không dùng modulo? Nghỉ lễ, ai đó nghỉ việc hoặc người mới đến sẽ đánh sập trật tự. Với con trỏ:

* **Nghỉ lễ:** Không trực và **không tốn lượt** — người đáng lẽ trực hôm nay sẽ trực vào ngày làm việc tiếp theo.
* **Người ra đi:** Bỏ qua họ từ ngày họ nghỉ; những người còn lại giữ vững số lượng.
* **Người mới đến:** Chèn vào giữa hai người bằng một con số lẻ ở `position` (vd: `2.5`); lịch tương lai đúng, lịch cũ không bị chạm.

```ts
type Member = { staffId: number; position: number; activeFrom: string; activeTo?: string | null };   // [activeFrom, activeTo)

function generate(anchorDate: string, anchorStaffId: number, members: Member[], skipDates: Set<string>,
                  from: string, to: string) {
  const shifts: { date: string; staffId: number }[] = [];
  let prev: { staffId: number; position: number } | null = null;

  for (const day of workingDays(anchorDate, to)) {                       // T2–T6
    if (skipDates.has(day)) continue;                                    // nghỉ lễ: không ca, không tốn lượt
    const active = distinctByStaff(members.filter((m) => m.activeFrom <= day && (!m.activeTo || day < m.activeTo)))
      .sort((a, b) => a.position - b.position);

    const staffId = prev === null
      ? anchorStaffId                                                    // ngày làm đầu tiên gán cho anchor
      : (active.find((m) => m.position > prev!.position) ?? active[0]).staffId;   // position cao hơn tiếp theo, hoặc quay vòng

    prev = { staffId, position: members.find((m) => m.staffId === staffId)!.position };
    if (day >= from) shifts.push({ date: day, staffId });
  }
  return shifts;
}
```

Ví dụ chạy tay: các thành viên `A(1) B(2) C(3) D(4)`, anchor `A` vào ngày làm số 1, ngày thứ 3 nghỉ lễ, `C` rời đi vào ngày số 5 (`activeTo = day 5`):

| Ngày làm việc | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
|---|---|---|---|---|---|---|---|---|---|
| Người trực | A | B | *nghỉ lễ* | C | D | A | B | D | A |

`C` tới lượt vào lúc nghỉ lễ nên làm lại vào ngày làm tiếp theo. Tới ngày 5 `C` bị bất hoạt, con trỏ đang ở chỗ `C` (vị trí 3) đáp sang `D`; vào ngày 8 vị trí lớn hơn tiếp theo từ `B` tự động bỏ qua `C`.

Lưu trữ bằng hai cột: `staff_id` (hiện tại) và `base_staff_id` (quy tắc gốc). Một cú đổi ca chỉ cập nhật `staff_id`; "đặt lại theo lịch" chỉ việc chép `base_staff_id` đè lên `staff_id`. Hàm tương tự đã được mang vào SQL.

---

## 4.10 `pg_cron` → HTTP dispatch với secret Vault

```sql
-- Dùng một lần, chạy tay từ bên trên cloud (không bỏ vào file migration vì chứa secret)
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
  return req;                                                 -- kết quả vào net._http_response (204 = accepted)
end $$;

revoke all on function public.dispatch_workflow() from public, anon, authenticated;   -- DB owner / cron only

-- Lên lịch lũy đẳng: gỡ lịch cũ, gắn cái mới vào 12:30 UTC.
select cron.unschedule(jobid) from cron.job where jobname = 'daily-sync';
select cron.schedule('daily-sync', '30 12 * * *', $$select public.dispatch_workflow()$$);
```

Lưu ý thiết kế: Secret không bao giờ lộ ra trong khai báo job (nó được đọc từ vault trong lúc chạy); hàm này là nơi duy nhất giữ bí mật; chạy đi chạy lại file này là an toàn; biến input `trigger = 'cron'` được mang đi lưu xuống log đồng bộ để tiện bề phân biệt chạy tay hay chạy máy.

---

## 4.11 Bộ lọc Log (che giấu mật khẩu, giới hạn kích thước)

Các hàng Audit không bao giờ được phép biến thành kho chứa secret. Việc lọc diễn ra **ở phía database**, nên không con đường code nào có thể qua mặt.

```sql
-- Che các chuỗi có vẻ như là token, sau đó cắt ngắn
create function clean_text(p text, p_max int) returns text language sql immutable as $$
  select left(
    regexp_replace(regexp_replace(regexp_replace(regexp_replace(p,
      'eyJ[A-Za-z0-9_-]+(\.[A-Za-z0-9_-]+)*', '[redacted]', 'g'),   -- JWTs
      'bearer\s+\S+',                         '[redacted]', 'gi'),  -- Authorization headers
      'sb-[A-Za-z0-9_-]+',                    '[redacted]', 'g'),   -- vendor key prefixes
      '[A-Za-z0-9+/=_-]{40,}',                '[redacted]', 'g'),   -- các chuỗi base64/hex dài
    p_max) $$;

-- Trình dọn JSON đệ quy: độ sâu tối đa (3), keys (20), mảng (20), chữ (200 chars);
-- key kiểu lạ sẽ tự loại bỏ; giá trị nào rơi vào key nhạy cảm sẽ bị ghi đè.
create function clean_json(v jsonb, depth int) returns jsonb language plpgsql immutable as $$
declare k text; r jsonb := '{}'; n int := 0; i int;
begin
  case jsonb_typeof(v)
    when 'string' then return to_jsonb(clean_text(v #>> '{}', 200));
    when 'object' then
      if depth >= 3 then return '"[truncated]"'; end if;
      for k in select jsonb_object_keys(v) order by 1 loop
        continue when k !~ '^[a-z][a-z0-9_]{0,39}$';                    -- chỉ snake_case sống sót
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

Bảo vệ cấp cuối: Nếu đoạn tham số đã lọc vẫn lớn hơn 4 KB, chúng sẽ bị thay hoàn toàn bằng `{"truncated": true, "bytes": n}`.