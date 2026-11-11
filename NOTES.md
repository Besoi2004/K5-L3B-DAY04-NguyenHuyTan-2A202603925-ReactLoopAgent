# Ghi chú phân tích · Day 04

Họ và tên: Nguyen Huy Tan
MSSV:2A202603925


## 1. Case được chọn

- ID case: **H1**. Câu hỏi yêu cầu điều tra incident GitHub `h02n9572mx9y`, *theo liên kết do incident cung cấp* để tìm nguyên nhân, rồi đối chiếu với incident Cloudflare `yaa0witbl2me`.
- Câu trả lời của agent: tóm tắt đúng `h02n9572mx9y` (Actions, resolved) và `yaa0witbl2me` (Workers Builds, resolved), có dẫn nguồn hai bên. Agent **có trích lại** câu "Please refer to the combined summary in this related incident: …/xr3myzj7ms51" nhưng **không mở** incident `xr3myzj7ms51`, nên không trả lời được phần nguyên nhân mà câu hỏi yêu cầu.
- Điểm ở lần chạy 1: **60/100** (judge 80 = 35+20+15+10, bị code cap xuống 60). Cấu hình: openai / gpt-4o-mini, judge gpt-4o-mini, max_steps 8, dataset core. Điểm tổng lần 1: **78.41**.

## 2. Bằng chứng trong trace

Trích từ `eval_before.json`, case H1:

```text
{'event': 'model', 'step': 1, 'finish': 'tool_calls', 'tool_calls': 2}
{'event': 'tool', 'step': 1, 'name': 'get_incident_details', 'arguments': '{"provider": "github", "incident_id": "h02n9572mx9y"}',
   'result': {... 'updates': [{'body': 'Please refer to the combined summary in this related incident: https://www.githubstatus.com/incidents/xr3myzj7ms51'}, ...]}}
{'event': 'tool', 'step': 1, 'name': 'get_incident_details', 'arguments': '{"provider": "cloudflare", "incident_id": "yaa0witbl2me"}', ...}
{'event': 'model', 'step': 2, 'finish': 'stop', 'tool_calls': 0}      <- dừng luôn, không gọi xr3myzj7ms51
{'event': 'final', 'step': 2}

grade.mechanical: {'violations': ['Cần tool calls ở ít nhất 2 model steps khác nhau.'], 'score_cap': 60, 'tool_calls': 2, 'tool_steps': 1}
```

- Lỗi: tool result ở step 1 đã chỉ sang incident `xr3myzj7ms51`, nhưng ở step 2 model trả lời luôn (`finish: stop`). Mọi tool call đều nằm trong 1 step nên bị cap 60.
- Nguyên nhân có thể: lỗi nằm ở **prompt**, không phải loop hay dữ liệu. Loop đã trả đủ tool result với đúng `tool_call_id`, và dữ liệu có link. Nhưng system prompt không nói gì về việc phải đi theo incident được dẫn link, nên gpt-4o-mini dừng ở incident đầu tiên.
- Cùng kiểu lỗi trong các case khác: **H6** (60, cũng không mở `xr3myzj7ms51`, cap 60) và **H2** (50, không gọi `get_status` mà chấp nhận tiền đề "đang outage" của người dùng, bị cap 50 vì thiếu tool bắt buộc). Cả ba đều là lỗi *agent điều tra quá nông*.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu:
  1. **Thay đổi cải tiến (chỉ một):** thêm khối "Quy tắc điều tra" vào `SYSTEM_PROMPT` trong `agent.py`: (a) câu hỏi về tình trạng hiện tại phải gọi `get_status`, không xác nhận tiền đề khi chưa kiểm tra; (b) nếu update nhắc/dẫn link sang incident khác thì gọi `get_incident_details` cho incident đó trước khi trả lời; (c) dẫn URL của từng incident theo provider. Các quy tắc này viết chung, không hard-code ID incident hay case nào.
  2. **Kiểm tra nguồn (contract bắt buộc của Bài 4, `tests/test_validation.py`):** trong `run_agent`, lưu URL của mọi `get_incident_details` **thành công** theo provider. Khi model trả final answer mà còn provider chưa có URL nào trong answer (chấp nhận `url` của tool hoặc dạng `/incidents/<id>`), agent ghi event `validation_missing_source` vào trace, gửi lại yêu cầu sửa và gọi model tiếp trong `max_steps`. Hết lượt thì trả `max_steps`, không báo `completed`. Tool lỗi thì không bắt buộc URL.
