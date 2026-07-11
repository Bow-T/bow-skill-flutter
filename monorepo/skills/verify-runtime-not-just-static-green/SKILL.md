---
name: verify-runtime-not-just-static-green
description: KHÔNG được tuyên bố "đã xong / đã fix / pass / chạy được" khi CHƯA chạy lệnh verify thật ngay trong lượt này — bằng chứng trước lời nói, luôn luôn. Analyzer xanh và tsc sạch KHÔNG phải là "chạy được": chúng bắt kiểu/cú pháp, không bắt hành vi sai (RLS chặn, enum thiếu case ở runtime, realtime không re-sync, Postgres constraint 23514, i18n rỗng khi đổi ngôn ngữ). Dùng skill này ngay TRƯỚC khi nói task hoàn thành, trước khi commit/push, trước khi trả lời "đã sửa xong", hoặc khi định khoe kết quả. Bổ trợ [[impact-sweep]] (quét blast radius), [[debugging-and-error-recovery]] (tìm root cause) và [[bow-commit]] (self-verify rubric trước push). Backstop bằng Stop hook self-verify-rubric.sh.
---

# verify-runtime-not-just-static-green — Bằng chứng trước lời nói

## Vì sao có skill này

Trong stack <PROJECT_KEY> (Flutter + Supabase + Next.js admin), phần lớn bug QC bắt được
là **runtime**, không phải kiểu tĩnh: RLS chặn đúng người dùng thật, enum widen nhưng một
màn hình không handle case mới, realtime không re-sync khi row đổi live, Postgres bắn
`23514` (CHECK constraint), chuỗi vẫn tiếng Anh khi bật FR. `fvm dart analyze` xanh và
`tsc --noEmit` sạch **không** thấy nổi mấy lỗi này — chúng chỉ chứng minh code **biên dịch
được**, không chứng minh code **chạy đúng**.

Tuyên bố "đã xong" mà chưa chạy verify thật là **nói dối, không phải hiệu quả**.

## Luật sắt

```
KHÔNG TUYÊN BỐ HOÀN THÀNH KHI CHƯA CÓ BẰNG CHỨNG VERIFY MỚI TRONG LƯỢT NÀY
```

Nếu bạn chưa chạy lệnh verify **ngay trong tin nhắn này**, bạn không được nói nó pass.
Bằng chứng cũ từ 5 tin nhắn trước không tính — code đã đổi từ lúc đó.

## Cổng kiểm tra — chạy TRƯỚC mọi lời tuyên bố

```
TRƯỚC khi nói bất kỳ trạng thái nào ("xong", "fix rồi", "pass", "chạy được", "ổn rồi"):

1. XÁC ĐỊNH: Lệnh nào / thao tác nào CHỨNG MINH được điều này?
2. CHẠY: Thực thi ĐẦY ĐỦ lệnh đó (mới, trọn vẹn — không chạy lại từ cache đầu óc)
3. ĐỌC: Toàn bộ output, exit code, đếm số lỗi/fail
4. ĐỐI CHIẾU: Output có xác nhận đúng điều mình định nói không?
   - KHÔNG → nói trạng thái THẬT kèm bằng chứng, đừng tô hồng
   - CÓ → nói kết luận KÈM bằng chứng (trích output)
5. CHỈ SAU ĐÓ mới tuyên bố

Bỏ bất kỳ bước nào = đang nói dối, không phải verify.
```

## "Verify" nghĩa là gì theo từng tầng <PROJECT_KEY>

Static green là **điều kiện cần, không đủ**. Sau khi tĩnh sạch, phải chứng minh **hành vi**:

| Đổi ở đâu | Static (cần) | Runtime verify (đủ — bắt buộc thêm) |
|-----------|--------------|--------------------------------------|
| **Flutter UI / VM** | `fvm dart analyze` sạch | Chạy màn hình / widget test cover đúng luồng đổi; nếu UI, mô tả state ChangeState quan sát được (loading→data→empty→error), không chỉ "compile ok" |
| **Model / parsing** | `fvm dart run build_runner build` sinh `.g.dart` không lỗi | Parse payload thật (hoặc test) — field mới không null ngoài ý muốn, enum lạ không nổ `_$…FromJson` |
| **Supabase migration / RLS / trigger** | migration apply được | Chạy query bằng **role của user thật** (không phải service_role) — RLS cho đúng người xem, chặn đúng người; test pgTAP ở `supabase/tests/` xanh; CHECK/FK không bắn 23514 với dữ liệu biên |
| **Edge function (Deno)** | `deno check` sạch | Gọi function với input thật + input biên (null/rỗng/sai quyền); đọc log, xác nhận status/response đúng |
| **Enum / status / vocabulary mới** | tsc/analyzer sạch | [[impact-sweep]]: mọi switch/if/ternary/UI/DB/i18n đã handle case mới ở runtime — grep hết consumer, không để nhánh rơi |
| **i18n / chuỗi hiển thị** | build ok | Bật FR **thật**, xác nhận chuỗi đổi ngôn ngữ (cả literal lẫn vocabulary từ DB/admin) — xem [[bow-i18n]] |
| **Realtime** | analyzer sạch | Đổi row live, xác nhận UI re-sync; dispose không leak channel — xem [[flutter-realtime]], [[express-realtime-review]] |

Nếu không chạy được luồng thật (thiếu thiết bị/DB/quyền), **nói rõ**: "tĩnh đã sạch, nhưng
runtime CHƯA verify vì <lý do> — cần <ai/việc gì> để chứng minh hành vi". Đừng gói nó thành
"đã xong".

## Những câu tự lừa — thấy là DỪNG

| Ý nghĩ | Thực tế |
|--------|---------|
| "Analyzer xanh rồi, chắc chạy được" | Xanh = biên dịch được, không phải chạy đúng. RLS/enum/realtime bug đều xanh. |
| "Fix nhỏ thôi, khỏi chạy lại" | Fix nhỏ nhất cũng đổi hành vi. Chạy lại. |
| "Vừa chạy pass lúc nãy rồi" | Code đã đổi từ lúc đó. Bằng chứng phải MỚI, trong lượt này. |
| "Chắc là ổn" / "Nhìn là biết đúng" | Đoán không phải bằng chứng. Chạy lệnh ra output. |
| "Test viết xong nghĩa là pass" | Viết ≠ chạy. Xem nó chạy và xanh thật. |

## Ăn khớp với hook và bow-commit

Repo monorepo có **Stop hook `self-verify-rubric.sh`** tự bung rubric quest khi lượt kết thúc
mà còn commit chưa push trên nhánh feature — đó là backstop, không thay thế skill này. Skill
này chạy **sớm hơn**: trước cả khi bạn nói "xong". Khi tới bước commit, [[bow-commit]] bước
"4b. self-verify quest checklist" là nơi đóng dấu bằng chứng lần cuối trước push.

Nguyên tắc gọn: **đừng để hook phải nhắc bạn điều lẽ ra bạn tự verify.**
