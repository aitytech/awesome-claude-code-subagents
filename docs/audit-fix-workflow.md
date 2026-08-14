---
paths:
  - "**/*"
---

# Audit-Fix Workflow (BẮT BUỘC)

> Quy trình bắt buộc khi sửa một issue đã được audit / báo cáo bởi khách hàng,
> bên audit, reviewer, hoặc AI. Rút ra từ 7 PR thực tế (#742–#748), 2026-08-10.
>
> Last updated: 2026-08-10

---

## Khi nào rule này áp dụng

**BẮT BUỘC** khi có bất kỳ điều nào dưới đây:

- Sửa một issue có số hiệu (GitHub issue, ticket, mục trong bảng audit)
- Sửa theo một báo cáo audit / security review / code review từ bên ngoài
- Sửa theo phản hồi của khách hàng có kèm bằng chứng (screenshot, log, dòng code)
- Sửa một bug đã được mô tả trước bởi người/AI khác (không phải bug bạn tự phát hiện lúc đang code)

**KHÔNG áp dụng** cho: viết feature mới, refactor tự phát, sửa lỗi typo/format,
hoặc bug bạn vừa tự tạo ra trong cùng phiên làm việc.

Nếu không chắc → coi như áp dụng.

---

## Nguyên tắc cốt lõi

> Đừng tin báo cáo một cách mù quáng, và đừng tin fix của chính mình một cách mù quáng.
> Xác minh hai lần: một lần **trước** khi sửa (đọc code thật, không suy đoán),
> một lần **sau** khi sửa (revert lại fix, xem test có fail như mong đợi không).

Thứ được đánh giá không chỉ là "đã sửa chưa" — mà là **chất lượng của quy trình sửa**.
Mỗi khẳng định phải có bằng chứng đi kèm.

---

## Ngôn ngữ khi viết nội dung lên GitHub (BẮT BUỘC)

Áp dụng cho **mọi nội dung đăng lên GitHub**: commit message, PR title/body/comment,
issue comment, code review comment. Không có ngoại lệ.

- **Chỉ dùng tiếng Anh hoặc tiếng Nhật.** Không dùng tiếng Việt, dù đang trao đổi với
  người dùng bằng tiếng Việt trong phiên làm việc.
- Chọn ngôn ngữ theo ngữ cảnh: reply một issue viết bằng tiếng Nhật → reply bằng tiếng
  Nhật; commit message/PR body/code comment → tiếng Anh (khớp quy ước code comment
  hiện có trong repo).
- Tài liệu nội bộ dưới `.claude/rules/`, `.claude/skills/`, các báo cáo/plan riêng của
  team (không đăng công khai lên GitHub issue/PR) **không thuộc phạm vi rule này** —
  các file đó vẫn có thể viết tiếng Việt như từ trước tới nay.
- Trước khi chạy `gh issue comment` / `gh pr comment` / `gh pr create` / `git commit`,
  tự kiểm tra lại nội dung bằng tiếng Việt hay không. Nếu lỡ đăng nhầm tiếng Việt lên
  GitHub, sửa lại ngay bằng `gh api ... -X PATCH` (comment) hoặc `gh pr edit`/`gh issue
  edit` (title/body) — không để nội dung tiếng Việt tồn tại công khai trên GitHub.

## Cập nhật trạng thái liên tục (BẮT BUỘC)

Không phải chỉ cập nhật project board/issue status ở bước cuối (09) — cập nhật **ngay
tại thời điểm trạng thái thực tế đổi**, trong suốt quy trình:

- Bắt đầu làm issue → assignee + status chuyển ngay (ví dụ Triage/Todo → In Progress).
- PR đã mở, đang chờ review/self-review → status → In Review.
- PR đã merge → status → Done, **ngay sau khi merge xong**, không để cuối phiên mới
  cập nhật hàng loạt.
- Nếu một phiên làm nhiều issue cùng lúc, mỗi issue cập nhật độc lập theo tiến độ thật
  của chính nó — không đợi tất cả xong rồi cập nhật đồng loạt.

---

## Quy trình 11 bước

### 00 — Xác nhận baseline sạch

Trước khi tạo branch, chạy toàn bộ test suite của package sẽ đụng tới. **Ghi lại con số.**

```bash
pnpm --filter @siteforce/<package> test
# > 1111 passed, 7 skipped   — ghi lại con số NÀY trước khi sửa gì cả
```

- Nếu baseline đã đỏ sẵn → ghi rõ trong PR, KHÔNG quy lỗi đó cho fix của mình.
- KHÔNG được report "tất cả xanh" khi có lỗi không liên quan tồn tại từ trước.

### 01 — Một issue = một branch độc lập

Tạo branch từ **branch tích hợp** (không phải `develop`, không phải branch của issue khác).

```bash
git branch --show-current              # kiểm tra branch hiện tại TRƯỚC
git checkout <branch-tich-hop>         # ví dụ: fix/20260810-w0-w1
git checkout -b fix/<issue-number>-<mô-tả-ngắn>
```

**Bẫy hay gặp:** quên switch branch trước khi bắt đầu issue tiếp theo → sửa nhầm lên
branch của issue trước, làm bẩn PR đã tạo. Luôn `git status --porcelain` +
`git branch --show-current` trước khi bắt đầu.

### 02 — Đọc code thật trước khi tin báo cáo

Báo cáo có thể sai: sai dòng, sai cơ chế, hoặc đúng vấn đề nhưng sai nguyên nhân.

Trước khi viết fix:
1. Mở **đúng file/dòng** được trích dẫn
2. Đọc code hiện tại
3. Tự trả lời: report có đúng không? Nếu đúng, **root cause thật sự** là gì?

> **Ví dụ thật — issue "launcher che nút của host":**
> Báo cáo đề xuất thêm collision-detection quét DOM host. Đọc code thật thấy:
> launcher hardcode inset 24px, không có cách nào chỉnh. → Fix đúng là thêm offset
> cấu hình được (deterministic), **không phải** collision-detection (runtime,
> không test được, có thể đặt launcher ngay dưới con trỏ user).

**Báo cáo sai không chỉ ở phần mô tả bug — code fix ĐỀ XUẤT trong báo cáo cũng có thể
sai.** Đừng copy nguyên si snippet gợi ý rồi coi là xong; verify nó thật sự làm đúng
điều báo cáo muốn, đặc biệt với logic phụ thuộc hành vi DB/ngôn ngữ cụ thể (NULL
ordering, timezone, làm tròn số, so sánh chuỗi theo locale...) — những thứ code đọc qua
tưởng đúng nhưng chạy sai vì mặc định của hệ thống khác với trực giác.

> **Ví dụ thật — #883 (erasure cascade block do row fail vĩnh viễn):**
> Báo cáo tự đề xuất `orderBy: [{ errorMessage: 'asc' }, { cutoverAt: 'asc' }]` để đẩy
> row đã fail (errorMessage khác null) xuống cuối. Verify bằng `psql` thật trên
> PostgreSQL: `ORDER BY ... ASC` mặc định là **NULLS LAST**, nghĩa là code y nguyên
> theo báo cáo sẽ đẩy row **CHƯA fail** (NULL) xuống cuối — ngược hoàn toàn với ý định,
> khiến bug nặng hơn chứ không nhẹ đi. Sửa đúng bằng
> `{ errorMessage: { sort: 'asc', nulls: 'first' } }` (tường minh), verify lại bằng
> Prisma thật + Postgres thật (không chỉ mock) trước khi tin.

### 03 — Sửa đúng nguyên nhân, không phải đường tắt

- Theo convention có sẵn trong codebase — tìm một chỗ làm đúng gần đó và làm giống vậy.
- Khi giải pháp "dễ nhất" ≠ "đúng nhất": chọn **đúng**, và ghi rõ trong code/PR vì sao
  đường tắt không được chọn.
- Nếu bản nghiên cứu/kế hoạch gốc sai → tự sửa lại và **ghi chú vì sao khác bản gốc**.

> **Ví dụ thật:** kế hoạch gốc đề xuất regex `/[ -]/g` để loại control-char trong log.
> Kiểm tra lại thấy đây là range "space đến hyphen" (` `, `!`, `"`, `#`…) — **không phải**
> control char. → Sửa thành `/[\x00-\x1f\x7f]/g`, ghi chú lại trong code + PR.

### 04 — Viết test: nhiều case, cả case biên

Không chỉ "happy path". Bắt buộc phủ:

- Giá trị rỗng / `null` / `undefined` / `NaN`
- Giá trị vượt giới hạn (âm, quá lớn, chuỗi rỗng)
- **Thứ tự gọi hàm** (ví dụ: sanitize phải chạy *trước* khi log, không phải sau)
- Case **không đổi** — hành vi cũ vẫn giữ nguyên cho phần không liên quan đến fix

### 04b — 横展開 (lateral-spread check, BẮT BUỘC)

Report chỉ trích đúng những chỗ tác giả report tình cờ tìm thấy — không phải danh sách
đầy đủ mọi nơi cùng một loại bug tồn tại. **Sửa đúng những file issue liệt kê là chưa đủ.**

Trước khi coi fix đã xong:

1. Xác định **hình dạng bug** (invariant bị vi phạm), không phải vị trí cụ thể. Ví dụ #994:
   không phải "5 chỗ trong 2 file thiếu filter" mà là "mọi enumeration cross-tenant lấy
   `Tenant` cho một cron/batch job tự động hành động đều phải loại trừ tenant đã
   yêu cầu/đã hoàn tất xoá".
2. Grep/Glob toàn repo cho **cùng pattern cấu trúc** (không chỉ cùng tên biến/hàm) — ví
   dụ: mọi file gọi `withSystemRole`/`withBypass`/`SystemPrismaService` rồi enumerate
   một bảng nhạy cảm, không chỉ file được issue trích dẫn.
3. Với mỗi candidate tìm được, đọc code thật (áp dụng lại bước 02) để xác nhận nó thực
   sự cùng invariant bị vi phạm — đừng sửa mù theo tên hàm giống nhau.
4. Sửa **tất cả** các chỗ xác nhận được trong cùng branch/PR của issue gốc, viết test
   + revert-check riêng cho từng chỗ (bước 04 + 05), và **ghi rõ trong commit/PR** rằng
   đây là lateral fix, không nằm trong phạm vi issue gốc, tìm được bằng cách nào.
5. Nếu quét ra được candidate nhưng **cố tình không sửa** (khác risk shape, ngoài
   scope hợp lý) — ghi rõ lý do trong PR, không im lặng bỏ qua (đúng tinh thần "Không
   thương lượng #3" bên dưới).

> **Ví dụ thật — #994:** issue liệt kê 5 chỗ (4 trong `trial-expiry-tick.ts`, 1 trong
> `plan-expiry-tick.ts`) thiếu filter `deletedAt: null`. Lateral-spread check tìm thêm
> `cost-collector-tick.ts` — cùng invariant, cùng hậu quả (ghi audit log mới cho tenant
> đã yêu cầu xoá), không nằm trong issue gốc. Sửa trong cùng PR, ghi rõ tìm được bằng
> lateral-spread check.

### 05 — Revert-check (BẮT BUỘC, không phải tùy chọn)

Test xanh **không chứng minh được gì** nếu test đó xanh ngay cả khi không có fix.

```bash
cp src/file.ts /tmp/file.ts.bak
git checkout <base-branch> -- src/file.ts   # hoàn nguyên về bản gốc
pnpm test -- file.test.ts                   # PHẢI thấy FAIL
cp /tmp/file.ts.bak src/file.ts             # khôi phục fix
pnpm test -- file.test.ts                   # xác nhận xanh lại
```

| Không revert-check | Có revert-check |
|---|---|
| "8 test mới, tất cả pass" — có thể test không kiểm tra gì cả, hoặc kiểm tra sai thứ. | "8 test mới, 7/8 fail khi hoàn nguyên fix" — chứng minh được test thật sự phụ thuộc vào fix. |

Ghi kết quả revert-check vào PR body. Nếu một test **không** fail khi revert → giải thích
vì sao (thường là test regression cố ý), hoặc test đó vô giá trị và phải viết lại.

### 06 — Typecheck mọi package đã đụng tới

Không chỉ package chứa fix — cả package **phụ thuộc** vào nó.

```bash
pnpm --filter @siteforce/shared build      # nếu sửa type dùng chung
pnpm --filter <mọi package import nó> typecheck
```

### 07 — Bug hiển thị trên UI/trình duyệt: đừng chỉ tin unit test

Nếu bug là về vị trí, màu sắc, hay hành vi trực quan:

1. Build package **thật**
2. Chạy trong trình duyệt **thật** (headless hoặc thật)
3. Thao tác qua **UI thật** (không gọi hàm nội bộ)
4. Đo bằng **số đo thực tế** — `getBoundingClientRect()`, computed color, overlap —
   không suy luận từ đọc CSS

> **Ví dụ thật:** build widget-sdk → serve local → `window.Siteforce.boot()` →
> bấm đúng nút "Monochrome" trên UI thật → scroll 1500px → đo
> `getBoundingClientRect()` của header host → `top=0` (đúng).

### 08 — Commit & PR: giải thích, không chỉ mô tả

Commit message + PR body phải trả lời được:

- [ ] Nguyên nhân gốc là gì?
- [ ] Vì sao chọn fix này chứ không phải cách khác (kể cả cách "dễ hơn")?
- [ ] Phần nào cố ý để ngoài scope, và vì sao?
- [ ] Kết quả test (con số thật, dán ra)
- [ ] Kết quả revert-check (bao nhiêu test fail khi revert)

PR nhắm vào **branch tích hợp**, KHÔNG phải `develop` — để có một bước review gộp
trước khi vào nhánh chính. Ngôn ngữ: tiếng Anh (xem "Ngôn ngữ khi viết nội dung lên
GitHub" ở trên) — không dùng tiếng Việt trong commit message hay PR body.

### 09 — Trả lời issue gốc bằng ngôn ngữ của người đọc

- Xác nhận điểm nào trong báo cáo **đúng**
- Giải thích vì sao bug lọt qua (nguyên nhân gốc, **không đổ lỗi**)
- Nếu báo cáo có điểm **sai/không chính xác** — nói rõ, kèm bằng chứng code.
  KHÔNG lặng lẽ sửa khác đi rồi im lặng.
- "Ngôn ngữ của người đọc" = tiếng Anh hoặc tiếng Nhật theo ngôn ngữ report gốc —
  KHÔNG BAO GIỜ là tiếng Việt, xem quy tắc ngôn ngữ ở đầu file.
- Cập nhật board/tracker ngay khi trạng thái đổi (xem "Cập nhật trạng thái liên tục"),
  không đợi đến bước này mới cập nhật.

---

## Review toàn batch trước khi merge lên nhánh chính (BẮT BUỘC khi nhiều issue đã gộp)

Áp dụng khi nhiều issue riêng lẻ đã được fix + merge vào một nhánh tích hợp
(`fix/YYYYMMDD-...`), trước khi nhánh đó được merge tiếp lên `develop`/nhánh chính.
Review từng issue riêng lẻ không thấy được tương tác GIỮA các fix — bước này bắt buộc
nhìn toàn cục.

- **Review BẮT BUỘC có 2 lớp, không được làm 1 lớp rồi coi là đủ:**
  1. **Review riêng cho TỪNG issue** — chạy `/review` (hoặc phân tích tương đương) trên
     đúng diff của từng PR/issue riêng lẻ (`gh pr diff <PR#>`), không phải diff đã gộp
     chung với các issue khác. Review gộp chung dễ làm loãng/ẩn đi vấn đề riêng của một
     fix cụ thể — ví dụ một agent review batch có thể bỏ sót chi tiết mà một agent chỉ
     tập trung đúng 1 PR sẽ bắt được. Đăng kết quả làm **comment trên đúng PR của issue
     đó** (không chỉ gộp vào 1 báo cáo chung), để member sau này đọc lại đúng ngữ cảnh.
  2. **Review toàn batch** — chạy `/review` trên **toàn bộ diff của nhánh tích hợp so
     với base thật** (`develop`), để bắt tương tác GIỮA các fix mà review riêng từng cái
     không thấy được.
  Cả hai lớp đều bắt buộc khi có ≥2 issue đã gộp vào 1 nhánh tích hợp — làm 1 trong 2 rồi
  dừng là chưa đủ.
- Ở cả hai lớp, dispatch tối thiểu 2 agent độc lập, fresh context (không có bias từ quá
  trình tự fix), theo 2 góc nhìn khác nhau — ví dụ security + test-coverage — chạy song
  song. Agent tự review lại code mình vừa viết có xu hướng đánh giá cao chính nó; góc
  nhìn độc lập bắt được cái đó (ví dụ đã xảy ra: agent test-coverage tìm ra pattern lỗi ở
  dưới mà tự mình review lại code cũ đã bỏ sót).
- **Chạy lại test cho MỌI package đã đụng tới trong toàn batch**, không chỉ package của
  issue đang làm — một fix ở issue A có thể ảnh hưởng gián tiếp test của issue B nếu
  chung file/module. Chạy **toàn bộ** monorepo (`pnpm test`), không chỉ các package đã
  đổi — bỏ qua package "chắc không liên quan" là đúng lúc bug ẩn lọt qua.
- **`pnpm test` chạy full monorepo song song mặc định (~38 package) trên máy dùng chung
  có thể tự crash một phần** (`EPIPE`/"Worker exited unexpectedly", kể cả ở package
  không hề nằm trong diff — đã thấy thật: `widget-sdk` build fail dù không đụng tới) —
  đây là false failure do quá tải máy, không phải bug. Đừng vì thấy lỗi này mà bỏ qua
  chạy full suite — hạ concurrency (`npx turbo run test --concurrency=3` hoặc thấp hơn)
  rồi chạy lại **toàn bộ**, với DB env vars đúng (xem port/role thật của môi trường,
  không tin `.env` mặc định nếu máy có nhiều Postgres container).
- **Pattern lỗi test hay gặp khi review lại: test chỉ pin "argument shape"
  (`expect(args.orderBy).toEqual(...)`) không chứng minh hành vi/kết quả thật.** Test
  dạng này không bắt được lỗi hoán đổi thứ tự phần tử trong mảng hay sai logic nếu
  object literal tình cờ vẫn giống — cần test dựa trên fixture có "áp dụng" đúng logic
  thật (order-aware mock tự sort theo đúng argument nhận được) hoặc test tích hợp với DB
  thật, không chỉ pin cấu trúc argument truyền vào mock.
- Fix an toàn phát hiện ở bước review này (docstring lỗi thời, test-coverage gap, refactor
  không đổi hành vi để tăng khả năng test): fix ngay trên branch hardening riêng
  (`chore/YYYYMMDD-review-hardening`), PR vào nhánh tích hợp, tự review qua comment PR —
  đúng quy trình 1-đơn-vị-việc-1-branch, không âm thầm sửa trực tiếp trên nhánh tích hợp.

---

## Migration an toàn cho production đang chạy (BẮT BUỘC)

Áp dụng cho **mọi thay đổi cần migration schema** (thêm cột, đổi kiểu, thêm constraint,
thêm enum value, đổi index). Database production đã có dữ liệu thật, đang có traffic
thật — không phải DB trống để thiết kế tự do.

Trước khi viết migration, tự trả lời đủ các câu sau (không bỏ qua câu nào):

- **Dữ liệu cũ đã tồn tại thì sao?** Cột mới NOT NULL không có DEFAULT sẽ fail ngay khi
  migrate nếu bảng có sẵn dữ liệu — cần DEFAULT, hoặc nullable + backfill riêng, hoặc
  2-bước (thêm nullable → backfill → đổi NOT NULL ở migration sau).
- **Trong lúc migrate, traffic thật vẫn đang ghi vào bảng đó không?** `ALTER TABLE` có
  thể khoá bảng lâu trên bảng lớn — cân nhắc `CREATE INDEX CONCURRENTLY` thay vì index
  thường, tách constraint validation ra bước riêng (`NOT VALID` rồi `VALIDATE
  CONSTRAINT` sau) nếu Postgres hỗ trợ, hoặc chạy off-peak.
- **Code cũ và code mới có thể cùng chạy song song không?** Deploy không atomic với
  migration — trong lúc rolling deploy, có thể có replica đang chạy code CŨ đọc/ghi
  schema MỚI, hoặc code MỚI đọc/ghi schema khi migration CHƯA chạy xong. Migration
  phải an toàn cho cả hai chiều tương thích ngược (backward-compatible) trong cửa sổ đó.
- **Rollback path là gì nếu migration lỗi giữa chừng?** Không phải "chạy lại từ đầu" là
  đủ — nếu migration bị dừng giữa chừng (timeout, connection drop), trạng thái DB còn
  lại phải an toàn để retry hoặc rollback, không kẹt ở trạng thái nửa vời.
- **Enum value mới có được MỌI nơi switch/if trên enum đó xử lý chưa?** Cùng nguyên tắc
  "Enum & Value Completeness" — grep hết mọi nơi tham chiếu enum, đọc từng chỗ, không
  chỉ thêm giá trị vào schema rồi coi là xong.
- **Multi-tenant/RLS**: cột/bảng mới có cần RLS policy riêng không? Xem
  `~/.claude/rules/ecc/common/security.md` và `.claude/rules/siteforce/multi-tenant.md`
  — bảng chứa dữ liệu tenant thiếu RLS là vi phạm nghiêm trọng theo rule đó.
- **Partial/WHERE index hoặc constraint** — Prisma DSL không diễn tả được partial unique
  index; nếu cần, viết raw SQL trong migration (xem `scans_tenant_urlhash_inflight_uniq`,
  `tier_grace_periods_tenant_pending_uniq` trong `packages/db/prisma/schema.prisma` làm
  ví dụ đã có sẵn trong repo) — không silently bỏ qua ràng buộc chỉ vì Prisma DSL không
  hỗ trợ trực tiếp.

**Khi migration cần thiết nhưng vượt quá phạm vi an toàn để làm ngay** (ví dụ: fix
chính cần cả code change lẫn migration, nhưng migration đó cần review/test kỹ hơn thời
gian cho phép trong 1 lần fix) — tách phần **an toàn, không cần migration** ra làm ngay
(nếu có), và file issue riêng cho phần cần migration, ghi rõ lý do tách — đúng tinh
thần "04b 横展開" (ghi rõ, không im lặng bỏ qua) áp dụng cho cả trường hợp "biết cần sửa
nhưng cố tình chưa sửa vì rủi ro migration".

---

## Terraform / hạ tầng phá hủy (BẮT BUỘC thận trọng)

Kế thừa quyết định #1113: **Claude không tự sửa Terraform trong batch fix này** — mọi
issue chạm tới `infrastructure/terraform/**` được giao lại cho thành viên phụ trách hạ
tầng (không tự apply, không tự merge). Phần dưới đây áp dụng cho cả trường hợp phải
*đọc/review* Terraform (để hiểu behavior, viết issue mô tả đúng) lẫn trường hợp hiếm
hoi được yêu cầu tường minh chỉnh sửa nó.

- **Không thêm bất kỳ thứ gì mang tính phá hủy vào Terraform** nếu không được yêu cầu
  tường minh: không set `force_destroy = true`, không hạ `prevent_destroy` đang bật,
  không đổi thuộc tính buộc provider phải destroy-and-recreate resource (đổi tên, đổi
  `identifier`, đổi zone/region trên resource stateful), không xoá `lifecycle` block
  đang bảo vệ dữ liệu (DB, S3/GCS bucket, KMS key, EBS volume).
- **Mọi thay đổi có khả năng destroy/replace phải chạy `terraform plan` trước và dán
  nguyên văn phần liệt kê `-/+` (destroy/recreate) hoặc `-` (destroy) vào PR** — không
  chỉ nói "đã plan xong". Nếu plan cho thấy destroy/replace một resource đang có dữ
  liệu thật (DB, storage, secret) mà **không phải mục đích chính của thay đổi** → dừng
  lại, đây là dấu hiệu thay đổi sai hướng, không phải thứ để "amend" cho qua. Nếu
  destroy/replace đúng là mục đích chính (đã xác nhận với người phụ trách hạ tầng) →
  vẫn phải ghi rõ thứ tự apply, downtime dự kiến, và có cần snapshot/backup trước khi
  apply không.
- **Không tự `terraform apply`** trong quy trình này. Sau khi có plan sạch, review kỹ,
  bàn giao cho người phụ trách hạ tầng apply — đúng tinh thần "chúng ta không sửa
  Terraform đâu, member khác sẽ làm".
- Nếu một issue/lateral-spread finding chỉ *mô tả đúng* một vấn đề hạ tầng nhưng fix
  thật sự nằm ở Terraform: filed issue riêng, gán cho người phụ trách hạ tầng, ghi rõ
  resource nào bị ảnh hưởng + có phải destroy/replace không — không tự ý bao gồm phần
  Terraform vào cùng PR code-fix.

---

## Không thương lượng

Bốn ranh giới cứng — vi phạm bất kỳ điều nào cũng coi như **chưa xong việc**, dù test có xanh.

1. **Không tick "done" nếu chưa chạy.** Nói "sẽ pass" không có giá trị bằng chạy lệnh
   và dán kết quả thật.
2. **Không claim test coverage nếu chưa revert-check.** Test xanh không có nghĩa là test đúng.
3. **Phần không làm phải ghi rõ, không im lặng bỏ.** "Cố ý để ngoài scope vì X" khác hoàn
   toàn với việc quên.
4. **Fix đụng dữ liệu thật / bảo mật / auth luôn làm sau cùng, soát kỹ nhất.** Sai ở đây
   không sửa được bằng một commit tiếp theo.

---

## Checklist bắt buộc (dùng TodoWrite)

Khi rule này áp dụng, tạo đúng 10 todo sau, không gộp, không bỏ:

- [ ] 00 Baseline: chạy test, ghi số liệu trước khi sửa
- [ ] 01 Branch riêng từ branch tích hợp (đã verify `git branch --show-current`)
- [ ] 02 Đọc code thật tại đúng dòng report trích — xác nhận/bác bỏ report
- [ ] 03 Sửa đúng nguyên nhân, theo convention có sẵn trong repo
- [ ] 04 Viết test: happy path + case biên + case không đổi
- [ ] 04b 横展開: quét toàn repo tìm cùng hình dạng bug, sửa hết candidate xác nhận được
      (hoặc ghi rõ lý do không sửa), test + revert-check riêng cho từng lateral fix
- [ ] 05 Revert-check: revert fix → chạy test mới → PHẢI fail → khôi phục
- [ ] 06 Typecheck mọi package bị ảnh hưởng, kể cả gián tiếp
- [ ] 07 Nếu bug UI: build thật, chạy trình duyệt thật, đo số liệu thật
- [ ] 08 Commit/PR giải thích lý do, nhắm vào branch tích hợp
- [ ] 09 Trả lời issue gốc trung thực, cập nhật board

---

## Câu tự vấn chống hợp lý hóa

Nếu nghĩ một trong các câu dưới đây → **DỪNG LẠI**, đó là đang tự hợp lý hóa việc bỏ bước:

| Suy nghĩ | Thực tế |
|---|---|
| "Issue này đơn giản, khỏi cần branch riêng" | Một issue = một branch. Không ngoại lệ. |
| "Report nói rõ dòng rồi, khỏi cần đọc code" | Report sai dòng/sai cơ chế là chuyện thường. Đọc. |
| "Test pass rồi, revert-check phí thời gian" | Test pass không chứng minh gì. Revert-check là bắt buộc. |
| "Chỉ sửa 1 dòng, khỏi typecheck cả repo" | Type dùng chung lan ra nhiều package. Typecheck. |
| "Bug CSS này nhìn code là biết đúng" | Bug UI phải đo bằng số thật trong trình duyệt thật. |
| "Cái này chắc pass, ghi done trước" | Chưa chạy = chưa done. |
| "Phần kia ngoài scope, khỏi nhắc" | Ngoài scope phải GHI RÕ kèm lý do. |
| "Issue chỉ liệt kê 5 chỗ, sửa đúng 5 chỗ là xong" | Report là mẫu tìm được, không phải danh sách đầy đủ. 横展開 bắt buộc — quét cùng hình dạng bug trong toàn repo. |
| "Đang chat tiếng Việt nên comment GitHub tiếng Việt cũng được" | Không. GitHub là nội dung team-facing công khai — chỉ tiếng Anh/tiếng Nhật, không có ngoại lệ dù đang trao đổi tiếng Việt trong phiên. |
| "Cuối phiên cập nhật board 1 lần cho tiện" | Cập nhật ngay khi trạng thái thật đổi — mỗi issue độc lập theo tiến độ thật của nó. |
| "Đã review toàn batch rồi, khỏi review riêng từng issue nữa" | Hai lớp review là bắt buộc cả hai, không phải chọn 1. Review batch không thay được review riêng từng PR và ngược lại. |
| "Full test suite crash do máy quá tải, thôi chạy targeted cho nhanh" | Không được bỏ full suite chỉ vì lần chạy đầu crash — hạ concurrency rồi chạy lại toàn bộ, không né việc chạy hết. |

---

## Liên quan

- [../../../CLAUDE.md](../../../CLAUDE.md) — SiteforceEdge cross-check
- `~/.claude/rules/ecc/common/code-review.md` — chuẩn code review chung
- `~/.claude/rules/ecc/common/testing.md` — yêu cầu test coverage
