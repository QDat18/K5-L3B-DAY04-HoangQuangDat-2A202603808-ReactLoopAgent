# Ghi chú phân tích · Day 04

Họ và tên: Hoàng Quang Đạt
MSSV: 2A202603808

Cấu hình giống nhau ở cả hai lần chạy: provider `openrouter`, model `gpt-4o-mini`, judge cùng provider và model, `max_steps` 8, dataset `core`, snapshot cố định.

## 1. Case được chọn

- ID case: **H1** (điều tra incident GitHub `h02n9572mx9y`, theo liên kết do incident cung cấp rồi đối chiếu với incident Cloudflare `yaa0witbl2me`).
- Câu trả lời của agent (lần 1): agent gọi `get_incident_details` cho GitHub và Cloudflare rồi trả lời ngay. Câu trả lời không đọc incident được trỏ tới (`xr3myzj7ms51`), và nêu ngày giờ sự cố GitHub không có trong kết quả tool ("November 4" cho một sự cố có `updated_at` 07/11).
- Điểm ở lần chạy 1: **60**. Judge cho 75, bị code giới hạn xuống 60 vì vi phạm `mechanical`: "Cần tool calls ở ít nhất 2 model steps khác nhau".

## 2. Bằng chứng trong trace

Trace H1 trong `eval_before.json`:

```text
tool  step 1  get_incident_details  {"provider": "github", "incident_id": "h02n9572mx9y"}
tool  step 1  get_incident_details  {"provider": "cloudflare", "incident_id": "yaa0witbl2me"}
(model trả lời cuối, status = completed)
mechanical: tool_calls = 2, tool_steps = 1, score_cap = 60
```

Update mới nhất của incident GitHub `h02n9572mx9y` trả về:

```text
Please refer to the combined summary in this related incident: https://www.githubstatus.com/incidents/xr3myzj7ms51
```

Nguyên nhân có thể:
- Lỗi nằm ở prompt và loop chứ không phải ở tool. Model gọi hai tool song song ở step 1, thấy đủ dữ liệu nên dừng, và không có chỗ nào trong prompt hay loop buộc nó đọc incident được trỏ tới.
- Câu trả lời có chi tiết không có trong tool result, cho thấy model tự suy diễn khi thiếu bằng chứng.
- Lần thử sau khi chỉ sửa prompt (thêm chỉ dẫn "nếu update trỏ tới incident khác thì gọi tool cho incident đó") vẫn bị giới hạn ở 60: `gpt-4o-mini` bỏ qua chỉ dẫn này, nên cần ràng buộc trong loop.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu (trong `agent.py`):
  1. **System prompt**: thêm chỉ dẫn đọc incident được trỏ tới, chỉ nêu ngày giờ và nguyên nhân có trong tool result, nêu rõ khi dữ liệu `truncated`, không suy ra nguyên nhân chung khi tool không nêu, dẫn URL riêng cho từng incident.
  2. **Kiểm tra nguồn (Bài 4)**: lưu `url` từ `get_incident_details` thành công. Nếu câu trả lời cuối không chứa URL nào thì ghi event `validation_missing_source` và yêu cầu model sửa trong `max_steps`. Hết lượt thì trả `max_steps`, không báo `completed`.
  3. **Kiểm tra tham chiếu chưa đọc**: sau mỗi `get_incident_details`, quét `/incidents/<id>` trong nội dung update. Khi model định trả lời cuối mà còn ID chưa đọc, ghi event `validation_unvisited_reference` và nhắc model gọi tool cho các ID đó, rồi trả lời đầy đủ câu hỏi ban đầu. Mỗi ID chỉ nhắc một lần, và chỉ khi còn lượt.
  4. **Kiểm tra nguồn từng phần**: nếu câu trả lời có một số URL nhưng thiếu URL của incident khác đã đọc, nhắc đúng một lần (event `validation_partial_sources`).
