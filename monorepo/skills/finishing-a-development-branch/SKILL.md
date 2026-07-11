---
name: finishing-a-development-branch
description: Chốt một nhánh feature khi code đã xong — verify hành vi thật ĐÃ pass → xác nhận nhánh/ticket → trình cho user chọn hướng tích hợp (push auto-merge develop / mở MR review / để nguyên chờ thêm) → dọn dẹp. KHÔNG tự merge, KHÔNG tự push khi user chưa chọn — push ở repo này có thể auto-merge vào develop nên đây là thao tác hệ trọng, phải hỏi. Dùng khi task/plan làm xong hết, khi user nói "xong rồi", "chốt nhánh này", "merge chưa", "giờ làm gì tiếp". Đi SAU [[verify-runtime-not-just-static-green]] và TRƯỚC/BAO quanh [[bow-commit]] (bow-commit lo commit+push; skill này lo quyết định tích hợp).
---

# finishing-a-development-branch — Chốt nhánh đúng cách

## Vì sao có skill này

Xong code không có nghĩa là xong việc. Ở monorepo <PROJECT_KEY>, **push có thể auto-merge vào
`develop`** (xem guard-push hook + quest gate) — nên quyết định "push hay chưa, merge kiểu gì"
là thao tác hệ trọng, không được tự ý. Skill này chốt một nhánh feature bằng cách: verify →
trình lựa chọn → thực thi lựa chọn của user → dọn dẹp.

**Nguyên tắc cốt lõi:** Verify hành vi thật → xác nhận môi trường → TRÌNH lựa chọn → làm theo
user chọn → dọn dẹp. Không nhảy cóc, không tự merge.

## Bước 1 — Verify hành vi (BẮT BUỘC, trước mọi thứ khác)

Chạy [[verify-runtime-not-just-static-green]] cho đúng tầng đã đổi. Tối thiểu:

```bash
git rev-parse --abbrev-ref HEAD          # đang ở nhánh feature?
cd apps/mobile && fvm flutter analyze     # nếu chạm mobile — 0 error, 0 warning
```
- Có test liên quan → chạy (`fvm dart test` / `bun test`), xem xanh THẬT, không chỉ "đã viết".
- Chạm Supabase → test pgTAP ở `supabase/tests/` liên quan phải xanh; RLS đúng role user thật.

**Nếu còn fail:** dừng lại. Báo số fail + trích output. **Không** sang bước 2, không trình lựa
chọn merge khi còn đỏ.

## Bước 2 — Xác nhận nhánh & ticket

- Đang trên nhánh feature `<type>/<PROJECT_KEY>-XXX-slug`, **không** phải `develop`/`main`.
- Bóc Jira ticket từ tên nhánh. Không có ticket → hỏi user (commit footer cần `Refs: <PROJECT_KEY>-XXX`).
- `git status` sạch hoặc đã stage đúng phạm vi (loại `.env*`, `.claude/settings.local.json`, secrets).
- Chạm cross-cutting (enum/status/shared/schema) → chạy [[impact-sweep]] xác nhận không sót consumer.

## Bước 3 — TRÌNH lựa chọn (đây là lúc DỪNG hỏi user)

Đừng tự quyết. Nêu gọn hiện trạng (nhánh, số commit ahead, ticket) rồi hỏi user chọn:

| Hướng | Khi nào hợp | Việc sẽ làm |
|-------|-------------|-------------|
| **A. Commit + push** (có thể auto-merge develop) | Code đã verify, đủ tự tin vào develop | Chạy [[bow-commit]] (safety scan → analyzer → commit chuẩn → push). Lưu ý push kích hoạt quest gate + có thể auto-merge. |
| **B. Commit local, mở MR review** | Cần review trước khi vào develop, hoặc thay đổi rủi ro | Commit qua [[bow-commit]] nhưng **dừng trước push tự-merge**; đẩy nhánh và tạo MR để người review (chạy [[receiving-code-review]] khi có feedback). |
| **C. Chưa chốt** | Còn hạng mục con, hoặc chờ QC/việc khác | Để nguyên nhánh, ghi rõ còn thiếu gì. Không push. |

Chỉ sau khi user chọn mới thực thi. User chưa chọn = không push, không merge.

## Bước 4 — Thực thi lựa chọn

- **A/B** → giao cho [[bow-commit]] (nó tự lo safety scan RLS/CORS/secrets, analyzer,
  commit message chuẩn Conventional + `Refs:`, và — với A — push). Không tự viết lại pipeline
  đó ở đây; skill này chỉ là tầng quyết định phía trên.
- **B** cụ thể: sau khi push nhánh (không phải merge), tạo MR trỏ vào `develop`, mô tả gọn
  WHAT/WHY + ticket. **Không** thêm trailer AI-authorship vào commit hay mô tả MR
  (docs/conventions/common.md → Git safety).
- **C** → in checklist việc còn lại, không đụng git remote.

## Bước 5 — Dọn dẹp

- Sau khi merge/MR: xác nhận nhánh đã lên remote đúng, không để lại file rác đã stage nhầm.
- Không tự xóa nhánh remote/local trừ khi user yêu cầu.
- Nhắc user (không tự làm) nếu cần cập nhật trạng thái Jira ticket — MCP Jira ở chế độ read-only.

## Ranh giới với các skill khác

- **[[verify-runtime-not-just-static-green]]** = bước 1 (bằng chứng). Skill này KHÔNG thay nó,
  nó gọi nó.
- **[[bow-commit]]** = cơ chế commit+push. Skill này là tầng **quyết định tích hợp** bao quanh,
  không lặp lại pipeline của bow-commit.
- **[[receiving-code-review]]** = khi chọn hướng B và MR có feedback.
