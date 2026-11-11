# Ghi chú phân tích · Day 04

Họ và tên: Tran Duong Trung Tinh
MSSV: 2A202603488

## 1. Case được chọn

- ID case: H1 — điều tra GitHub `h02n9572mx9y`, đọc incident liên quan và đối chiếu Cloudflare `yaa0witbl2me`.
- Câu trả lời của agent: Trích từ `eval_before.json`: "GitHub h02n9572mx9y: một sự cố liên quan Actions, đã được đánh dấu là resolved; nguyên nhân gốc chưa được công bố trong các cập nhật hiện tại; liên kết tổng quan nguyên nhân có ở một incident liên quan." Agent còn hỏi: "Bạn có muốn mình lấy chi tiết thêm từ:" rồi đề xuất đọc incident `xr3myzj7ms51`, dù câu hỏi đã yêu cầu theo liên kết để điều tra nguyên nhân và ảnh hưởng.
- Điểm ở lần chạy 1: 60/100. Điểm judge thô là 90, nhưng kiểm tra bằng code áp trần 60 vì tool calls chỉ nằm trong một model step.

## 2. Bằng chứng trong trace

Các trích đoạn dưới đây lấy từ case H1 trong hai file eval; các trường tool result được chọn lọc, không phải toàn bộ trace.

```text
eval_before.json — result.trace:
{"event": "model", "step": 1, "finish": "tool_calls", "tool_calls": 2}
Tool: get_incident_details({"provider": "github", "incident_id": "h02n9572mx9y"})
result.updates[0].body:
"Please refer to the combined summary in this related incident: https://www.githubstatus.com/incidents/xr3myzj7ms51"
Tool: get_incident_details({"provider": "cloudflare", "incident_id": "yaa0witbl2me"})
{"event": "model", "step": 2, "finish": "stop", "tool_calls": 0}

grade.mechanical:
{"violations": ["Cần tool calls ở ít nhất 2 model steps khác nhau."], "score_cap": 60, "tool_calls": 2, "tool_steps": 1}

eval_after.json — result.trace:
{"event": "model", "step": 1, "finish": "tool_calls", "tool_calls": 2}
Tool: get_incident_details({"provider": "github", "incident_id": "h02n9572mx9y"})
Tool: get_incident_details({"provider": "cloudflare", "incident_id": "yaa0witbl2me"})
{"event": "model", "step": 2, "finish": "tool_calls", "tool_calls": 1}
Tool: get_incident_details({"provider": "github", "incident_id": "xr3myzj7ms51"})
result.ok: true
result.components: ["Webhooks", "Actions", "Pages"]
result.updates[0].truncated: true
result.updates_total: 9
result.url: "https://stspg.io/pjgptg3cfc6n"
{"event": "model", "step": 3, "finish": "stop", "tool_calls": 0}

grade.mechanical:
{"violations": [], "score_cap": 100, "tool_calls": 3, "tool_steps": 2}
```

Nguyên nhân có thể xảy ra: loop ban đầu cho phép kết thúc ngay khi model trả final answer, chưa theo dõi đầu mối incident liên quan còn chưa đọc. Model đã coi liên kết là thông tin để giới thiệu cho người dùng thay vì bước điều tra cần thực hiện. Hai tool calls ở bước 1 chỉ đọc hai incident ban đầu; chưa có bằng chứng từ incident tổng hợp để kết luận về nguyên nhân GitHub. Đây là lỗi thiếu bước thu thập bằng chứng, không phải lỗi API hay lý do để chỉ tăng số bước.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: Bổ sung trong `run_agent` của `agent.py`, giữ nguyên chữ ký hàm và template. Nhắc model đọc tiếp incident liên quan khi câu hỏi yêu cầu điều tra nguyên nhân/ảnh hưởng. Hàm `record_evidence` lưu incident đã gọi, URL nguồn từ tool chi tiết thành công và các incident liên quan chưa đọc; chỉ nhận liên kết có hostname thuộc provider đã đăng ký và đường dẫn `/incidents/<id>`. Đưa các đầu mối còn thiếu vào ngữ cảnh để model gọi tiếp tool. Nếu model định kết thúc khi còn đầu mối chưa đọc, ghi `validation_unread_incident` và yêu cầu sửa trong `max_steps`; không tự tải nội dung URL ngoài tools.
- Vì sao chọn thay đổi này: H1 có liên kết rõ ràng ngay trong observation nhưng baseline chưa đọc. Theo dõi đầu mối giúp thu thập bằng chứng thực sự ở bước sau, không hard-code ID, đáp án hay thay đổi cách chấm. Sau cải tiến, H1 đã đọc `xr3myzj7ms51` và dùng nội dung về thay đổi cấu hình dịch vụ định tuyến sự kiện để giải thích nguyên nhân GitHub, đồng thời không suy ra nguyên nhân chung với Cloudflare.

