# bow-skill-flutter

Bộ skill cho công nghệ **Flutter**, dùng với [bow-agent](https://github.com/Bow-T/bow-agent).
Theo hướng **mỗi công nghệ một repo**: bộ này chỉ lo phần Flutter (màn hình, model, đa ngôn ngữ, layout,
test). Backend là bộ khác — một dự án tick bộ này CÙNG với `supabase` hoặc `flutter-api` (REST) ở ô **Công nghệ**.

## Cấu trúc

```
bow-skill-flutter/
├── bow-skill.json        # manifest: id "flutter" + quy tắc tự nhận diện repo + `needs`
├── THIRD_PARTY.md        # skill nào chép từ đâu, commit nào, giấy phép gì; skill đã xem nhưng không lấy
├── LICENSES/             # văn bản giấy phép của nguồn bên ngoài
└── skills/
    ├── bow-ui/                           # team — màn hình + view-model theo khuôn MVVM của team
    ├── bow-model/                        # team — model @JsonSerializable + build_runner
    ├── bow-i18n/                         # team — mọi chuỗi người dùng đọc được đều qua l10n
    ├── flutter-realtime/                 # team — Supabase realtime trong Flutter (cần bộ `supabase`)
    ├── flutter-fix-layout-issues/        # ⬇ nguồn chính thức — sửa lỗi overflow / unbounded constraints
    ├── flutter-build-responsive-layout/  # ⬇ nguồn chính thức — layout co giãn theo kích thước cửa sổ
    └── flutter-add-widget-test/          # ⬇ nguồn chính thức — widget test bằng WidgetTester
```

| Skill | Dùng khi | Nguồn |
| --- | --- | --- |
| `bow-ui` | tạo / sửa màn hình, page, view-model, widget, dialog | team Bow |
| `bow-model` | tạo / sửa model, parse payload, thêm field / enum, đụng file sinh `.g.dart` / `.gr.dart` | team Bow |
| `bow-i18n` | thêm / sửa chữ hiển thị; đưa một cột lấy từ DB / admin lên màn hình; bug "chưa dịch" | team Bow |
| `flutter-realtime` | thêm / sửa subscription realtime, badge / số đếm cập nhật trực tiếp | team Bow |
| `flutter-fix-layout-issues` | gặp "RenderFlex overflowed", "Vertical viewport was given unbounded height"… | Flutter (BSD-3-Clause) |
| `flutter-build-responsive-layout` | giao diện phải đẹp trên cả điện thoại lẫn tablet / desktop | Flutter (BSD-3-Clause) |
| `flutter-add-widget-test` | viết test cho một widget: hiển thị đúng, phản ứng với bấm / cuộn / gõ | Flutter (BSD-3-Clause) |

Bốn skill của team mô tả khuôn base source Flutter của team Bow (`BaseViewModel` + `MixinBasePage`,
`AppColor`, `context.appStyle`, `Utilities`, `S.of(context)` sinh bằng `intl_utils`, chạy lệnh qua `fvm`).
Chúng đã được gỡ phần riêng của một sản phẩm: mã ticket, đường dẫn monorepo (`apps/mobile` giờ chỉ là ví dụ),
tên hàm / bảng trong các bài học từ sự cố, và cặp ngôn ngữ EN/FR giờ chỉ là ví dụ cho "các ngôn ngữ app hỗ trợ".

Ba skill còn lại chép nguyên văn từ [flutter/agent-plugins](https://github.com/flutter/agent-plugins) —
chi tiết ở `THIRD_PARTY.md`.

**Khi hai nguồn nói khác nhau, quy ước team thắng.** Skill chính thức viết lệnh `flutter …` và ví dụ
dùng `Colors.*`, `Text('…')` trần; ở dự án theo khuôn team thì chạy qua `fvm` và theo `bow-ui`
(`AppColor`, `context.appStyle`). Điều này ghi ngay trong `bow-ui` (mục "Precedence") để agent đọc được —
README thì agent không đọc.

Cố ý KHÔNG lấy `flutter-apply-architecture-best-practices`, `flutter-implement-json-serialization`,
`flutter-setup-localization`, `flutter-setup-declarative-routing` vì chúng đụng quy ước của team
(`bow-ui` / `bow-model` / `bow-i18n`); `flutter-add-integration-test` vì nó gọi tool MCP trùng tên với
tool test-sim của bow; `dart-resolve-package-conflicts` vì nó nâng cả đồ thị phụ thuộc. Đừng thêm lại —
lý do từng cái ở bảng "Đã xem nhưng KHÔNG lấy" của `THIRD_PARTY.md`.

## Ghép với bộ khác (`needs`)

`bow-skill.json` khai:

```json
"needs": { "flutter-realtime": ["supabase"] }
```

Nghĩa là skill `flutter-realtime` (Supabase realtime trong Flutter) chỉ được trải khi dự án tick CẢ bộ
`supabase`. Dự án Flutter gọi REST API riêng (không tick `supabase`) sẽ không có skill này; sáu skill còn lại
vẫn được trải. Lưu ý: ví dụ trong `bow-ui` / `bow-i18n` vẫn nhắc Supabase (`SupabaseError.handle`, cột lấy từ
Supabase / admin) vì base source của team đang dùng Supabase — bài học thì áp cho backend nào cũng được.

Skill của team còn trỏ tới `[[bow-commit]]` (bộ `commit`), `[[impact-sweep]]` và
`[[verify-runtime-not-just-static-green]]` (bộ `process`). Đó chỉ là tham chiếu: dự án không tick các bộ
ấy thì skill ở đây vẫn dùng được, chỉ không có skill được trỏ tới.

## Tự nhận diện

bow-agent tự tick bộ này cho dự án chưa khai công nghệ khi repo khớp quy tắc ở `targets.detect`:

| Quy tắc | Nghĩa |
| --- | --- |
| `pubspec.yaml` chứa `sdk: flutter` | repo có app / package Flutter (tìm cả trong thư mục con, vd `apps/mobile/pubspec.yaml`) |

Chỉ soi file git đã track. Package Dart thuần (không phụ thuộc Flutter SDK) sẽ KHÔNG khớp.

## Đăng ký vào bow-agent

Thêm vào `stacks` của `~/.bow-agent/registry.json` (bow chỉ tải repo có trong danh sách này, đúng tag đã ghim):

```json
{ "id": "flutter", "label": "Flutter", "repo": "github.com/Bow-T/bow-skill-flutter", "ref": "v2.0.0" }
```

Lưu ý trong lúc chuyển tiếp: id cũ `flutter-supabase` (tag v1.x của chính repo này) vẫn tải được cho tới khi
registry đổi sang id mới — tag cũ không bị đụng tới.

Thử local trước khi đẩy GitHub: `"repo": "file:///đường/dẫn/tới/bow-skill-flutter"`.

## Đổi gì so với v1

v1 (id `flutter-supabase`) là bundle ghép 31 skill "Flutter + Supabase". v2 chỉ còn phần Flutter — 7 skill:

| Trước ở đây | Giờ ở đâu |
| --- | --- |
| skill Supabase (`supabase-migration`, `supabase-security-review`) | [bow-skill-supabase](https://github.com/Bow-T/bow-skill-supabase) |
| skill Stripe (`stripe-best-practices`, `stripe-projects`, `upgrade-stripe`) | bow-skill-stripe |
| skill quy trình + Jira (`impact-sweep`, `bow-lean`, `test-driven-development`, `jira-triage-bug-qc`…) | bow-skill-process |
| `bow-commit` | [bow-skill-commit](https://github.com/Bow-T/bow-skill-commit) |
| `watch` | vốn đã có ở bow-skill-core |
| skill riêng của một sản phẩm (`express-realtime-review`, `notification-lifecycle`, `parity-with-sibling-surface`, `store-status-lifecycle`, `bow-read-path`, `bow-web-form`, `codemagic-build-prod`) | KHÔNG còn ở đây — thuộc về repo của sản phẩm |
| `monorepo/CLAUDE.md` + `monorepo/hooks/` | KHÔNG còn ở đây — thuộc về repo của sản phẩm |

Thay đổi khác:

- Skill nằm ở `skills/` ngay gốc repo; các thư mục `monorepo/`, `agent-skills/`, `prompt/` đã xoá. Manifest
  không còn `monorepoDir`. Hai thư mục sau chỉ là bản backup skill core của bow-agent (`watch`, `qc-triage`,
  `coding-convention`), vốn không nằm trong `skillsDir`.
- Thêm ba skill chính thức của Flutter (sửa lỗi layout, layout co giãn, widget test).
- Quy tắc tự nhận diện hẹp lại: chỉ còn `pubspec.yaml` chứa `sdk: flutter` (v1 khớp cả chuỗi `flutter` bất kỳ
  và sự tồn tại của `apps/mobile`, `lib/src/pages`).

## Phát hành phiên bản mới

Sửa skill qua PR → tăng `version` trong `bow-skill.json` → gắn tag (`git tag v2.0.1 && git push --tags`)
→ cập nhật `ref` trong registry. bow tải theo tag và cache theo `<id>@<ref>`, nên sửa nội dung mà giữ
nguyên tag sẽ KHÔNG được tải lại. Cập nhật skill từ nguồn bên ngoài: xem cuối `THIRD_PARTY.md`.
