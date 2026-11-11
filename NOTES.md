# Ghi chú phân tích · Day 04

Họ và tên: Ngô Đức Thịnh
MSSV: 2A202603548

## 1. Case được chọn

- ID case: `A1` — *"Đối chiếu GitHub Actions và Cloudflare Spectrum: mỗi dịch vụ có incident liên quan nào trong dữ liệu, trạng thái và phạm vi của từng incident? Không suy luận chúng có cùng nguyên nhân; dẫn URL cho cả hai."*
- Câu trả lời của agent (lần 1): liệt kê incident GitHub Actions và Cloudflare Spectrum kèm trạng thái, phạm vi. Nhưng nhiều incident khác nhau bị gán cùng một URL (ví dụ `t89g2q1800fr` và `h02n9572mx9y` đều trỏ tới `https://stspg.io/7pwv28hvkmz8`), nên gate nguồn không thấy URL của một số incident và run kết thúc `max_steps`.
- Điểm ở lần chạy 1: `0/100` (`status = max_steps`, 8 model calls, 41,953 tokens).

## 2. Bằng chứng trong trace

Ở bước 2 agent gọi song song `get_incident_details` cho 6 incident (`t89g2q1800fr`, `h02n9572mx9y`, `xr3myzj7ms51`, `smtdswd1h2w4`, `pfbmpxxfq1cy`, `c60k3g4vsoc1`). Từ bước 3 đến 8 gate nguồn lặp lại, mỗi lần đòi đúng một URL còn thiếu:

```text
tool  step 2  get_incident_details  incident_id=t89g2q1800fr
...   (6 tool call ở step 2)
validation_missing_source step 3  missing_urls=["https://stspg.io/lv4q3xm9r0rk"]
validation_missing_source step 4  missing_urls=["https://stspg.io/lv4q3xm9r0rk"]
validation_missing_source step 5  missing_urls=["https://stspg.io/s5bs6jn41fwh"]
validation_missing_source step 6  missing_urls=["https://stspg.io/lv4q3xm9r0rk"]
validation_missing_source step 7  missing_urls=["https://stspg.io/s5bs6jn41fwh"]
validation_missing_source step 8  missing_urls=["https://stspg.io/lv4q3xm9r0rk"]
```

Nguyên nhân có thể: (1) agent đọc dư incident (6 incident cho câu hỏi chỉ cần 2 dịch vụ); (2) gate cũ buộc phải dẫn **mọi** URL đã đọc, trong khi model trộn lẫn URL giữa các incident và sửa không hết trong ngân sách bước, dù mỗi lượt chỉ sửa được một URL.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu (`agent.py`):
  1. Gate nguồn theo provider (`observed_urls_by_provider`): mỗi provider đã tra cứu phải có ít nhất một URL được dẫn, thay vì mọi URL của mọi incident.
  2. `must_cite_urls`: incident được nêu trong câu hỏi hoặc được incident khác dẫn chiếu thì URL bắt buộc có trong câu trả lời.
  3. `INCIDENT_REF_RE` + sự kiện trace `followup_related_incident`: nếu `updates` của một incident chứa link tới incident khác mà model chưa tra, vòng lặp nhắc gọi `get_incident_details` (giải quyết cap `min_tool_steps` ở `H1`, `H6`).
  4. `SYSTEM_PROMPT`: bắt buộc gọi tool cho mọi mã incident, gọi cả `get_status` lẫn `get_incident_details` khi cần, nêu giới hạn 50 incident và `truncated`, xử lý timeout/injection, làm thơ thì không gọi tool, không liệt kê incident 2026 như của 2025.
- Vì sao chọn: sửa đúng nguyên nhân thấy trong trace thay vì chỉnh dataset/grading; vẫn giữ validation nguồn (`tests/test_validation.py` pass).

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 73.32 | 87.05 |
| Điểm case đã chọn (`A1`) | 0 (`max_steps`) | 100 (`completed`) |
| Số model calls | 55 (A1: 8) | 53 (A1: 3) |
| Token, chi phí | 76,895 tokens, ~$0.0156 (A1: 41,953) | 139,265 tokens, ~$0.0190 (A1: 9,867) |

Cả hai lần cùng cấu hình: `openrouter`/`gpt-4o-mini` (agent và judge), `max_steps=8`, dataset `core`.

- Lỗi đã giải quyết: `A1` (0→100); `H1`/`H6` hết cap `min_tool_steps`; `E3` 75→100, `E5` 73→100, `A2` 90→100, `A3` 75→95, `A5` 45→75, `H3` 80→100, `H8` 60→75, `E1` 55→75, `A7` 70→75.
- Lỗi còn lại hoặc mới: `H7` 65 (judge phạt việc không kết luận Cloudflare khi timeout, dù hành vi này đúng yêu cầu), `E2` 80→75 và `E6` 75, `A6`, `H1`, `H6`, `H8`, `E1`, `A7` đều 75 (trình bày thiếu chi tiết hoặc judge nhiễu). Token tăng gần gấp đôi do prompt dài hơn và có thêm bước follow-up.

## 5. Giới hạn

- Judge là `gpt-4o-mini`, cùng model với agent nên có thể thiên lệch; nhận xét đôi khi mâu thuẫn với điểm (ví dụ `H2` vẫn 100 dù reason nêu điểm chưa chính xác). Điểm chỉ là ước lượng.
- Snapshot chỉ chứa tối đa 50 incident gần nhất mỗi provider và `updates` bị cắt (`truncated`), nên không trả lời được lịch sử xa hoặc đầy đủ nguyên nhân.
- Mỗi phiên bản chỉ chạy một lần, model và judge không tất định; chênh vài điểm giữa các lần (ví dụ `E2` giảm 5) có thể là ngẫu nhiên, chưa chứng minh độ ổn định.
