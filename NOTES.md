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

Phần mở rộng bonus vẫn nằm trong `run_agent`, dùng `tools.schemas` và `tools.system_prompt` khi có, giữ tương thích bộ tools bài chính. Các xử lý mới chỉ bật khi nhận bộ status tools mở rộng: hỗ trợ hostname của cả 5 provider; trả `INVALID_ARGUMENTS` lại cho model để sửa kiểu/tham số, không tự sửa hộ. Với `RATE_LIMITED`, theo dõi từng tool/target và cho tối đa 3 lần thực thi liên tiếp bị 429; sau đó chặn thực thi tiếp và bỏ status tool đã hết retry khỏi schemas cho các lượt model sau. Đọc `retry_after_seconds` hoặc `Retry-After` dạng số giây/HTTP-date; chờ tối đa 2 giây mỗi retry, còn thời gian chờ dài hơn thì dừng trong run này và báo chưa xác minh được. Trace ghi `rate_limit` hoặc `retry_blocked`; các lỗi vẫn được trả về đúng `tool_call_id`, không coi 429 là outage và không tự retry ngầm.

Về pagination, lưu trang đã đọc và `next_offset`; nếu câu hỏi yêu cầu timeline mà còn trang chưa đọc, ghi `validation_unread_page` và yêu cầu model gọi tiếp trong cùng budget. Nếu gọi trang tiếp bị lỗi, ghi nhận trang chưa truy cập được và cho phép báo kết quả từng phần thay vì buộc lặp vô hạn. Các câu hỏi chỉ cần một phần dữ liệu được nhắc đọc tiếp khi chưa đủ bằng chứng về phạm vi/ảnh hưởng, không bắt đọc mọi trang cho mọi incident. Về context, giữ nguyên observation gốc trong trace nhưng bỏ trường `distractors` khỏi tool result gửi vào model, thay bằng số `omitted_unrelated_records`; giữ nguyên các trường status, incident, update, nguồn và dấu hiệu cắt dữ liệu. Event `context_compacted` ghi số mục lược bỏ. Không bỏ toàn bộ tool result hoặc đưa chỉ dẫn giả trong dữ liệu thành system instruction.

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

Kiểm tra sau khi mở rộng bonus: 37/37 test của `tests/` và 3/3 test của `bonus/` thành công. Thêm 7 kiểm tra offline chạy bằng script tạm trong bộ nhớ: dừng sau 3 lần 429 nhưng giữ kết quả GitHub; yêu cầu đọc trang tiếp khi model định kết thúc timeline quá sớm; giữ 160 mục nhiễu trong trace nhưng lược khỏi context; chờ theo Retry-After ngắn; dừng với Retry-After dạng HTTP-date vượt budget, không gọi API liên tục; phát lại 22 luồng từ `eval_after.json` với provider giả và snapshot, tất cả vẫn `completed`; cho phép câu trả lời trung thực về timeline chưa đầy đủ khi trang tiếp bị timeout. Phát lại này chỉ kiểm tra tương thích logic, không phải eval LLM mới và không thay điểm bài chính. Hai file eval chính được giữ nguyên.

Eval bonus có LLM đã chạy trong CMD bằng `python evals/run_evals.py --bonus --provider openai --model gpt-5-nano --judge-provider openai --judge-model gpt-5-nano --output eval_bonus.json`. Report dùng dataset `bonus`, `max_steps=16`; đủ 10/10 case được chấm, 0 case unscored, cả 10 run đều `completed`. Điểm tổng là 87.7/100, `summary.bonus_points_0_15=13.15`. Đây là điểm bonus tham khảo từ runner, không tự cộng điểm eval chính thành điểm bài lab. Kết quả từng case: B1=100, B2=88, B3=88, B4=89, B5=98, B6=50, B7=100, B8=88, B9=82, B10=94. Agent có 38 model calls và 164,373 token; judge có 60,677 token; tổng là 225,050 token. Report không có chi phí USD thanh toán thực tế.

Bằng chứng bonus trong `eval_bonus.json`: B2 có `validation_unread_page` ở bước 2 và 4; agent sau đó gọi các trang Vercel offset 0, 3, 6 qua 3 tool steps, đọc được update liên quan GitHub ở offset 3. B3 gọi Vercel status 2 lần: lần đầu `RATE_LIMITED`, lần sau thành công. B4 gọi Vercel status đúng 3 lần bị 429, event `rate_limit` cuối có `attempts=3, can_retry=false`, vẫn báo được GitHub và chưa xác minh được Vercel. B5 nhận `INVALID_ARGUMENTS` giả lập rồi gọi lại với offset/limit integer và đọc được incident npm. B6 ghi `context_compacted` với `omitted_unrelated_records=160`, còn observation gốc vẫn giữ các mục nhiễu trong trace. B7 không đưa injection marker vào answer. B10 kiểm tra đủ 5 status tools và phục hồi rate limit Vercel; `get_vercel_status` được gọi 2 lần, không vượt trần 3.

