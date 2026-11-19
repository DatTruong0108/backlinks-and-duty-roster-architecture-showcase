# 3. Biểu đồ Tuần tự

Có bốn luồng, khó nhất được đặt lên đầu:

| ID | Luồng | Điểm khó khăn |
|---|---|---|
| S1 | Đổi ca trực | Tính lũy đẳng + khóa hàng + **đồng hồ máy chủ trong transaction** + ghi log chính xác một hàng + thông báo realtime, dưới điều kiện nhiều người dùng đồng thời |
| S2 | Lên lịch đồng bộ dữ liệu | Bộ scheduler không đáng tin cậy → kích hoạt dư thừa nhưng tuyệt đối không được chạy hai lần |
| S3 | Làm mới thời gian thực | Giao diện Live mà không bao giờ cần tin tưởng vào dữ liệu từ kênh gửi tới |
| S4 | Lưu trữ log an toàn (Fail-closed) | Chỉ xóa những gì *đã được chứng minh* là lưu ở nơi khác |

---

## S1 — Đổi ca trực (hai chiều, có hiệu lực ngay lập tức)

Tác nhân: nhân viên đã đăng nhập tự đổi ca của *họ* trong một ngày với ca của người khác trong một ngày khác. Ở đây không có bước mời/chấp nhận; quá trình đổi diễn ra nguyên tử (atomic) và ngay lập tức, đồng thời được ghi lại ở bảng lịch sử hiển thị cho người dùng và bảng audit log chỉ dành cho dev.

```mermaid
sequenceDiagram
  autonumber
  actor U as User (trình duyệt)
  participant UI as UI + ước lượng giờ server
  participant PX as Proxy (middleware)
  participant RT as Route handler<br/>withAction
  participant FN as Hàm Postgres<br/>swap_twoway (SECURITY DEFINER)
  participant SH as các hàng ca trực
  participant SW as swap_log (lịch sử)
  participant AL as action_log (audit)
  participant CH as duty_change
  participant RL as Realtime

  Note over UI: Trạng thái nút bấm được quyết định bởi thời gian ƯỚC LƯỢNG<br/>(monotonic clock + độ trễ từ các mẫu thăm dò).<br/>Chỉ dùng để hiển thị. Không bao giờ là nguồn chân lý.

  U->>UI: chọn ngày A và ngày B, bấm "Đổi ca"
  UI->>UI: tạo Idempotency-Key (UUID v4) 1 lần cho 1 thao tác<br/>vô hiệu hóa nút, tạo một in-flight promise (chống double-click)
  UI->>PX: POST /api/swap/twoway {dateA, dateB, expectedA, expectedB, reason}<br/>header Idempotency-Key
  PX->>PX: làm mới session cookie
  PX->>RT: chuyển tiếp (forward)

  rect rgb(240,245,252)
    Note over RT: Kiểm tra vòng ngoài (nếu từ chối = trả về + 1 hàng audit, ngoại trừ bước 1)
    RT->>RT: 1 đã xác thực chưa? nếu chưa → 401 (không ghi DB: vì không có danh tính)
    RT->>RT: 2 Idempotency-Key có phải UUID v4 không → request_id (thiếu → tự tạo)
    RT->>RT: 3 Origin khớp không, 4 Content-type là JSON, 5 body ≤ 32 KB
    RT->>RT: 6 xác thực input tự viết tay → 422 kèm chi tiết trường lỗi
  end

  RT->>FN: RPC swap_twoway(dateA, dateB, expectedA, expectedB, reason, ctx)<br/>chạy bằng JWT của NGƯỜI DÙNG (không bao giờ service_role)<br/>ctx = {request_id, api_method, api_path}

  rect rgb(245,245,245)
    Note over FN,AL: MỘT transaction cơ sở dữ liệu duy nhất
    FN->>FN: xác thực định dạng ctx (sai = lỗi lập trình, raise)
    FN->>FN: kiểm tra vai trò (từ email JWT + provider claim)<br/>nếu không có → trả về FORBIDDEN_ROLE, không ghi log

    FN->>AL: tra cứu phát lại bằng request_id (lần 1)
    alt request_id này đã có kết quả của chính người dùng này
      AL-->>FN: trả về kết quả cũ
      FN-->>RT: kết quả cũ + replayed=true (không có tác dụng phụ)
    else yêu cầu mới
      FN->>SH: SELECT … WHERE date IN (A,B) ORDER BY date FOR UPDATE<br/>(thứ tự cố định → hai yêu cầu đổi ca chéo nhau không thể deadlock)
      Note right of SH: Một lệnh đổi ca đồng thời chạm vào A hoặc B<br/>sẽ bị block tại đây cho đến khi commit/rollback
      FN->>AL: tra cứu phát lại (lần 2)<br/>một lệnh trùng lặp phải chờ lock lúc nãy sẽ nhìn thấy commit của lệnh đầu tiên
      FN->>SW: kiểm tra mồ côi (orphan): hàng lịch sử có mà hàng audit bị mất?<br/>→ tái tạo lại kết quả ban đầu từ hàng lịch sử

      FN->>FN: now = clock_timestamp()<br/>ĐỌC SAU KHI ĐÃ CẦM KHÓA<br/>(thời điểm ra quyết định được lưu cùng với audit log)

      FN->>FN: thang xác thực (xem bảng bên dưới)
      alt nếu bất kỳ quy tắc nào từ chối
        FN->>AL: INSERT hàng audit kết quả=rejected + mã lỗi
        Note over FN,AL: từ chối là GIÁ TRỊ TRẢ VỀ, không phải exception,<br/>do đó transaction VẪN COMMIT kèm hàng audit
        FN-->>RT: {ok:false, code, message}
      else tất cả đều vượt qua
        FN->>SH: UPDATE ca A ← nhân viên B, ca B ← nhân viên A<br/>(updated_at = now, last_request_id)
        FN->>SW: INSERT hàng lịch sử {changes[], reason, request_id}<br/>các tên được LƯU SNAPSHOT lúc đổi
        FN->>AL: INSERT hàng audit kết quả=success (khối phụ: lỗi → chỉ warning)
        FN->>CH: INSERT (scope='shift', ref='A,B')
        FN-->>RT: {ok:true, data:{changes, decided_at}}
      end
    end
  end

  RT->>RT: ánh xạ mã kết quả → HTTP status (409 SWAP_LOCKED / SHIFT_CHANGED, 422, 403, 200…)
  RT->>AL: log_finish(request_id, http_status, duration_ms) — chạy best-effort, thử lại một lần
  RT-->>UI: gói gọn JSON
  UI->>UI: key được giữ lại cho các lỗi retryable, hủy bỏ sau khi có kết quả cuối cùng
  CH-->>RL: event thay đổi (chỉ là gợi ý)
  RL-->>U: màn hình của những người dùng khác tải lại (xem S3)
```