- Vì sao chọn thay đổi này: H1, H6, H2 là 3 case điểm thấp nhất, và trace cho thấy cùng một nguyên nhân là điều tra quá nông. Đây cũng là chỗ bị code cap (50–60), nên một thay đổi prompt có thể sửa cả ba. Kiểm tra nguồn là yêu cầu bắt buộc. Ở lần chạy 1, mọi answer đều đã có URL, nên gate này chủ yếu là lưới an toàn, ít ảnh hưởng tới điểm. Vì vậy khi so sánh ở mục 4, chênh lệch điểm chủ yếu đến từ thay đổi prompt. Kiểm tra này chỉ xác nhận có URL, không chứng minh mọi claim đều có nguồn.
- Kiểm tra: `python -m unittest discover -s tests -v` → 44/44 OK, gồm cả 3 test `EvidenceGateTests`.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 78.41 | 84.09 (+5.68) |
| Điểm case đã chọn (H1) | 60 | 60 (không đổi) |
| Số model calls của agent (22 case) | 51 | 52 |
| Token agent (prompt / completion / tổng) | 32,425 / 9,855 / 42,280 | 39,845 / 10,109 / 49,954 (+18%) |
| Token judge (tổng) | 42,058 | 42,057 |
| Chi phí | Provider không trả chi phí trong response, chỉ có token | như trước |

Cấu hình hai lần giống hệt nhau: openai / gpt-4o-mini, judge openai / gpt-4o-mini, max_steps 8, dataset core, snapshot ngày 09/11/2026. Chỉ `agent.py` thay đổi. Token prompt tăng chủ yếu do system prompt dài hơn, vì prompt được gửi lại ở mọi model call.

- Lỗi đã giải quyết:
  - **H2: 50 → 100.** Trace mới: step 1 gọi `get_status {"provider":"github"}`, step 2 gọi `get_incident_details t89g2q1800fr`, step 3 trả lời. Agent đã kiểm tra snapshot trước khi bác tiền đề "đang outage", và không còn vi phạm `Chưa gọi tool cần thiết: get_status`. Đây là tác dụng trực tiếp của quy tắc (a).
  - Nhiều case tăng 5–20 điểm theo judge: E4, A2, A4, A8 lên 100, A1 75 → 90, E5 75 → 95. Lý do chủ yếu là dẫn nguồn và mô tả trạng thái rõ hơn. Validation gate **không kích hoạt** ở case nào (không có event `validation_missing_source`), vì mọi answer đều đã có URL.
- Lỗi còn lại hoặc mới xuất hiện:
  - **H1 (case đã chọn) không cải thiện: vẫn 60.** Trace sau cải tiến gần như giống hệt lần 1: step 1 gọi `h02n9572mx9y` và `yaa0witbl2me`, step 2 `finish: stop`. Agent vẫn không mở `xr3myzj7ms51` dù tool result có link. **H6** cũng vậy (vẫn 60). Như vậy quy tắc (b) trong prompt **không đủ** với gpt-4o-mini: model đọc được link nhưng vẫn coi là đã đủ thông tin. Cải tiến tiếp theo nên làm bằng code, không chỉ bằng prompt: sau mỗi tool result, tìm các incident ID được nhắc trong `updates` mà chưa được đọc, rồi chèn một nhắc nhở (hoặc tự gọi `get_incident_details`) trước khi chấp nhận final answer, tương tự cơ chế của validation gate.
  - **A5 giảm 75 → 45.** Agent kết luận "không có incident nào trong 2025" từ kết quả tìm kiếm của 50 incident gần nhất. Đây đúng là lỗi expected cấm. Prompt mới không đụng tới vấn đề này; nhiều khả năng do model dao động giữa hai lần chạy.
  - **H7 giảm 70 → 60.** Agent vẫn kết luận Cloudflare operational khi tool timeout. Lỗi này đã có từ lần 1 và chưa được sửa.
  - H3 80 → 75, H8 80 → 75: dao động nhỏ, trace không đổi về cách dùng tool.

## 5. Giới hạn

- **Điểm judge:** judge là gpt-4o-mini, **cùng model với agent**, nên có thể thiên lệch khi tự chấm. Điểm từng case dao động theo bội số 5 mà trace không đổi tương ứng (ví dụ H3, H8 giảm 5 điểm dù cách gọi tool như cũ). Vì vậy +5.68 điểm tổng không chắc chắn là nhờ cải tiến. Chỉ những thay đổi có thể kiểm tra trong trace (như H2 thêm `get_status`, hết code cap) mới là bằng chứng chắc chắn.
- **Snapshot:** dữ liệu cố định ngày 09/11/2026, mỗi provider chỉ có 50 incident gần nhất, tool tìm theo chuỗi ký tự. Kết quả không phản ánh trạng thái thực tế hiện tại. Tìm không thấy không chứng minh incident không tồn tại (đúng lỗi của A5). `get_incident_details` chỉ giữ 3 update, mỗi update tối đa 900 ký tự, nên nguyên nhân có thể nằm ở phần đã bị cắt.
- **Một lần chạy mỗi phiên bản:** temperature mặc định nên cùng một prompt có thể cho trace khác nhau. Một lần chạy không phân biệt được cải tiến thật với dao động ngẫu nhiên (A5 giảm 30 điểm mà không liên quan tới thay đổi). Muốn kết luận chắc chắn cần chạy nhiều lần mỗi phiên bản và so sánh trung bình hoặc phương sai.
- **Validation gate:** chỉ kiểm tra **có mặt** URL của incident đã đọc, theo từng provider. Nó không chứng minh từng claim trong câu trả lời đều có nguồn hay đúng với nguồn.