Các kiểm tra bổ sung trong cùng hàm: nếu đã có incident chi tiết thành công nhưng câu trả lời thiếu mọi URL incident đã quan sát, ghi `validation_missing_source` và yêu cầu sửa; không bắt URL khi không có incident chi tiết thành công. Theo dõi status tool thất bại và số incident thực sự trả về để phát hiện một số khẳng định trạng thái chưa được xác minh (`validation_unverified_status`) hoặc nhầm số mục trả về với cửa sổ tìm kiếm (`validation_incident_count`). Gộp lỗi vào một lượt yêu cầu sửa và tính các lượt sửa trong cùng `max_steps`; hết budget trả `max_steps`, không trả `completed` cho câu trả lời bị chặn. Bộ kiểm tra `python -m unittest discover -s tests -v` đã chạy lại: 37/37 test thành công, bao gồm contract kiểm tra nguồn.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 87.32/100; 22/22 case được chấm | 87.95/100; 22/22 case được chấm; tăng 0.63 điểm |
| Điểm case đã chọn | H1: 60/100 | H1: 93/100; tăng 33 điểm |
| Số model calls | Agent toàn bộ 22 case: 53; riêng H1: 2 | Agent toàn bộ 22 case: 53; riêng H1: 3 |
| Token, chi phí nếu có | Agent: 126,790 token; judge: 96,819; tổng: 223,609. Không có số tiền USD thực tế trong report. | Agent: 170,003 token; judge: 89,452; tổng: 259,455. Không có số tiền USD thực tế trong report. |

Hai file cùng cấu hình: agent và judge đều dùng `openai` / `gpt-5-nano`, `max_steps=8`, dataset `core`; đối chiếu các observation cho thấy dùng cùng snapshot cố định ngày 2026-11-09. Không có case `unscored` và cả 22 case của mỗi file đều có `result.status=completed`. Số model calls trong bảng đếm event `model` của agent, không cộng lượt judge. Token là tổng `usage.total_tokens` của agent và `grade.judge_usage.total_tokens` của judge, có tính cả reasoning token theo usage API trả về; đây không phải số tiền thanh toán. Tổng lượt agent không đổi nhưng token agent tăng 43,213, khoảng 34.1%, nên chưa thể kết luận cải tiến tiết kiệm chi phí.

Trong quá trình sửa có lần eval trung gian đạt 84.36/100; sau khi phân tích lỗi đã điều chỉnh thêm. `eval_after.json` được chạy lại và ghi đè theo tên file nộp bài, hiện chứa kết quả cuối 87.95/100. Bảng so sánh dùng baseline gốc và bản cuối; không coi đây là toàn bộ lịch sử thử nghiệm hay bằng chứng về hai lần chạy độc lập duy nhất.