### Thang xác thực (Thứ tự rất quan trọng)

Thứ tự được cố định một cách có chủ đích để lỗi *có ý nghĩa nhất* sẽ được trả về trước và các kiểm tra ít tốn kém sẽ nằm trước các kiểm tra trạng thái.

| # | Kiểm tra (Check) | Mã kết quả (Result code) |
|---|---|---|
| 1 | Người gọi có role trực ban | `FORBIDDEN_ROLE` (không có log) |
| 2 | Phát lại: cùng `request_id` đã được hoàn tất bởi người này | kết quả đã lưu, `replayed: true` |
| 3 | *(khóa 2 hàng, sau đó đọc đồng hồ DB)* | — |
| 4 | Cấu hình khung giờ khóa có thể đọc và hợp lệ (cấu hình hỏng ⇒ **khóa an toàn: locked**) | `SWAP_LOCKED` |
| 5 | Đầu vào: cả hai ngày đều có mặt, đúng format, có đủ `expected*`, lý do ≤ 500 ký tự | `VALIDATION_ERROR` |
| 6 | Hai ngày phải khác nhau | `SWAP_SAME_DAY` |
| 7 | Cả hai phải là ngày trong tuần | `SWAP_WEEKEND` |
| 8 | Không ngày nào nằm trong quá khứ | `SWAP_PAST_DATE` |
| 9 | Nếu ngày nào là **hôm nay**: nếu đang trong khung giờ khóa → `SWAP_LOCKED`; qua khỏi giờ khóa → `SWAP_DAY_ENDED` | `SWAP_LOCKED` / `SWAP_DAY_ENDED` |
| 10 | Cả hai ngày đều phải có người được gán trực | `SWAP_NO_STAFF` |
| 11 | Hai người phải khác nhau | `SWAP_SAME_STAFF` |
| 12 | Người gọi là 1 trong 2 người này (quản lý có thể đổi thay) | `SWAP_NOT_PARTICIPANT` |
| 13 | `expected*` phải khớp với người đang trực hiện tại (màn hình có bị cũ không?) | `SHIFT_CHANGED` |

