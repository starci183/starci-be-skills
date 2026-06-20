---
name: starci-be-cannon-apply
description: >
  Khi viết code BACKEND MỚI (module/service/resolver/entity/handler...) trong repo StarCi, tự động ÁP
  bộ "Code Cannon" (cannon/CODE-CANNON.md — code pattern chuẩn rút từ source thật) ngay từ lúc gõ, rồi
  tự-kiểm trước khi xong. Khác `-audit` (soi code cũ, chỉ báo cáo): skill này là chế độ VIẾT cho code
  mới sao cho đúng chuẩn từ đầu. Trigger khi user gõ `/starci-be-cannon-apply`, hoặc khi tạo
  module/service/resolver/entity/CQRS-handler mới trong backend, hoặc nói "viết code BE theo chuẩn",
  "tạo module BE chuẩn cannon", "apply code pattern backend".
---

# /starci-be-cannon-apply — Viết code BE mới đúng chuẩn Code Cannon từ đầu

Chế độ VIẾT: mọi code backend mới phải tuân **Code Cannon** ngay khi sinh ra — không để lệch rồi mới sửa.

## Bước 0 — đọc Cannon trước (bắt buộc, trước khi gõ dòng code đầu)
Đọc **`cannon/CODE-CANNON.md`** + các section liên quan đến thứ đang viết:
- Tạo **module** → §Module & DI (`.module-definition.ts` + `.module.ts` + register isGlobal + parent aggregator).
- **Resolver/GraphQL** → §GraphQL (Input/ObjectType, @Query/@Mutation, gọi service).
- **Entity / data-access** → §TypeORM (`InjectEntityManager`, KHÔNG `InjectRepository`; transaction).
- **CQRS** → §CQRS (command/query handler, bus, projection).
- Mọi file → §Type-safety & naming (`export const` arrow, `{Action}Params/{Action}Result`, branded types,
  `import type`, JSDoc per-member, English-only).

## Quy trình
1. Đọc Cannon (§ liên quan). Nếu thiếu ngữ cảnh repo → đọc 1–2 file mẫu THẬT cùng loại trong source để
   bắt chước đúng cấu trúc thư mục/naming hiện hành (đừng bịa style).
2. Thiết kế: file nào · tầng nào · params/result type nào · entity/DI wiring ra sao — theo Cannon.
3. Viết code. Trong lúc viết bám sát: 1 file = 1 concern; type/enum/util tách file; JSDoc đầy đủ; comment
   logic line-by-line (theo Cannon §Type-safety).
4. **Tự-kiểm trước khi báo xong** — chạy nhanh checklist Cannon trên code vừa viết (như mini `-audit`):
   không sai tầng · data-access đúng · không inline object type trong generic · naming đúng · JSDoc đủ ·
   English-only · barrel/import đúng hướng. Sửa hết BLOCKER do chính mình tạo.
5. `npx tsc --noEmit` + `npm run lint` sạch (trừ baseline lỗi pre-existing đã biết).

## Nguyên tắc
- **Cannon là luật, không phải gợi ý.** Cần phá 1 rule → nêu lý do rõ + hỏi user, đừng phá lặng lẽ.
- **Bắt chước source thật > trí nhớ.** Cấu trúc/naming theo file cùng loại đang có trong repo.
- **Không lặp lại lỗi đã biết.** Pattern xấu Cannon cấm = tuyệt đối tránh ngay từ bản nháp đầu.
- Học được nguyên tắc mới khi viết → ghi `cannon/drafts/<temp>.md` (gộp vào CODE-CANNON.md khi `/merge`),
  KHÔNG sửa CODE-CANNON.md trực tiếp giữa lúc làm.

→ Viết xong: tóm tắt file đã tạo + xác nhận đã qua self-check Cannon. Muốn rà kỹ code cũ quanh đó → `/starci-be-cannon-audit`.