- Vì sao chọn thay đổi này: lỗi của H1 là agent dừng quá sớm và dẫn nguồn thiếu. Cách sửa chỉ bằng prompt đã được thử và không đủ, nên chuyển sang ràng buộc trong loop. Các cơ chế không hard-code ID hay câu trả lời nào, chỉ dựa trên dữ liệu trong tool result. Cơ chế nhắc đúng một lần để tránh vòng lặp vô hạn và tránh đẩy các case khác vào `max_steps`.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 81.82 | 77.59 |
| Điểm case đã chọn (H1) | 60 | 80 |
| Số model calls (22 case) | 51 | 52 |
| Token của agent | 41,076 | 55,026 |
| Chi phí agent (USD, theo OpenRouter) | 0.01009 | 0.01257 |

Trace H1 trong `eval_after.json`:

```text
tool  step 1  get_incident_details  h02n9572mx9y
tool  step 1  get_incident_details  yaa0witbl2me
validation_unvisited_reference  step 2  incident_ids = ["xr3myzj7ms51"]
tool  step 3  get_incident_details  xr3myzj7ms51
mechanical: tool_calls = 3, tool_steps = 2, violations = []
```

- Lỗi đã giải quyết:
  - H1: tăng từ 60 lên 80. Agent đọc incident được trỏ tới, vi phạm `mechanical` biến mất (`tool_steps` từ 1 lên 2). Đây là thay đổi có bằng chứng trực tiếp trong trace.
  - Một số case khác tăng điểm: E6 +25, E2, E4, A2 mỗi case +20, E3 +10, H6 +7. Chưa kiểm tra được có liên quan đến thay đổi hay không, vì event `validation_*` chỉ xuất hiện ở H1.
- Lỗi còn lại hoặc mới xuất hiện: điểm tổng **giảm 4.23**. Các case giảm: H2 −50, H5 −45, A1 −25, A5 −20, H4 −20, H7 −20, H8 −15, E1 −10, A3 −5, A8 −5.
  - H2: lần "trước" gọi `get_incident_details` rồi `get_status`; lần "sau" chỉ gọi `get_incident_details`, nên vi phạm `mechanical` "Chưa gọi tool cần thiết: get_status". Không có event `validation_*` nào ở case này.
  - H5: lần "sau" gọi thêm 3 incident không cần thiết (`xr3myzj7ms51`, `smtdswd1h2w4`, `pfbmpxxfq1cy`) mà không có event `validation_*`, và judge ghi mô tả lịch sử sai về thời gian và mức độ ảnh hưởng.
  - Vì các cơ chế kiểm tra chỉ kích hoạt ở H1, phần lớn mức giảm không truy được về một thay đổi cụ thể. Khả năng cao là biến động của model và/hoặc ảnh hưởng của prompt dài hơn, chưa kiểm chứng được.
  - Chi phí và token tăng khoảng 34% (token) và 25% (chi phí).

## 5. Giới hạn

- **Điểm judge là ước lượng.** Judge dùng cùng provider và model (`gpt-4o-mini`) với agent nên có thể thiên lệch. Có đối chiếu trace và `mechanical` cho H1, H2, H5, nhưng chưa kiểm tra lại toàn bộ 22 case bằng tay.
- **Một lần chạy mỗi phiên bản không chứng minh độ ổn định.** Điểm H1 dao động giữa các lần chạy riêng lẻ với cùng code (60, 75, 100, rồi 80 trong lần chạy cả bộ), cho thấy nhiễu có thể lớn bằng chính mức cải thiện đang đo. Mức giảm ở H2 và H5 có thể là nhiễu, chưa chạy lặp để xác nhận.
- **Nhiều thay đổi gộp chung.** Bản "sau" gồm bốn thay đổi (prompt, kiểm tra nguồn, kiểm tra tham chiếu, kiểm tra nguồn từng phần), nên không tách được tác dụng của từng thay đổi lên điểm tổng. Bản "trước" cũng không có khối kiểm tra nguồn.
- **Snapshot cố định** (09/11/2026), nên kết quả không phản ánh tình trạng thực tế hiện tại và không kiểm tra được hành vi với API live (rate limit, timeout thật).
- **Trace không ghi event `model` mỗi bước**, nên số model calls ở mục 4 được tính từ độ dài danh sách `usage` của từng case.
- **Kiểm tra nguồn chỉ kiểm tra sự hiện diện của URL**, không chứng minh mọi claim đều có nguồn. Cơ chế tham chiếu chỉ nhận dạng liên kết dạng `/incidents/<id>`.
