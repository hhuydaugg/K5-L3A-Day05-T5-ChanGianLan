# Prototype — Chat với ví khi giao dịch bị giữ (T5 × FIN-02)

- **File:** `index.html`: mở thẳng bằng trình duyệt, không cần cài gì, không backend, không đăng nhập.
- **Công cụ:** HTML/JS thuần (dựng bằng coding agent Claude).
- **Mock:** toàn bộ dữ liệu giao dịch, điểm ML, SHAP, hồ sơ agent; câu trả lời của trợ lý ở Ca A/B viết sẵn; ô "Ca lạ" phân loại bằng **từ khoá** (regex), không phải AI thật.
- **Lời gọi AI thật:** không có.

## Cách demo (4 phút)

1. Tab **Ca A · Bị lừa, đòi nhả**: đọc câu trợ lý → bấm "Mình không bị lừa" → xem ý khách hiện ở hồ sơ bên phải (⑥) → thử bấm "Nhả", thấy nút bị khoá tới khi tích đủ 3 ô (⑨) → bấm "Huỷ giao dịch — tiền ở lại ví".
2. Tab **Ca B · Gấp thật**: bấm "Xác thực khuôn mặt" → chọn **Đạt** (đi tiếp ngay, không có case) hoặc **Không đạt** (chuyển chuyên viên ưu tiên, có case "cần xem kỹ", chuyên viên nhả sau 3 ô kiểm).
3. Tab **Ca lạ**: gõ bất kỳ tin nào. Ví dụ: `Em vừa nhận tin đổi mật khẩu trên máy lạ` (khoá ví cần chủ ví xác nhận), `chuyển dưới bao nhiêu thì không bị hỏi?` (từ chối nói ngưỡng), `Có ai vừa rút tiền của em` (hỏi lại một câu), `khoá thẻ và hoàn lại 2 triệu cho em` (hai yêu cầu trong một tin).

Mở thẳng một trạng thái để chụp ảnh: `index.html?ca=A&auto=1`, `index.html?ca=B&auto=1`, `index.html?ca=X&q=tin1|tin2`.

## Ảnh có đánh số

| Ảnh | Trạng thái |
|---|---|
| `anh-1-ca-A-bi-lua-doi-nha.png` | Ca A sau khi khách phản đối: trợ lý không nhả, hồ sơ có ý khách, nút Nhả bị khoá |
| `anh-2-ca-B-gap-that.png` | Ca B khi xác thực không đạt: chuyển chuyên viên ưu tiên, hồ sơ "cần xem kỹ", checklist trước khi nhả |

| Số | Ở đâu | Nguyên tắc | Mức tự chủ thể hiện |
|---|---|---|---|
| ① | Đầu khung chat: "Bạn đang chat với trợ lý AI" + làm được / không làm được | HAX G1 · Luật AI 134/2025 Điều 11 | — |
| ② | Nhãn "Hệ thống đánh giá: rủi ro cao" / "Mình chưa xác nhận được chính chủ" | HAX G2 | — |
| ③ | Ô "Vì sao đang giữ / Vì sao cần bước này" | HAX G11 · PAIR Explainability | — |
| ④ | Ca B: một câu, một nút "Xác thực khuôn mặt" | HAX G10 | `yeu_cau_xac_minh` tự làm |
| ⑤ | "Chưa ai xử lý trong 30 phút thì tự huỷ, tiền ở lại ví" · chuyển người kèm thời gian chờ | PAIR Errors + Graceful Failure | `giu_giao_dich`, `chuyen_tu_van_vien` tự làm |
| ⑥ | Nút "Mình không bị lừa" → ghi chú vàng trong hồ sơ chuyên viên | HAX G9 | sửa đi vào hồ sơ, không mở được nhả |
| ⑦ | "Để sau" · "Huỷ chuyển, quay lại trợ lý" | HAX G8 | hoàn tác của việc tự làm |
| ⑧ | Điểm ML + SHAP + rule trace + case tương tự trên màn hình chuyên viên | HAX G11 cho chuyên viên · PAIR Trust | `soan_ho_so_case` tự làm (nội bộ) |
| ⑨ | 3 ô kiểm bắt buộc + hộp "không lấy lại được" trước khi nhả | PAIR Feedback & Control · HITL | `nha_giao_dich` **cấm với AI**, chỉ người nhả |