### Các thuộc tính và cách đạt được chúng

| Thuộc tính | Cơ chế |
|---|---|
| **Hiệu lực chính xác một lần** dưới click đúp / retry / mạng chập chờn | Client gửi một `Idempotency-Key` trên mỗi intent. DB: dùng `UNIQUE (request_id)` cho cả bảng history và audit + phát lại (replay) trước *và sau* khi cầm khóa. |
| **Không có race condition kiểm tra/sử dụng** | Khóa trước, xem giờ thứ hai, xác thực thứ ba. Một yêu cầu phải chờ khóa sẽ được phán xét ở chính giây phút nó thực sự chạy. |
| **Quyền lực của đồng hồ** | `clock_timestamp()` trong DB. Đồng hồ của Client/app-server không bao giờ được dùng để ra quyết định. |
| **Đồng hồ Client không thể làm giả hoặc lệch** | Bộ ước lượng UI chỉ dùng monotonic clock + độ lệch từ các mẫu máy chủ (S3, đoạn mã 4.6); nếu nghi ngờ client máy đang ngủ ⇒ nút mờ đi cho đến khi đồng bộ lại. |
| **Không Deadlock** | Các hàng được khóa bằng `ORDER BY date`. |
| **Tính chính xác của ranh giới** | Khóa khi và chỉ khi `từ ≤ thời gian trong ngày < đến`. Đã test tại 15:29:59.999 (cho phép), 15:30:00.000 (chặn), 18:59:59.999 (chặn), 19:00:00.000 (cho phép) *và* với tiến trình test chạy ở một múi giờ khác. |
| **Tính trọn vẹn của Audit** | Một hàng audit cho mỗi request (`UNIQUE`), ghi bằng hàm trong cùng một giao dịch; một job đối chiếu sẽ đi tìm các hàng nghiệp vụ nếu nó thiếu hàng audit tương ứng. |
| **Việc ghi Log không thể làm hỏng hành động** | Thao tác insert audit nằm trong một sub-block (savepoint); nếu lỗi chỉ báo cảnh báo. |
| **Không thể giả mạo ủy quyền** | Danh tính đọc từ JWT ngay trong hàm. Không tồn tại tham số "actor" nào cả. |
| **Cấu hình an toàn (Fail-safe)** | Thiếu / hỏng cấu hình khóa hoặc `from ≥ to` sẽ bị coi là *khóa*. Mất chức năng đổi ca tạm thời vẫn tốt hơn là bị hở quyền trong lúc đang phát sóng trực tiếp. |

### Kịch bản đồng thời (Concurrency scenario)

```mermaid
sequenceDiagram
  autonumber
  participant A as Yêu cầu A (đổi ngày 1 ↔ 2)
  participant B as Yêu cầu B (đổi ngày 2 ↔ 3)
  participant DB as hàng ca trực

  A->>DB: khóa ngày 1,2 (theo thứ tự) → lấy được
  B->>DB: khóa ngày 2,3 (theo thứ tự) → CHỜ ở ngày 2
  A->>DB: đọc đồng hồ, xác thực, update, insert lịch sử + audit
  A->>DB: COMMIT (giải phóng khóa)
  DB-->>B: lấy được khóa, đọc lại các hàng
  B->>B: người trực dự kiến của ngày 2 không còn khớp nữa
  B->>DB: hàng audit bị từ chối SHIFT_CHANGED, COMMIT
  Note over B: người dùng nhìn thấy "Lịch vừa thay đổi, hãy tải lại và chọn tiếp"
```

### Chỗ của `pg_cron` trong hệ thống này

`pg_cron` **không** tham gia vào transaction đổi ca. Nó làm nhiệm vụ kích hoạt pipeline *đồng bộ* (S2). Khung giờ khóa đổi ca có ảnh hưởng gián tiếp đến việc lập lịch: hành động "chạy bảo trì" thủ công sẽ bị từ chối trong khung giờ khóa đổi ca, và job bảo trì tự động được đặt lịch bên ngoài khung giờ bận rộn.

---

## S2 — Lên lịch đồng bộ dữ liệu (kích hoạt dư thừa, không bao giờ chạy đúp)