- Lỗi đã giải quyết: H1 đã thực sự đọc incident liên quan ở bước 2, từ 2 tool calls/1 tool step thành 3 tool calls/2 tool steps, bỏ được trần 60 điểm. H3 tăng 50 lên 89, trace sau cải tiến có gọi trực tiếp chi tiết ID không tồn tại và nhận `INCIDENT_NOT_FOUND`. Ở H7, trace sau cải tiến ghi `validation_unverified_status` tại bước 2 khi model định kết luận Cloudflare Spectrum operational theo yêu cầu người dùng; câu trả lời cuối đã nêu chưa xác minh được do timeout. H8 đã phân biệt 5 mục trả về mỗi provider với 50 mục khớp trong cửa sổ, không nhận đã liệt kê toàn bộ giai đoạn 2020–2026; tuy nhiên điểm H8 vẫn giảm 89 xuống 76.
- Lỗi còn lại hoặc mới xuất hiện: E2 giảm 92 xuống 62; agent suy đoán ảnh hưởng tới website không thuộc FedRAMP từ các incident khu vực đã resolved, chưa đọc chi tiết incident đang xét, nên kết luận vượt bằng chứng về phạm vi và thời gian. H4 giảm 80 xuống 65; agent không gọi tool cho yêu cầu làm thơ nhưng trả lời còn dài. Riêng H1, tool của incident liên quan chỉ trả 3/9 update và có `truncated=true`; câu trả lời sau cải tiến vẫn chưa nêu rõ giới hạn cụ thể này, dù judge chấm cao. Không khẳng định mọi lỗi đã được giải quyết hoặc mọi case đều cải thiện.

## 5. Giới hạn

Điểm LLM judge là ước lượng, có thể dao động hoặc đánh giá chưa sát bằng chứng. Agent và judge dùng cùng model nên có thể cùng bỏ sót lỗi. Ví dụ H1 baseline được judge thô 90 nhưng thiếu bước đọc liên kết nên bị code giới hạn còn 60; ở bản sau, nhận xét judge về giới hạn dữ liệu cũng cần đối chiếu lại với câu trả lời và `truncated=true`. Vì vậy cần đọc cả answer, trace, expected và mechanical; `completed` chỉ cho biết agent dừng bình thường, không bảo đảm nội dung đúng. Điểm eval 87.95 không tự động là điểm bài lab.

Snapshot cố định phản ánh dữ liệu được lấy ngày 2026-11-09, không xác nhận trạng thái dịch vụ hiện tại. Tool tìm trong 50 incident gần nhất và chỉ hiển thị tối đa 5 mục khớp; không tìm thấy không chứng minh incident không tồn tại, và không thể suy ra lịch sử đầy đủ 2020–2026. Tool chi tiết cũng có thể chỉ trả một phần update hoặc cắt nội dung. Timeout/rate limit là lỗi thu thập dữ liệu, không chứng minh operational hay outage; incident lịch sử hoặc giới hạn FedRAMP không tự chứng minh lỗi của website người dùng.

Mỗi phiên bản được giữ trong hai file nộp chỉ có một kết quả chạy; các lần sửa trung gian không phải phép đo lặp có kiểm soát. Tăng 0.63 điểm tổng và tăng 33 điểm H1 chưa chứng minh độ ổn định hay toàn bộ chênh lệch do code. Cần nhiều lần chạy cùng cấu hình để ước lượng dao động, nhưng các lần chạy này chưa được thực hiện cho baseline và bản cuối. Chi phí USD phải đối chiếu billing thực tế, không suy ra chỉ từ số lượt model.

Các validator chỉ kiểm tra hẹp bằng URL, đầu mối và mẫu diễn đạt: có một URL đã quan sát không chứng minh mọi claim đều có nguồn; bộ nhận diện trạng thái/số lượng không bao phủ mọi cách diễn đạt và có thể bỏ sót hoặc chặn nhầm. Theo liên kết vẫn phụ thuộc nhận diện nhu cầu điều tra, provider được hỗ trợ và `max_steps`; không bảo đảm mọi incident liên quan đều đã có dữ liệu đầy đủ. Unit tests thành công xác nhận các contract được kiểm thử, không bảo đảm chất lượng mọi câu trả lời. Bài này hoàn thành phần chính; chưa có `eval_bonus.json` và không báo điểm bonus.
