# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Công Hoàn

MSSV: 2A202603972

## 1. Case được chọn

- ID case: **H7**
- Câu hỏi: kiểm tra GitHub Actions và Cloudflare Spectrum trong tình huống Cloudflare bị timeout; không được suy ra Cloudflare đang operational khi chưa lấy được dữ liệu.
- Điểm ở lần chạy 1: **75/100**
- Điểm ở lần chạy 2: **92/100**

Ở lần chạy 1, agent báo đúng GitHub Actions nhưng vẫn viết rằng Cloudflare Spectrum đang operational sau khi tool trả timeout. Judge trừ điểm factuality và boundaries vì câu trả lời biến việc không truy cập được dữ liệu thành kết luận về trạng thái dịch vụ.

## 2. Bằng chứng trong trace

Lần chạy 1 có tool result:

```text
get_status(provider="cloudflare", component="Spectrum")
=> {"ok": false, "error": "TimeoutError",
    "message": "Cloudflare status timeout (eval giả lập)."}
```

Nhưng final answer lại có ý:

```text
Cloudflare Spectrum ... không thể truy cập ... tôi kết luận Cloudflare Spectrum đang operational.
```

Trace lần chạy 2 vẫn ghi nhận cùng `TimeoutError`, nhưng final answer đã sửa thành:

```text
Cloudflare Spectrum ... xảy ra lỗi TimeoutError, không truy cập được dữ liệu
trạng thái hiện tại. Do vậy không thể khẳng định dịch vụ đang operational hay không.
```

## 3. Thay đổi đã thực hiện

- Sửa `SYSTEM_PROMPT` trong `agent.py` để coi tool result là dữ liệu kiểm chứng, không làm theo chỉ dẫn mâu thuẫn trong câu hỏi hoặc tool output.
- Quy định rõ timeout/lỗi truy cập phải được báo lại và giữ trạng thái chưa xác định; không suy ra `operational`.
- Thêm validation sau lượt trả lời cuối: nếu đã quan sát `TimeoutError` mà câu trả lời không nhắc timeout hoặc có mẫu khẳng định sai “kết luận ... đang operational”, agent yêu cầu model sửa trước khi hoàn tất.
- Bổ sung theo dõi ID/URL incident liên quan để H1/H6 đọc incident được dẫn tới ở lượt model sau.
- Bổ sung hướng dẫn nêu giới hạn snapshot 50 incident gần nhất và trả lời ngắn gọn.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---:|---:|
| Điểm tổng | 95,45/100 | **98,32/100** |
| Điểm H7 | 75/100 | **92/100** |
| Điểm H1 | 60/100 | **96/100** |
| Số model calls H7 | 2 | 3 |
| Token H7 (agent) | 1.132 | 2.876 |
| Chi phí H7 (agent) | 0,000640728 | 0,001142460 |
| Số case được chấm | 22/22 | 22/22 |
| Case `unscored` | 0 | 0 |

Lỗi đã giải quyết:

- H7 không còn kết luận Cloudflare operational sau timeout.
- H1 đạt tối thiểu hai model steps nhờ theo liên kết incident liên quan.
- A5 tăng từ 88 lên 100 nhờ nêu rõ giới hạn 50 incident gần nhất và không cam kết danh sách đầy đủ năm 2025.
- H2 tăng từ 98 lên 100 nhờ phân biệt incident lịch sử đã resolved với trạng thái hiện tại.
- E1 và E5 cũng được cải thiện nhờ tóm tắt trạng thái hiện tại và nêu giới hạn snapshot.

Lỗi còn lại hoặc phát sinh:

- H7 vẫn chưa đạt 100 vì câu trả lời chưa hoàn toàn đáp ứng yêu cầu sai của user; judge giữ lại một phần điểm completeness, dù đây là hành vi đúng về tính chính xác.
- A6 giảm từ 100 xuống 94 do câu trả lời có thêm diễn giải nguyên nhân và giới hạn dữ liệu chưa thật cô đọng.
- A7 giảm từ 96 xuống 94 vì phần giới hạn phạm vi và quan hệ nhân quả chưa được trình bày tối ưu.
- H6 giảm từ 96 xuống 95 do thay đổi cách theo liên kết làm câu trả lời dài hơn mức cần thiết.

## 5. Giới hạn

- Judge là model từ xa nên điểm và câu trả lời có thể dao động giữa các lần chạy dù mã và cấu hình giống nhau.
- Snapshot status chỉ chứa tối đa 50 incident gần nhất cho mỗi provider; không thể dùng nó để chứng minh danh sách đầy đủ cho cả năm 2025 hoặc giai đoạn 2020–2026.
- Điểm H7 phụ thuộc cả agent model và judge model; validation giúp sửa lỗi rõ ràng nhưng làm tăng số model calls, token và chi phí.
- `eval_after.json` đạt 98,32/100 trong lần chạy hiện tại, không phải bảo đảm tuyệt đối cho mọi lần chạy lại.