```mermaid
sequenceDiagram
  autonumber
  participant PC as pg_cron (DB)
  participant FN as dispatch_sync_workflow()<br/>SECURITY DEFINER
  participant VT as Vault
  participant PN as pg_net
  participant GH as GitHub API / Actions
  participant JOB as job đồng bộ (runner)
  participant DB as Postgres
  participant GS as Google Sheets API

  PC->>FN: vào đúng mốc thời gian đã định (cron expression, UTC)
  FN->>VT: đọc bí mật giải mã "dispatch token"
  FN->>PN: http_post(workflow_dispatch, inputs.trigger='cron')
  PN->>GH: POST …/dispatches (timeout 10 giây)
  GH-->>PN: 204 No Content (hàng response được giữ lại để kiểm tra)
  GH->>JOB: bắt đầu chạy (CI concurrency group: mỗi lần chỉ chạy 1)

  JOB->>DB: dọn hàng 'running' quá hạn (cũ hơn 30 phút → đánh thành failed)
  JOB->>DB: còn hàng 'running' nào không?
  alt đang có lần chạy khác
    JOB-->>GH: thoát — "skipped: đang có bản sync chạy"
  else trống
    JOB->>DB: MỘT lệnh INSERT nhiều hàng chứa 'running' cho tất cả source
    Note right of DB: partial unique index (source, site) WHERE status='running'<br/>→ hai bản chạy sát nhau: lệnh INSERT đến sau dính lỗi 23505 → skip
    loop duyệt từng source theo thứ tự (domain pool trước, sau đó project: sheet, search-console, backlink-tool)
      JOB->>DB: đánh dấu started
      JOB->>GS: đọc các tab (chỉ đọc)
      JOB->>JOB: phân giải header → parse → xác thực → gom vấn đề
      JOB->>DB: upsert các hàng cơ sở
      JOB->>DB: đóng hàng sync_log (success / partial / failed + số liệu + vấn đề)
      Note over JOB: một source lỗi không bao giờ làm hỏng các source khác
    end
    JOB->>DB: tải tập dữ liệu (bao gồm cả data từ các source bị lỗi lúc nãy)
    JOB->>JOB: core tính toán mọi trang (hàm thuần túy)
    JOB->>DB: ghi snapshot (+ dọn dẹp)
    JOB-->>GH: exit code phản ánh lỗi
  end
```

### Đường dẫn dự phòng (scheduler gốc được biết là có lúc chậm hàng giờ)

```mermaid
sequenceDiagram
  autonumber
  participant GH as Lịch trình gốc GitHub (chậm)
  participant G as bước bảo vệ
  participant DB as Postgres (REST)
  participant JOB as job đồng bộ

  GH->>G: kích hoạt (có thể trễ hàng giờ)
  G->>DB: có sync_log nào status in (success, partial) trong vòng 12h?
  alt có
    G-->>GH: bỏ qua các bước sau ("cái chính chạy rồi")
  else không, hoặc bản thân câu query bị lỗi
    Note over G: query lỗi KHÔNG bỏ qua:<br/>chạy thừa một lần vẫn tốt hơn mất hẳn một ngày
    G->>JOB: chạy đồng bộ
  end
```

Hai lớp chống chồng chéo độc lập: nhóm **`concurrency` của CI** (chi phí rẻ, nằm ở orchestrator) và **DB mutex** (mang tính thẩm quyền tuyệt đối, vẫn chạy đúng dù đổi orchestrator). Hết hạn do để quá lâu (30 phút) giúp ngăn trường hợp hệ thống crashed giữ khóa mãi mãi.

---

## S3 — Làm mới Thời gian thực (chỉ gợi ý, không phải data)

```mermaid
sequenceDiagram
  autonumber
  participant B as Browser
  participant API as Máy chủ Next.js
  participant AUTH as Auth (phân tích token)
  participant RT as Kênh Realtime
  participant DB as Postgres

  B->>API: GET /api/realtime-token (dùng session cookie, httpOnly)
  API->>AUTH: getUser() — kiểm tra chữ ký và đảm bảo tài khoản vẫn tồn tại
  AUTH-->>API: ok
  API-->>B: access token thời hạn ngắn (còn ≥30s, tối đa 1h) + url + anon key<br/>(refresh token không bao giờ rời khỏi server)
  B->>RT: đăng ký nghe duty_change

  Note over DB: mọi hàm ghi thành công sẽ kết thúc bằng<br/>duty_notify(scope, ref) → INSERT duty_change

  DB-->>RT: sự kiện thay đổi hàng {scope, ref, at}
  RT-->>B: gợi ý (hint)
  B->>B: gộp các gợi ý nhận được trong 200ms thành MỘT lần gọi refetch
  B->>API: gọi refetch trang qua đường server thông thường
  API->>DB: sử dụng session người dùng → chạy lại RLS + role checks
  DB-->>API: các hàng dữ liệu
  API-->>B: data
  B->>B: so sánh fingerprint của trạng thái — chỉ render lại nếu có thay đổi
```

