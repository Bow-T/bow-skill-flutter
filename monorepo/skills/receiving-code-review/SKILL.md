---
name: receiving-code-review
description: Khi nhận feedback review (từ người, từ /code-review, từ du-quest AI reviewer, hay từ comment MR/PR) — đánh giá KỸ THUẬT trước, đừng diễn cảm xúc. Verify từng góp ý với code thật TRƯỚC khi sửa; góp ý sai/không hợp codebase thì phản biện có lý lẽ, không implement bừa. CẤM "You're absolutely right!" / "Chuẩn luôn!" / "Để em sửa ngay" khi chưa kiểm chứng. Dùng khi ai đó review code của bạn, khi có comment trên MR, khi /code-review hoặc du-quest trả findings, hoặc khi bạn định "gật" theo một đề xuất. Bổ trợ [[bow-lean-review]], [[impact-sweep]] và [[verify-runtime-not-just-static-green]] (sửa xong phải verify lại).
---

# receiving-code-review — Nhận review bằng lý lẽ, không bằng cảm xúc

## Vì sao có skill này

Nhận review đòi hỏi **đánh giá kỹ thuật**, không phải trình diễn sự đồng thuận. Gật đầu ngay
("Chuẩn luôn!", "Để em sửa liền") trước khi kiểm chứng dẫn tới sửa sai, sửa thừa, hoặc phá
một quyết định vốn cố ý. Trong monorepo <PROJECT_KEY>, feedback đến từ nhiều nguồn — người
review MR, `/code-review`, du-quest AI reviewer chấm điểm, comment QC — và **không phải góp ý
nào cũng đúng với codebase này**. Nhiệm vụ của bạn là lọc, không phải chép.

**Nguyên tắc cốt lõi:** Kiểm chứng trước khi sửa. Hỏi trước khi giả định. Đúng kỹ thuật hơn
là dễ chịu xã giao.

## Mẫu phản hồi

```
KHI nhận một góp ý review:

1. ĐỌC   : Đọc hết góp ý, chưa phản ứng vội
2. HIỂU  : Diễn đạt lại yêu cầu bằng lời của mình (hoặc hỏi nếu chưa rõ)
3. VERIFY: Đối chiếu với code thật — mở đúng file/dòng, xem context
4. ĐÁNH GIÁ: Có đúng kỹ thuật CHO codebase NÀY không? (không phải "nói chung")
5. TRẢ LỜI: Ghi nhận kỹ thuật, hoặc phản biện có lý lẽ nếu sai
6. SỬA   : Từng mục một, verify lại từng mục ([[verify-runtime-not-just-static-green]])
```

## Câu CẤM nói

**KHÔNG BAO GIỜ:**
- "You're absolutely right!" / "Chuẩn luôn anh!" / "Đúng rồi ạ!" (nịnh, phản xạ)
- "Góp ý hay quá!" / "Cảm ơn feedback tuyệt vời!" (diễn)
- "Để em sửa ngay" (khi CHƯA verify góp ý đúng chưa)

**THAY VÀO ĐÓ:**
- Diễn đạt lại yêu cầu kỹ thuật cho chính xác
- Hỏi lại nếu góp ý mơ hồ
- Phản biện bằng lý lẽ kỹ thuật nếu góp ý sai với codebase
- Hoặc chỉ bắt tay làm luôn (hành động > lời nói)

## Xử lý góp ý không rõ

Đừng đoán ý rồi sửa. Hỏi đúng trọng tâm:
> "Góp ý này ý là <A> hay <B>? Vì ở [file:line] hiện đang <hành vi thực>, nên hướng sửa
> sẽ khác nhau."

## Khi góp ý SAI hoặc không hợp codebase

Có quyền — và có trách nhiệm — phản biện. Nhưng phải **bám bằng chứng**, không phải "em thấy
vậy":

- Mở file/dòng thật, trích context chứng minh vì sao đề xuất không áp dụng được.
- Ví dụ đặc thù <PROJECT_KEY>: reviewer bảo "bỏ `removeChannel` cho gọn" → phản biện, vì
  [[flutter-realtime]] yêu cầu dispose channel để không leak; bỏ đi là tái tạo bug leak.
- Ví dụ khác: reviewer bảo "parse `map['key']` trực tiếp cho nhanh" → phản biện, vì
  [[bow-model]] cấm index map thô, mọi model đi qua `@JsonSerializable`.
- Nếu vẫn bất đồng sau khi trình bằng chứng → nêu trade-off ngắn gọn, để người quyết, không
  cãi vòng vo.

## Khi góp ý ĐÚNG

Ghi nhận gọn bằng hành động, rồi:
1. Sửa **từng mục một** — không gộp nhiều fix vào một mớ khó review lại.
2. Với thay đổi lan rộng (đổi signature/enum/shared): chạy [[impact-sweep]] để không sửa sót
   consumer.
3. Sửa xong **verify lại hành vi** ([[verify-runtime-not-just-static-green]]) — không tuyên bố
   "đã fix theo review" khi chưa chạy.

## Riêng với du-quest AI reviewer

du-quest chấm điểm theo rubric (Codebase Impact, security…). Findings của nó thường đúng
hướng nhưng **vẫn phải verify** — nó không thấy được runtime context. Với finding về
security/RLS → chuyển sang [[supabase-security-review]] hoặc [[security-and-hardening]] để xử
đúng tầng, đừng vá hời hợt cho qua điểm.
