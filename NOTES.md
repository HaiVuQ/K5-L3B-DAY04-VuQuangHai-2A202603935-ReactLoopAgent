# Ghi chú phân tích · Day 04

Họ và tên: Vũ Quang Hải
MSSV: 2A202603935

## 1. Case được chọn

- ID case: H4
- Yêu cầu: viết thơ về biển và không kiểm tra trạng thái GitHub/Cloudflare.
- Câu trả lời lần 1: có bài thơ, nhưng agent vẫn gọi `get_status` cho cả GitHub và Cloudflare rồi gọi `get_incident_details` với ID rỗng.
- Điểm lần 1: 40/100.

## 2. Bằng chứng trong trace

Lần 1, trace có bốn tool event ở step 1:

```text
get_status(provider=github)
get_status(provider=cloudflare)
get_incident_details(provider=github, incident_id="")
get_incident_details(provider=cloudflare, incident_id="")
```

Hai lần gọi chi tiết trả `INCIDENT_NOT_FOUND`. Judge ghi nhận yêu cầu không cần tra status nhưng agent vẫn sử dụng tools.

## 3. Thay đổi đã thực hiện

- Bổ sung vào `SYSTEM_PROMPT` trong `agent.py` chỉ dẫn không gọi tools cho yêu cầu ngoài phạm vi status/incident.
- Mục tiêu là giảm gọi tool không cần thiết, không phải ép agent bịa hoặc tự tạo câu trả lời status.
- Lần chạy sau H4 vẫn gọi `get_status` một lần, vì vậy cải tiến chưa loại bỏ hoàn toàn lỗi.

## 4. So sánh hai lần chạy

Cả hai full eval dùng Ollama qua OpenAI-compatible endpoint, cùng model agent/judge `llama3.2:3b`, dataset core và `max_steps=8`.

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---:|---:|
| Điểm tổng | 60.0/100 (22/22 case) | Chưa có; 20/22 case được chấm |
| Trung bình phần đã chấm | — | 58.7/100 (tham khảo, không phải điểm tổng) |
| H4 | 40/100 | 50/100 |
| Số agent model calls | 43 | 47 |
| Agent tokens được báo cáo | 26,868 | 40,343 |
| Chi phí | Không có số liệu; model chạy local | Không có số liệu; model chạy local |

- H4 giảm từ bốn tool events xuống một, nhưng vẫn vi phạm yêu cầu không dùng tool. Điểm H4 tối đa bị mechanical check giới hạn 50 do `forbid_all_tools`.
- `eval_after.json` có A8 và H1 `unscored` do judge timeout; do đó điểm tổng là `null` và không thể kết luận điểm toàn bộ tăng/giảm so với lần 1.
- Chạy lại riêng H1 cũng chưa cho điểm: trong lần retry agent gặp `TimeoutError` khi gọi Ollama. Tăng timeout local lên 1800 giây vẫn không giải quyết ổn định; timeout trong code được giữ ở 600 giây để tránh các request local treo quá lâu.

## 5. Giới hạn

- Judge LLM chỉ là ước lượng. Kết quả phụ thuộc vào model và có thể dao động giữa các lần chạy; H4 vẫn cho thấy agent chưa tuân thủ đầy đủ yêu cầu không gọi tool.
- Snapshot phản ánh dữ liệu tại thời điểm chụp, không phải trạng thái hiện tại ngoài thực tế.
- Mỗi phiên bản chỉ chạy một full eval. Ngoài ra, phiên bản sau có case chưa chấm được nên không có phép so sánh điểm tổng hợp hợp lệ.
- Ollama local bị chậm/timeout trên các request dài; cần chạy lại eval với judge/provider ổn định hơn để có đủ 22/22 case cho lần sau.