Các chế độ suy giảm (Máy trạng thái hiển thị trên giao diện dưới dạng thẻ kết nối):

```mermaid
stateDiagram-v2
  [*] --> connecting
  connecting --> live: subscribed
  connecting --> reconnecting: mất kênh hoặc lấy token thất bại
  live --> reconnecting: mất kênh
  reconnecting --> live: subscribed (đồng bộ lại 1 lần)
  reconnecting --> polling: vẫn down sau 3 giây chờ
  polling --> live: subscribed
  polling --> offline: poll thất bại liên tục
  offline --> polling: một lần poll thành công
  offline --> reconnecting: trình duyệt báo online
  live --> offline: trình duyệt báo offline
  polling --> offline: trình duyệt báo offline
  live --> live: đối chiếu mỗi 60 giây (1 GET + so sánh fingerprint)
```

Các trigger để đồng bộ lại (resync): tab đang ẩn biến thành visible, trình duyệt online lại, một bước check ổn định 5 giây sau mỗi lần subscribe (máy chủ realtime có thể chưa sẵn sàng phát tín hiệu lúc đó). Khoảng thời gian Polling là 5 giây.

Cách parse gợi ý mang tính phòng thủ: chỉ nhận những `scope` có trong danh sách trắng (whitelist) và một `ref` ngắn gọn; bất kỳ điều gì bất thường (payload trống, sai kiểu dữ liệu, sai scope) sẽ lập tức chuyển thành "refetch everything" (đồng bộ mọi thứ) — khi nghi ngờ, tải lại luôn an toàn hơn bỏ lỡ một thay đổi.

Lý do thiết kế này an toàn: Kênh truyền tải chỉ mang `{scope, ref, timestamp}`. Một token bị lộ hoặc event bị phát lại không tiết lộ nội dung gì cả, và nếu lỡ mất event thì đã có cơ chế đối chiếu/poll — tính đúng đắn của dữ liệu không bao giờ nằm trong tay kênh truyền này.

---

## S4 — Lưu trữ log an toàn (Fail-closed)

```mermaid
sequenceDiagram
  autonumber
  participant J as job bảo trì
  participant DB as Postgres (service role)
  participant SH as Sheet Lưu trữ (quyền ghi)

  J->>DB: batch = lấy tối đa N hàng audit cũ nhất quá hạn (vd: 90 ngày)
  loop chạy cho tới khi hết hàng hoặc đạt tới giới hạn batch
    J->>SH: thêm (append) các hàng có request_id CHƯA có trong sheet (chạy lại vô tư, không trùng lặp)
    J->>SH: ĐỌC LẠI sheet ngay sau đó (không tái sử dụng kết quả đọc cũ)
    J->>J: mọi request_id của đợt này đã xuất hiện trong sheet chưa?
    alt thiếu bất kỳ id nào
      J-->>J: DỪNG — không xóa gì hết, báo lỗi job rõ ràng
    else đã xác nhận tất cả
      J->>DB: DELETE WHERE id IN (chính xác các id của batch này)
      Note over J,DB: xóa theo id, không bao giờ xóa theo timestamp:<br/>không có độ lệch giữa "lưu" và "xóa"
    end
  end
  J->>DB: dọn các hint realtime quá 1 ngày
  J->>DB: đối chiếu: các hàng nghiệp vụ đang thiếu audit (trong vòng 2 ngày)
  J->>DB: đo kích thước (bảng log, cả DB) so với ngưỡng
  J-->>J: thoát (exit non-zero) nếu có khoảng trống, vượt ngưỡng, lưu trữ lỗi hoặc bất kỳ bước nào bỏ qua
```

An toàn sau sự cố (Crash-safety): nếu việc xóa bị hỏng sau khi append thành công, ở lần chạy tới job sẽ nhận ra hàng đó đã được lưu trữ, nó sẽ bỏ qua bước append và chỉ tiến hành xóa. Các cột được lưu trữ loại bỏ hoàn toàn param, email, và dữ liệu kết quả thô.