# Nguồn bên ngoài

Skill chép từ repo khác được ghi ở đây: lấy từ đâu, tại commit nào, giấy phép gì, và đã sửa gì.
Bản chép là **ảnh chụp tại một commit** — bow-agent không tự kéo bản mới của nguồn. Muốn cập nhật
thì làm lại các bước ở cuối file, đọc lại nội dung, rồi mới gắn tag.

| Skill | Nguồn | Commit | Giấy phép | Đã sửa |
| --- | --- | --- | --- | --- |
| `flutter-add-widget-test` | [flutter/agent-plugins](https://github.com/flutter/agent-plugins) `skills/flutter-add-widget-test` | `0ef3972f93e2baa4156ba1cbb1e515cd53079c68` (2026-09-30) | BSD-3-Clause | Không — nguyên văn |
| `flutter-build-responsive-layout` | [flutter/agent-plugins](https://github.com/flutter/agent-plugins) `skills/flutter-build-responsive-layout` | `0ef3972f93e2baa4156ba1cbb1e515cd53079c68` (2026-09-30) | BSD-3-Clause | Không — nguyên văn |
| `flutter-fix-layout-issues` | [flutter/agent-plugins](https://github.com/flutter/agent-plugins) `skills/flutter-fix-layout-issues` | `0ef3972f93e2baa4156ba1cbb1e515cd53079c68` (2026-09-30) | BSD-3-Clause | Không — nguyên văn |

Văn bản giấy phép: [`LICENSES/flutter-agent-plugins-BSD-3-Clause.txt`](LICENSES/flutter-agent-plugins-BSD-3-Clause.txt)
(file `LICENSE` ở gốc repo nguồn; `.claude-plugin/plugin.json` của nguồn cũng khai `BSD-3-Clause`, plugin
`dart-flutter` bản 1.0.6).

Mỗi skill ở nguồn chỉ có đúng một file `SKILL.md` — không có CHANGELOG, file mẫu hay ảnh nào để bỏ,
nên không bỏ gì.

Ghi chú nguồn gốc: commit của nguồn mang tên "auto-sync skills directory from dart-lang/skills", và
README của nguồn trỏ tới `dart-lang/skills` cho các skill `dart-*`. Bản ở đây chép từ
`flutter/agent-plugins` (BSD-3-Clause). Giấy phép của riêng repo `dart-lang/skills`: **chưa kiểm**.

## Đã soát gì trước khi chép (2026-10-02)

- Không có script / file thực thi nào trong ba skill — chỉ Markdown, mỗi skill một file thuần ASCII.
- Không có ký tự vô hình (tag Unicode, zero-width, bidi), không có đường link nào.
- Lệnh skill bảo chạy: `flutter pub add`, `flutter test`. Không có lệnh cài đặt kéo từ mạng kiểu `curl … | sh`.
- Không có mục nào bảo agent gửi phản hồi / mở issue.
- Mô tả của `flutter-fix-layout-issues` nhắc "Dart and Flutter MCP tools" nhưng phần thân không gọi tool MCP nào.
- `flutter-build-responsive-layout` có câu "Do not lock screen orientation" và `flutter-add-widget-test` chỉ bọc
  widget trong `MaterialApp` (không nói tới `S.delegate` hay locator mà page của team cần) — `bow-ui` có mục
  "Precedence" nói rõ quy ước team thắng.
- Frontmatter có `metadata.model: models/gemini-3.1-pro-preview`: nguồn sinh skill bằng công cụ của họ
  (`resources/flutter_skills.yaml` + `tool/generator`). Chỉ là siêu dữ liệu, không ảnh hưởng cách bow nạp skill.

## Đã xem nhưng KHÔNG lấy

Ba dòng đầu chép từ đánh giá team đã làm ở `bow-skill-flutter-api` (giữ nguyên lý do); ba dòng kế là
skill đụng thẳng quy ước của `bow-ui` / `bow-model` / `bow-i18n`; hai dòng cuối bị loại sau khi đọc (2026-10-02).

| Skill | Nguồn | Vì sao |
| --- | --- | --- |
| `flutter-use-http-package`, `flutter-implement-json-serialization` | flutter/agent-plugins (chính thức, BSD-3) | Dạy package `http` + JSON viết tay + `FutureBuilder`, trái khuôn dio/retrofit/json_serializable/VM; mô tả lại bắt đúng mọi việc "gọi REST API" nên kéo agent sai hướng |
| `dart-run-static-analysis`, `dart-fix-runtime-errors` | flutter/agent-plugins | Xui `dart fix --apply` / `dart format .` cả repo và bật `strict-casts` → diff khổng lồ; mô tả `dart-fix-runtime-errors` lệch nội dung (nhắc tool MCP không có) |
| `dart-generate-test-mocks`, `dart-add-unit-test` | flutter/agent-plugins | Giả định inject qua constructor + `http.Client`; nhiều repo Flutter + API dùng GetIt (service locator), constructor injection không áp được. Xem lại khi team bắt đầu viết test |
| `flutter-apply-architecture-best-practices` | flutter/agent-plugins | Dựng MVVM theo khuôn khác: Repository + Service, inject Repository vào ViewModel qua constructor, View nghe bằng `ListenableBuilder` — đụng `bow-ui` (`BaseViewModel` + `MixinBasePage`, query singleton lấy qua locator) |
| `flutter-setup-localization` | flutter/agent-plugins | Dạy `generate: true` + `l10n.yaml` + `AppLocalizations` (gen-l10n) — đụng `bow-i18n` (`flutter_intl` / `intl_utils`, `S.of(context)`; `flutter gen-l10n` hỏng ở khuôn này) |
| `flutter-setup-declarative-routing` | flutter/agent-plugins | Dạy `go_router` (`GoRouter`, `context.go()`) — đụng routing của `bow-ui` (`@RoutePage()` + `app_router.gr.dart` sinh bằng build_runner, `context.router.push`) |
| `flutter-add-integration-test` | flutter/agent-plugins | Bảo agent gọi tool của Dart/Flutter MCP server (`launch_app`, `tap`, `enter_text`, `scroll`…) — bow không nạp MCP đó, và `launch_app` / `tap` TRÙNG TÊN tool của MCP test-sim của bow; mô tả lại bắt cả "exploring UI components via MCP" nên dễ bật nhầm trong lượt 📱 Test sim / 🌐 Quay web. Còn xui sửa entry point thật (`enableFlutterDriverExtension()` trong `lib/main.dart`) và `pumpWidget(MyApp())` không qua bootstrap của app |
| `dart-resolve-package-conflicts` | flutter/agent-plugins | Thân bài rộng hơn mô tả: bảo chạy `dart pub upgrade` / `dart pub upgrade --tighten` — nâng cả đồ thị phụ thuộc, diff lớn; dùng `dart` trần trong khi dự án ghim SDK qua `fvm`. Điểm tốt đáng nhớ: "NEVER delete the entire `pubspec.lock`" |

`flutter-implement-json-serialization` ngoài lý do ở dòng đầu còn đụng `bow-model`: skill đó dạy viết tay
`fromJson` / `toJson` bằng `dart:convert`, còn `bow-model` cấm viết tay và bắt dùng `@JsonSerializable` + build_runner.

Các skill còn lại của nguồn (`dart-build-cli-app`, `dart-collect-coverage`, `dart-migrate-to-checks-package`,
`dart-setup-ffi-assets`, `dart-use-doc-examples`, `dart-use-ffigen`, `dart-use-path-package`,
`dart-use-pattern-matching`, `dart-use-primary-constructors`, `dart-write-documentation`,
`flutter-add-widget-preview`) **chưa được đánh giá** cho bộ này.

## Cập nhật từ nguồn

```bash
git clone --depth 1 https://github.com/flutter/agent-plugins /tmp/up && git -C /tmp/up rev-parse HEAD
# chạy ở gốc repo này: so từng skill với bản ở đây, ĐỌC phần khác biệt (đặc biệt lệnh / link / mục mới), rồi chép đè
for s in flutter-add-widget-test flutter-build-responsive-layout flutter-fix-layout-issues; do
  diff -r /tmp/up/skills/$s skills/$s
done
diff /tmp/up/LICENSE LICENSES/flutter-agent-plugins-BSD-3-Clause.txt
```
Sửa bảng ở trên (commit + ngày), tăng `version` trong `bow-skill.json`, gắn tag mới.