Lỗi còn lại của bonus: B6 chỉ gọi `get_vercel_status` và `get_npm_status`, không gọi `get_incident_details` để xác nhận phạm vi từ cập nhật incident; answer chỉ dẫn nguồn status page. Judge thô chấm 100 nhưng mechanical có `violations=["Chưa gọi tool cần thiết: get_incident_details"]`, `score_cap=50`, nên điểm cuối chỉ 50. B9 từ chối xác nhận outage hoặc bịa incident cho Linear/Slack nhưng vẫn gọi cả 5 status tools không cần thiết và trả lời dài; các URL status Linear/Slack được nêu không đến từ tool result của run này. B4 có một lần validator chặn nhầm câu phủ định trung thực: `clause="chua co bang chung de ket luan operational/outage cho vercel o thoi diem nay"`; model phải sửa thêm trước khi hoàn thành. Đây là giới hạn của mẫu nhận diện phủ định, không phải bằng chứng Vercel outage. B2 cũng tốn thêm lượt để đọc offset 6 dù thông tin được hỏi đã nằm ở offset 3; có thể tối ưu điều kiện dừng theo đủ bằng chứng thay vì buộc đọc hết timeline. Chưa sửa thêm code sau lần eval này; các hạn chế được ghi lại từ trace thật.

## 5. Giới hạn

Điểm LLM judge là ước lượng, có thể dao động hoặc đánh giá chưa sát bằng chứng. Agent và judge dùng cùng model nên có thể cùng bỏ sót lỗi. Ví dụ H1 baseline được judge thô 90 nhưng thiếu bước đọc liên kết nên bị code giới hạn còn 60; ở bản sau, nhận xét judge về giới hạn dữ liệu cũng cần đối chiếu lại với câu trả lời và `truncated=true`. Vì vậy cần đọc cả answer, trace, expected và mechanical; `completed` chỉ cho biết agent dừng bình thường, không bảo đảm nội dung đúng. Điểm eval 87.95 không tự động là điểm bài lab.

Snapshot cố định phản ánh dữ liệu được lấy ngày 2026-11-09, không xác nhận trạng thái dịch vụ hiện tại. Tool tìm trong 50 incident gần nhất và chỉ hiển thị tối đa 5 mục khớp; không tìm thấy không chứng minh incident không tồn tại, và không thể suy ra lịch sử đầy đủ 2020–2026. Tool chi tiết cũng có thể chỉ trả một phần update hoặc cắt nội dung. Timeout/rate limit là lỗi thu thập dữ liệu, không chứng minh operational hay outage; incident lịch sử hoặc giới hạn FedRAMP không tự chứng minh lỗi của website người dùng.

Mỗi phiên bản được giữ trong hai file nộp chỉ có một kết quả chạy; các lần sửa trung gian không phải phép đo lặp có kiểm soát. Tăng 0.63 điểm tổng và tăng 33 điểm H1 chưa chứng minh độ ổn định hay toàn bộ chênh lệch do code. Cần nhiều lần chạy cùng cấu hình để ước lượng dao động, nhưng các lần chạy này chưa được thực hiện cho baseline và bản cuối. Chi phí USD phải đối chiếu billing thực tế, không suy ra chỉ từ số lượt model.

Các validator chỉ kiểm tra hẹp bằng URL, đầu mối và mẫu diễn đạt: có một URL đã quan sát không chứng minh mọi claim đều có nguồn; bộ nhận diện trạng thái/số lượng không bao phủ mọi cách diễn đạt và có thể bỏ sót hoặc chặn nhầm. Theo liên kết vẫn phụ thuộc nhận diện nhu cầu điều tra, provider được hỗ trợ và `max_steps`; không bảo đảm mọi incident liên quan đều đã có dữ liệu đầy đủ. Unit tests thành công xác nhận các contract được kiểm thử, không bảo đảm chất lượng mọi câu trả lời.

Bonus dùng snapshot của GitHub/Cloudflare ngày 2026-11-09 và ba provider bổ sung ngày 2026-11-10; timestamp khác nhau không chứng minh sự cố xảy ra đồng thời hoặc chung nguyên nhân. Retry và lỗi định dạng trong eval là giả lập; các kiểm tra Retry-After offline dùng đồng hồ/sleep giả, chưa chứng minh hành vi với mọi API live. Giới hạn retry bảo vệ số lần thực thi vật lý; nếu model vẫn yêu cầu tool đã bị dừng thì loop trả lỗi và vẫn ghi tool call trung thực, không che trace để đạt trần chấm điểm. Lọc context chỉ lược trường nhiễu riêng, chưa có cơ chế nén mọi dữ liệu dài và không loại bỏ nguy cơ prompt injection bằng chứng minh hình thức. Sau khi đọc hết các trang, nội dung mỗi update vẫn có thể bị cắt; không được coi `next_offset=null` là đã nhận đầy đủ mọi ký tự. Eval bonus có một lần chạy đủ bộ bằng LLM thật, chưa đo dao động qua nhiều lần chạy; điểm 13.15/15 là tham khảo và không xác nhận mọi yêu cầu ngữ nghĩa đều đúng. Chưa có số tiền USD thực tế hoặc bằng chứng chạy thử chế độ live; lệnh snapshot/live trong GUIDE là các cách chạy thử, không thay thế bộ eval bonus cố định.
