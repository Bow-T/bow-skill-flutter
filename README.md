# bow-skill-flutter

Bộ **skill đầy đủ** cho stack **Flutter + Supabase** của team Bow, dùng với
[bow-agent](https://github.com/Bow-T/bow-agent). Đây là bản mirror của toàn bộ thư
mục `skills/` trong bow-agent — vừa để **backup/đồng bộ**, vừa để bow **tải tự động**
khi chọn stack Flutter.

## Cấu trúc

```
bow-skill-flutter/
├── bow-skill.json      # manifest — skillsDir trỏ "monorepo/skills"
├── monorepo/
│   ├── skills/         # 24 skill THẬT của dự án — bow trải TỪ ĐÂY
│   │   ├── bow-ui/ flutter-realtime/ bow-model/ supabase-migration/ …
│   ├── hooks/          # git hooks của monorepo
│   └── CLAUDE.md       # cẩm nang dự án (đã lọc tên thật/board/account ID)
├── agent-skills/       # skill kèm code dùng chung (watch, qc-triage) — backup
├── prompt/             # skill prompt-only (coding-convention) — backup
└── README.md
```

**Mấu chốt:** `bow-skill.json` có `"skillsDir": "monorepo/skills"` → bow-agent trải
đúng 24 skill Flutter thật vào `.claude/skills/` của repo đích khi chọn stack này.
Các thư mục `agent-skills/` và `prompt/` là **backup đồng bộ** (không nằm trong
skillsDir nên bow không tự trải — chúng vốn là skill core của bow-agent).

## Đã lọc thông tin nội bộ

`monorepo/CLAUDE.md` và vài skill có tên thật/board Jira/account ID đã được thay bằng
placeholder (`<TECH_LEAD>`, `<BOARD_ID>`, `<JIRA_ACCOUNT_ID>`…). Điền lại giá trị
thật khi dùng nội bộ.

## Phát hành phiên bản mới

Sửa skill → tăng `version` trong `bow-skill.json` → gắn tag:
```bash
git tag v1.0.1 && git push --tags
```
Rồi báo admin cập nhật `ref` trong registry của bow-agent (bow luôn tải theo tag).
