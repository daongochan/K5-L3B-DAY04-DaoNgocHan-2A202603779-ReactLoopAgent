# Ghi chú phân tích · Day 04

Họ và tên: Đào Ngọc Hân
MSSV: 2A202603779

## 1. Case được chọn

- ID case: A6
- Câu trả lời của agent:
  Agent trả lời incident h02n9572mx9y liên quan đến Actions đã được giải quyết, bản cập nhật lúc 18:16 UTC dẫn sang incident xr3myzj7ms51. Ở phần xr3myzj7ms51, agent chỉ nêu thời gian ảnh hưởng từ 15:49 UTC đến 16:48 UTC (bỏ sót khoảng 17:20 UTC đến 17:55 UTC và độ trễ 25 phút), chưa nêu mốc thời gian chụp dữ liệu snapshot và chưa làm rõ ranh giới giữa thông tin đã xác nhận từ nguồn với phần chưa được xác nhận.
- Điểm ở lần chạy 1: 70/100

## 2. Bằng chứng trong trace

```text
  {
    "event": "tool",
    "step": 1,
    "name": "get_incident_details",
    "arguments": {
      "provider": "github",
      "incident_id": "h02n9572mx9y"
    },
    "result": {
      "ok": true,
      "provider": "github",
      "captured_at": "2026-11-09T03:50:47.065020+00:00",
      "id": "h02n9572mx9y",
      "name": "Incident with Actions",
      "status": "resolved",
      "updates": [
        {
          "status": "resolved",
          "created_at": "2026-11-04T18:16:08.361Z",
          "body": "Please refer to the combined summary in this related incident: https://www.githubstatus.com/incidents/xr3myzj7ms51",
          "truncated": false
        }
      ],
      "updates_total": 4,
      "url": "https://stspg.io/7pwv28hvkmz8"
    }
  },
  {
    "event": "tool",
    "step": 2,
    "name": "get_incident_details",
    "arguments": {
      "provider": "github",
      "incident_id": "xr3myzj7ms51"
    },
    "result": {
      "ok": true,
      "id": "xr3myzj7ms51",
      "name": "Incident with Actions and Pages",
      "status": "resolved",
      "updates": [
        {
          "status": "resolved",
          "created_at": "2026-11-04T17:01:11.503Z",
          "body": "On November 4, 2026, between 15:49 UTC and 16:48 UTC, and again between 17:20 UTC and 17:55 UTC, GitHub Actions, Pages, Webhooks, Code Scanning, and Copilot coding agent experienced delays...",
          "truncated": true
        }
      ],
      "updates_total": 4,
      "url": "https://stspg.io/pjgptg3cfc6n"
    }
  }
```

Nguyên nhân:
- System prompt ban đầu viết khá chung chung, chưa nhắc agent phải luôn nêu mốc `captured_at` của snapshot, chưa yêu cầu tách rõ phần thông tin đã xác nhận và chưa xác nhận khi đọc incident gộp, cũng như quên báo cho người dùng khi update bị cắt (`truncated: true`, `updates_total: 4` nhưng chỉ trả 3 updates).
- Ở một số case tương tự có link sang incident khác (như H1, H6) hoặc gặp lỗi timeout (E6), prompt chưa hướng dẫn kỹ nên agent dừng sớm hoặc trả lời còn chung chung.
- Ngoài ra, trong `run_agent` chưa có bước kiểm tra URL nguồn từ `get_incident_details` trước khi kết thúc câu trả lời.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu:
  - Trong hàm `run_agent` (`agent.py`): Tạo danh sách `incident_urls` lưu lại trường `url` mỗi khi `get_incident_details` trả về `ok: True`. Khi model dừng ở `finish == 'stop'`, nếu `incident_urls` không rỗng mà câu trả lời chưa chứa URL nào trong đó thì ghi event `validation_missing_source` vào `trace`, thêm message nhắc model bổ sung URL nguồn rồi chạy bước tiếp theo.
  - Trong `SYSTEM_PROMPT` (`agent.py`): Viết rõ các yêu cầu khi trả lời: theo dõi tiếp khi incident dẫn link sang ID khác, luôn nêu thời điểm `captured_at`, phân biệt phạm vi khách hàng (FedRAMP / non-FedRAMP) và địa lý (Phoenix / Việt Nam), nêu rõ giới hạn khi dữ liệu bị cắt (`truncated`) hoặc khi gặp `TimeoutError`.
- Vì sao chọn thay đổi này:
  - Kiểm tra `validation_missing_source` trong vòng lặp giúp bắt lỗi quên dẫn link nguồn của incident và pass đủ bộ test trong `tests/test_validation.py`.
  - Chỉnh lại `SYSTEM_PROMPT` giúp model bám sát dữ liệu trong trace hơn ở cả A6 và các case khác mà không phải hard-code bất kỳ thông tin sự cố nào.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 80.68 / 100 | 84.55 / 100 |
