---
name: starci-be-cannon-audit
description: >
  Soi (audit) source backend CŨ xem lệch chuẩn "Code Cannon" của StarCi tới đâu. Đọc bộ cannon
  (cannon/CODE-CANNON.md — code pattern chuẩn rút từ source thật) rồi quét file/module/diff được chỉ
  định, liệt kê VI PHẠM kèm rule bị phá · file:line · mức độ · cách sửa. KHÔNG tự sửa (chỉ báo cáo;
  fix khi user duyệt). Dùng cho code có sẵn / PR / module legacy. Trigger khi user gõ
  `/starci-be-cannon-audit <path|module|diff>`, hoặc nói "audit code chuẩn BE", "soi lệch chuẩn",
  "check pattern backend", "nghiệm thu code module BE".
---

# /starci-be-cannon-audit — Soi source cũ lệch chuẩn Code Cannon

Đối chiếu code BACKEND đang có với **Code Cannon** (SSOT code-pattern, grounded từ source thật) → ra
**báo cáo vi phạm** để sửa. Audit = CHỈ BÁO CÁO; không tự ý đổi code.

## Bước 0 — đọc Cannon trước (bắt buộc)
Đọc **`cannon/CODE-CANNON.md`** (bộ luật: Module & DI · GraphQL · REST · TypeORM data-access · CQRS &
projection · Type-safety & naming · Cross-cutting). Đây là chuẩn để chấm; KHÔNG chấm theo cảm tính.

## Phạm vi
- `path/module` → quét toàn bộ file `.ts` trong đó.
- `diff` / không tham số → chỉ chấm phần thay đổi (`git diff`), nhanh cho PR.

## Quy trình
1. Đọc Cannon → rút checklist rule (mỗi rule có id ngắn).
2. Đọc code đích (Read/Grep). Với codebase lớn → spawn Explore/agent quét song song theo nhóm rule.
3. Mỗi vi phạm ghi: **rule bị phá · `file:line` · trích đoạn sai · mức độ (BLOCKER/WARN/NIT) · cách sửa đúng** (1 dòng, theo Cannon).
4. Gom theo domain (Module/GraphQL/TypeORM/CQRS/Naming/...). Thống kê: #BLOCKER / #WARN / #NIT.
5. **Output:** in bảng vi phạm trong chat + (nếu ≥10 vi phạm hoặc user yêu cầu) ghi `.audits/cannon/<scope>-<date>.md`.

## Nguyên tắc
- **CHỈ chấm cái Cannon có nói** — không phát minh luật mới. Thấy pattern xấu mà Cannon chưa phủ → ghi
  vào mục "Đề xuất bổ sung Cannon" CUỐI báo cáo (không tính là vi phạm).
- **Bằng chứng bắt buộc:** mỗi vi phạm phải có `file:line` thật. Không chắc → để mục "Cần xác minh".
- **Không tự sửa.** Cuối báo cáo hỏi: "Sửa các BLOCKER không?" → nếu user đồng ý mới sửa (hoặc chuyển
  `/starci-be-cannon-apply` để viết lại đúng chuẩn).
- Mức độ: **BLOCKER** = phá MUST/NEVER (sai tầng, sai data-access, lộ type inline); **WARN** = lệch SHOULD;
  **NIT** = style nhỏ.

→ Sửa xong / user feedback nguyên tắc mới → ghi `cannon/drafts/<temp>.md` (đừng sửa CODE-CANNON.md trực tiếp; gộp khi `/merge`).