| Điểm case đã chọn (A6) | 70 / 100 | 100 / 100 |
| Số model calls | 50 | 49 |
| Token, chi phí nếu có | 41,193 tokens (~$0.0104) | 65,224 tokens (~$0.0137) |

- Lỗi đã giải quyết:
  - Case A6 tăng từ 70 lên 100 điểm: đã nêu rõ `captured_at`, lấy đủ cả hai khung giờ sự cố trong `xr3myzj7ms51` và tách riêng mục thông tin đã xác nhận / chưa xác nhận.
  - Số case đạt 100 điểm tăng từ 6 lên 9 cases. Nhiều case khác cũng tăng điểm rõ rệt nhờ nêu đúng mốc snapshot, phạm vi và giới hạn dữ liệu: E6 (65 → 100), E1 (65 → 95), E2 (75 → 100), A8 (80 → 100), H4 (80 → 100), A5 (65 → 80), H5 (90 → 100).
- Lỗi còn lại hoặc mới xuất hiện:
  - H1 và H6 vẫn giữ ở 60 điểm: do ở bước 1 model đã gọi 2 tools cùng lúc nên sang bước 2 model trả lời luôn mà không gọi thêm bước nữa để đọc `xr3myzj7ms51` (bị giới hạn điểm `min_tool_steps: 2`).
  - H2 giảm từ 100 xuống 50 điểm: ở lần chạy 1 model gọi cả `get_incident_details` và `get_status`, nhưng ở lần chạy 2 model chỉ gọi `get_incident_details` nên bị trừ điểm thiếu tool `get_status`. Bên cạnh đó, ở A1 model tra cứu chi tiết nhiều incident hơn nên tổng số token tăng từ 41k lên 65k.

## 5. Giới hạn

- Điểm Judge: Do dùng LLM (`openai/gpt-4o-mini`) tự chấm nên điểm số chỉ mang tính ước lượng, phụ thuộc nhiều vào cách diễn đạt và có thể chênh lệch giữa các lần chấm. Trạng thái `completed` chỉ cho biết vòng lặp dừng bình thường chứ không đảm bảo câu trả lời đúng; bước kiểm tra `validation_missing_source` cũng chỉ kiểm tra có mặt chuỗi URL chứ không xác minh được nội dung câu trả lời.
- Snapshot và Tools: Dữ liệu snapshot được lưu tĩnh từ ngày 09/11/2026 nên không phản ánh trạng thái hiện tại ngoài thực tế. Công cụ `get_status` chỉ tìm theo chuỗi ký tự con trong 50 incidents gần nhất (trả tối đa 5 kết quả), còn `get_incident_details` chỉ giữ 2 update mới nhất và 1 update đầu tiên (tối đa 900 ký tự mỗi update), vì vậy tìm không thấy không đồng nghĩa với việc không có sự cố.
- Chỉ chạy một lần mỗi phiên bản: Kết quả sinh văn bản và quyết định chọn tool của LLM không hoàn toàn cố định giữa các lần gọi (như trường hợp H2 ở lần 2 bỏ qua `get_status`). Việc chỉ chạy 1 lần cho bản trước và 1 lần cho bản sau chưa đủ để đánh giá độ ổn định lâu dài của agent.

## 6. Ghi chú phần Bonus (`eval_bonus.json`)

- Kết quả: Chấm đủ 10/10 cases, điểm trung bình 76.5 / 100, quy đổi đạt 11.47 / 15 điểm bonus (B2: 100, B6: 100, B8: 95, B4: 80, B9: 80, B3: 75, B5: 75, B7: 60, B1: 50, B10: 50).
- Xử lý chính và giới hạn:
  - Retry và Rate limit: Đếm số lần gọi mỗi tool qua `tool_counts`, chờ theo `retry_after` khi gặp lỗi 429 và chặn tối đa 3 lần gọi (`max_calls = 3`). Ở B3 agent gọi lại lần 2 thành công; ở B4 sau 3 lần bị 429 liên tiếp agent dừng gọi và báo kết quả từng phần; ở B5 agent tự sửa lại tham số `offset=0, limit=2` kiểu integer khi gặp `INVALID_ARGUMENTS`.
  - Pagination và lọc nhiễu: Ở B2 (100đ), agent đọc tiếp các trang qua `next_offset` (`offset=0 -> 2 -> 4`) để tìm đúng update nhắc tới GitHub. Ở B6 (100đ), agent giữ nguyên `result` trong `trace` nhưng bỏ mảng 160 mục `distractors` khỏi message gửi cho model để tránh nhiễu.
  - Giới hạn còn lại: Ở B10, khi gọi 5 status tools cùng lúc ở bước 1 và chỉ có Vercel bị 429, model tổng hợp luôn kết quả của 4 bên kia ở bước 2 mà chưa gọi lại lần 2 cho Vercel.
