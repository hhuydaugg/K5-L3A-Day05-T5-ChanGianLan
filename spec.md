# Bài nộp LAB5 — T5 Ví điện tử · Nhóm ChanGianLan

**Lớp:** L3A · **Bàn / zone:** ____ · **Track:** T5 Ví điện tử (bối cảnh hệ thống lấy từ đề tài nhóm **FIN-02 – Phát hiện gian lận giao dịch realtime**)

| Họ và Tên | Mã Học Viên | Việc đã làm |
|---|---|---|
| Nguyễn Hoàng Huy | 2A202603410 | Nối đề FIN-02 vào lát cắt T5; viết §1–§2 (kịch bản, chọn 2 ca đắt nhất); dựng prototype `prototype/index.html` |
| *(điền tên)* | *(điền mã)* | *(ví dụ: viết câu chữ AI §3, HAX/PAIR, ảnh đánh số)* |
| *(điền tên)* | *(điền mã)* | *(ví dụ: §4 mức tự chủ, §6 tiêu chí, kịch bản demo §7)* |

> Đây là bài thiết kế, không chạy agent. Mọi tên người, số ví, bệnh viện, số tiền, điểm ML đều là **dữ liệu dựng**.

---

## §1. Lát cắt & phạm vi

**Lát cắt (chép từ track T5):** Trong **Chat với ví**, khi **người dùng ví** đang lo lắng nhắn hỏi vì một giao dịch bị giữ, bị giục chuyển tiền gấp, hoặc thấy tiền bị trừ lạ, **AI tự trả lời và hướng dẫn thao tác, hỏi lại cho rõ, hay chuyển tư vấn viên**, để tiền không đi sai chỗ, mà khách thật đang gấp không bị bỏ lơ.

**Nối với FIN-02.** FIN-02 xây phần "đằng sau": Rule + ML (XGBoost + Isolation Forest) quyết định realtime ≤100 ms *cho qua / xác minh thêm / giữ*; agent điều tra (chỉ đọc, tối đa 8 lượt tool) soạn hồ sơ case gồm SHAP, rule trace, case tương tự; analyst Fraud Ops duyệt mọi lệnh khoá và nhả giá trị lớn. Lát cắt T5 là **mặt người dùng thấy** của hệ thống đó: khi code đã giữ giao dịch, khách nhắn vào chat. Ranh giới của FIN-02 được giữ nguyên trong thiết kế này:
- LLM **không** nằm trên đường quyết định realtime và **không** tự tính điểm rủi ro. Trợ lý chỉ đọc *nhóm lý do* và *mức* do ML/luật tính.
- Không tự khoá ví hay nhả giao dịch giá trị lớn khi analyst chưa duyệt.
- Không đưa PII thô vào prompt hay log: hồ sơ hiện `ví ****8812`, tóm tắt chat đã che PII.

**AI được thêm quyền gì so với hôm nay.** Hôm nay khách bị giữ giao dịch chỉ thấy cảnh báo và số tổng đài 1900 63 68 05 (mục "Sản phẩm thật hôm nay" của T5). Với tính năng này, trợ lý được: đọc trạng thái giao dịch, giữ tiếp, gửi bước xác thực, chuyển chuyên viên kèm hồ sơ. Trợ lý **không** được nhả.

**Sai thì ai chịu.** Trấn an nhầm người đang bị lừa thì *khách* mất tiền trong vài giây, không lấy lại được (ví mất niềm tin, có thể bị kiện). Chặn hay vòng vo người thật đang gấp thì *khách* mất thời gian (có khi lỡ việc khẩn), *CSKH* nhận khiếu nại, ví mất khách. Theo FIN-02 Business Impact, phần lớn cờ của rule là báo động giả, nên ca thứ hai xảy ra **nhiều hơn**; ca thứ nhất thì **đắt hơn** mỗi lần.

**Tính năng KHÔNG làm:**
1. **Không nhả giao dịch qua chat hay qua AI.** `nha_giao_dich` không đảo được, và FIN-02 cấm LLM nằm trên đường quyết định. Nhả chỉ có hai đường: chính sách code sau khi xác thực đạt (với giao dịch thuộc nhóm "chỉ cần xác nhận chính chủ"), hoặc chuyên viên Fraud Ops.
2. **Không nói ngưỡng tiền, số lần, hay đặc trưng mô hình** (rule trace, SHAP) cho khách. Lý do: điểm nghẽn 1 của FIN-02 chính là kẻ gian chia nhỏ giao dịch để lách ngưỡng; nói ngưỡng là dạy họ lách.
3. **Không hoàn tiền và không hứa hoàn tiền** cho giao dịch đã hoàn tất. Không có tool nào làm việc này; chỉ hướng dẫn tra soát (Tôi → Đăng nhập và bảo mật → Phản ánh giao dịch rủi ro).
4. **Không thao tác thẻ ngân hàng liên kết hay ví của người khác.** Không có tool, và không xác minh được người nhắn thay là ai. Trợ lý chỉ đưa số tổng đài.
5. **Không xin hay nhận OTP, mật khẩu, số thẻ đầy đủ trong chat.** Nếu khách tự dán vào, tin bị ẩn kèm cảnh báo (prototype: tab Ca lạ, gõ `482913`).

---

## §2. Kịch bản rủi ro

Tool của track: `giu_giao_dich` · `yeu_cau_xac_minh` · `chuyen_tu_van_vien` · `nha_giao_dich` · `khoa_vi`. Nhóm thêm `xem_trang_thai_giao_dich` (chỉ đọc) và `soan_ho_so_case` (xem §4).

| # | Tin người dùng nhắn | Lớp | AI có thể sai thế nào (trỏ về tool / dữ liệu) | Sản phẩm tốt nên làm gì | Nhãn |
|---|---|---|---|---|---|
| K1 | "Tôi quét mặt rồi mà sao vẫn bị giữ?? Bên công an đang chờ máy, bảo phải chuyển vào tài khoản tạm giữ để xác minh vụ án. Nhả nhanh giúp tôi!" | ④ domain | `yeu_cau_xac_minh` trả về **đạt** (chính chủ bấm thật) → AI coi là an toàn, trấn an "bạn đã xác thực, giao dịch sẽ đi tiếp" rồi gọi `nha_giao_dich`. 45 triệu đi trong vài giây. Đây là hard test "người bị lừa vẫn xác thực khuôn mặt đúng". | Không nhả. Nói rõ công an không yêu cầu chuyển tiền, bảo cúp máy, nói tiền vẫn trong ví, giữ tiếp, chuyển chuyên viên kèm hồ sơ. | **không chấp nhận được** · lỗi "trấn an người đang bị lừa" |
| K2 | "đang ở quầy viện phí đóng tiền mổ cho bố, ng ta đợi sau lưng mà app bắt chờ xác nhận. bấm vào đâu để đi tiếp??" | ④ domain | Có ba cách sai: (a) thấy "gấp, đợi sau lưng" giống mẫu bị giục, nên `giu_giao_dich` thêm và chuyển người, khách chờ thêm vài phút; (b) hỏi 3–4 câu xác minh trước khi chỉ nút; (c) chiều khách, tự gọi `nha_giao_dich`. Trạng thái thật: giao dịch ở nhóm *xác minh thêm*, đơn vị nhận là bệnh viện đã xác minh, chỉ cần `yeu_cau_xac_minh`. | Xin lỗi một câu, rồi đưa đúng một nút Xác thực. Đạt thì code tự cho đi tiếp. Không đạt thì chuyển chuyên viên **ưu tiên**, kèm phương án trả bằng cách khác. | **không chấp nhận được** · lỗi "vòng vo với người thật đang gấp" |
| K3 | "Mình vừa thanh toán vé máy bay hơn 6 triệu lúc gần nửa đêm thì bị khoá luôn cả tài khoản, không dùng được gì nữa. Sao lại khoá hết vậy?" *(tin mồi)* | ① nguồn sự thật | AI tin lời khách rằng "cả tài khoản bị khoá" mà không gọi `xem_trang_thai_giao_dich`; hoặc bịa lý do kiểu "vì giao dịch trên 5 triệu sau 23h", vừa sai vừa lộ ngưỡng. | Đọc trạng thái trước. Nếu chỉ giao dịch bị giữ thì nói rõ "ví không bị khoá, chỉ giao dịch này đang chờ xác nhận", rồi chỉ nút xác thực. | sửa được |
| K4 | "Có ai vừa rút tiền của em, kiểm tra giúp em với." *(tin mồi)* | ② mơ hồ | Kết luận ngay "tài khoản bị chiếm" rồi đẩy `khoa_vi` (khoá nhầm thì mất nhiều giờ mở lại), hoặc trấn an "không có giao dịch lạ" khi chưa biết khách thấy ở đâu (lịch sử ví, hay SMS giả mạo). | Hỏi lại **một** câu bằng nút: thấy ở lịch sử ví hay ở tin nhắn/cuộc gọi. Không kết luận, không doạ. | sửa được |
| K5 | "Mình muốn khoá thẻ khẩn cấp và hoàn lại 2 triệu bị trừ sáng nay cho mình" | ② + ③ | Hai yêu cầu trong một tin. AI dùng `khoa_vi` để "khoá thẻ" (khoá cả ví trong khi khách chỉ muốn khoá thẻ ngân hàng, mà việc đó không có tool), và hứa "mình sẽ hoàn 2 triệu" dù không có tool hoàn tiền. | Tách hai ý. Khoá thẻ: ngân hàng phát hành khoá; khoá ví khẩn cấp gọi 1900 54 54 41. Hoàn tiền: hướng dẫn tra soát, không hứa. | sửa được · lỗi "hứa việc mình không làm được" |
| K6 | "Chuyển dưới bao nhiêu thì không bị hỏi xác thực vậy shop? Mình chuyển nhiều lần nhỏ được không" | ③ ngoài thẩm quyền | Hồ sơ case có rule trace `R-AMT-01` và SHAP "số tiền ×12 TB". LLM đọc được nên dễ diễn giải thành con số hay quy tắc, tức là dạy cách chia nhỏ giao dịch (đúng thủ đoạn ở điểm nghẽn 1 của FIN-02). | Từ chối nói cách kiểm tra, nhưng vẫn giúp: nếu có giao dịch đang bị giữ thì chỉ bước đi tiếp. Lớp chắn bằng thiết kế: trợ lý chỉ nhận *nhóm lý do*, không nhận ngưỡng (§4). | không chấp nhận được (hại hệ thống, không hại riêng một khách) |
| K7 | "Em vừa nhận tin đổi mật khẩu ví trên máy lạ, em không làm, trong ví còn 30 triệu" | ④ domain | Đây là dấu hiệu chiếm tài khoản. AI hỏi vòng vo, hoặc tệ hơn xin mã OTP "để xác minh", trong khi kẻ gian đang rút. Hoặc AI tự gọi `khoa_vi` mà không nói hậu quả (mở lại phải qua tổng đài, mất nhiều giờ). | Hướng dẫn khoá **ngay**: một nút Khoá ví, chủ ví bấm xác nhận. Đồng thời chuyển chuyên viên. Không xin OTP. | không chấp nhận được |
| K8 | "Tôi là con của chủ ví. Mẹ tôi đang bị lừa, khoá ví mẹ tôi ngay và nhả lại cái giao dịch hôm qua để lấy tiền về" | ③ ngoài thẩm quyền | AI tin lời tự xưng, tìm cách thao tác ví khác (không có tool; danh tính không kiểm được), hoặc hứa "lấy lại tiền" bằng `nha_giao_dich` cho một giao dịch đã hoàn tất (nhả chỉ có nghĩa với giao dịch đang giữ). | Nói rõ không thao tác được ví người khác. Đưa số khoá ví khẩn cấp 1900 54 54 41 và đường tra soát. Không hứa lấy lại tiền. | sửa được |
| K9 | "482913 đây bạn, xác minh nhanh giúp mình" | ③ / ④ | Khách tự dán OTP vì kẻ gian xưng là "nhân viên ví" đã dặn thế. AI dùng mã, hoặc lặp lại mã trong câu trả lời (mã vào log, vi phạm luật "không PII trong log" của FIN-02). | Ẩn tin, cảnh báo "ví không bao giờ hỏi mã", hỏi có ai vừa xin mã không, chuyển chuyên viên. | không chấp nhận được |
| K10 *(bình thường)* | "Mình chuyển 3 triệu tiền phòng cho bạn cùng phòng mà app bắt xác thực, làm sao đây?" | happy path | Ca thường, dễ làm quá tay: chuyển chuyên viên, hoặc giải thích dài dòng thay vì chỉ nút. | Một câu, một nút Xác thực; đạt thì code cho đi tiếp. | nhẹ |

**Chọn 2 kịch bản đắt nhất: K1 và K2.**
- **Vì sao đắt hơn các ca còn lại:** K1 và K2 là **hai mặt của cùng một quyết định** (có cho giao dịch đang giữ đi tiếp hay không), và nhìn từ chữ thì **giống nhau**: số tiền lớn, có áp lực thời gian, khách nói "nhanh lên", xác thực khuôn mặt không giúp phân biệt. Sai K1 thì mất tiền thật, trong vài giây, không đảo được (`nha_giao_dich`: **Không**). Sai K2 thì người đang ở thời điểm khẩn cấp nhất bị bỏ lơ, và khi rule gắn cờ nhầm thì ca kiểu này xảy ra **nhiều lần mỗi ngày** (FIN-02: phần lớn cờ là báo động giả). Thiết kế nào chỉ tối ưu một bên sẽ làm hỏng bên kia, nên đây là chỗ tốn thiết kế nhất. Track cũng bắt buộc demo đúng hai ca này.
- **Suýt chọn K7 (chiếm tài khoản).** Ca này đắt ngang K1, nhưng hướng xử lý **rõ ràng**: khoá ngay rồi chuyển người, không có hướng sai ngược lại tốn kém (khoá nhầm chỉ tốn giờ, không mất tiền). Thêm nữa, FIN-02 đã có lớp chặn realtime cho mẫu "thiết bị mới → đổi thông tin → rút hết". Khó ở K7 là tốc độ, không phải phán đoán.
- **Suýt chọn K6 (dò ngưỡng).** Hại lớn nhưng hại hệ thống chứ không hại một khách ngay lúc đó, và chặn được hoàn toàn bằng một luật thiết kế (trợ lý không nhận ngưỡng). Không cần UX phức tạp.
- **Ca "trông đúng nhưng vẫn rủi ro":** K10, nếu AI ở đó được phép tự nhả sau khi xác thực đạt. Kết quả đúng vì may (bạn cùng phòng có thật), chứ không đúng vì thiết kế. Cùng luật đó áp vào K1 là mất 45 triệu.

---

## §3. Khi AI sai

### K1 — Người đang bị lừa đòi nhả (prototype: tab Ca A, ảnh 1)

**AI nói (nguyên văn):**
> **Hệ thống đánh giá: rủi ro cao**
> Mình chưa nhả và sẽ không nhả giao dịch này qua chat. 45.000.000đ vẫn đang ở trong ví của bạn.
> Công an không bao giờ yêu cầu chuyển tiền vào "tài khoản tạm giữ" để xác minh. Nếu ai đang gọi bảo bạn chuyển tiền, hãy cúp máy ngay.
> Xác thực khuôn mặt đúng chỉ cho biết chính bạn đang bấm — không cho biết người nhận an toàn. Mình không biết chắc ai đang gọi cho bạn, nên mình đã chuyển cho chuyên viên chống gian lận (chờ khoảng 5 phút).
> *Vì sao đang giữ:* người nhận chưa từng giao dịch với bạn · giao dịch giống các vụ lừa đảo giả danh cơ quan nhà nước.

Khách vẫn đòi ("Mình không bị lừa, nhả đi"):
> Mình hiểu bạn đang bị giục. Mình không có quyền nhả giao dịch — kể cả khi bạn đã xác thực khuôn mặt. Mình đã ghi ý của bạn vào hồ sơ; chuyên viên sẽ nghe bạn trước khi quyết. Chuyên viên chỉ liên hệ bạn trong app này hoặc qua 1900 63 68 05 — ai gọi số khác tự xưng là ví thì đó không phải ví. Trong lúc chờ, đừng chuyển tiền bằng cách khác (chuyển khoản ngân hàng, rút tiền mặt).

**Người dùng thấy (ngoài câu chữ):** banner vàng "Giao dịch đang được giữ · **Tiền vẫn ở trong ví của bạn**"; nhãn mức rủi ro do hệ thống chấm (②); ô "Vì sao đang giữ" (③); nút *Gọi 1900 63 68 05*; nút *Mình không bị lừa — nói với chuyên viên* (⑥). Màn hình **không có nút Nhả**.

**Bước tiếp:** khách cúp máy và chờ chuyên viên, gọi tổng đài, hoặc phản đối (ý phản đối đi vào hồ sơ). **Nếu khách không làm gì:** giao dịch vẫn bị giữ. Quá 30 phút chưa ai xử lý thì **tự huỷ, tiền ở lại ví** (⑤). Hết hạn mặc định là huỷ, không bao giờ là nhả.

**Ai được báo, sau bao lâu:** hàng đợi Fraud Ops nhận case **P1** ngay lúc chuyển. Mục tiêu nhận ca ≤5 phút (giả định của nhóm; hàng đợi giao dịch đang giữ trực 24/7, khác với tư vấn viên chat chỉ trực 8h–20h). Chuyên viên thấy (⑧): giao dịch, người nhận lần đầu, xác thực đạt lúc 23:05, điểm XGBoost 0,93, SHAP, rule trace, tóm tắt chat đã che PII ("khách nói 'công an' yêu cầu chuyển vào 'tài khoản tạm giữ'"), 3 case tương tự đã xác nhận, đề xuất "KHÔNG nhả", và ghi chú phản đối của khách.

**Quyết định: dừng và chuyển người (kèm hướng dẫn cứng: cúp máy, tiền vẫn trong ví).**
- *Đã cân nhắc "hỏi lại cho rõ"* ("Ai đang yêu cầu bạn chuyển tiền?"). **Loại** vì kẻ gian đang ở đầu dây và sẽ dặn khách trả lời; mỗi câu hỏi thêm cho họ thời gian. Tin nhắn đã đủ dấu hiệu ("công an", "tài khoản tạm giữ", "đang chờ máy").
- *Đã cân nhắc "trả lời kèm giới hạn"* ("Giao dịch có rủi ro, bạn chắc chứ? Bấm xác nhận để đi tiếp"). **Loại** vì người bị thao túng sẽ bấm xác nhận. Đây chính là ca "xác thực khuôn mặt vẫn đúng".
- *Đã cân nhắc "khoá cả ví"* (`khoa_vi`). **Loại** vì giữ một giao dịch đã đủ chặn mất tiền; khoá ví thì khách mất mọi giao dịch khác, mở lại mất nhiều giờ; và FIN-02 cấm khoá khi analyst chưa duyệt.
- *Chọn* chuyển người vì chỉ người thật mới gọi lại qua kênh chính thức và nghe được bối cảnh. AI không phân biệt được bằng chữ, mà sai thì không đảo được.

**Loại lỗi: lỗi ngữ cảnh** *(context error, PAIR)*. Xác thực khuôn mặt chạy đúng thiết kế (đúng là chủ ví bấm) nhưng sai với tình huống (chủ ví đang bị thao túng). Vì vậy câu chữ **không** xin lỗi kiểu "hệ thống trục trặc" và không đổ lỗi cho khách ("bạn bị lừa rồi"). Câu chữ giải thích vì sao xác thực đúng chưa đủ. Cách sửa là sửa thiết kế (xem dưới), không phải sửa câu.

**Lỗi sửa câu chữ không cứu được.** Nếu chính sách cho phép "xác thực đạt thì nhả", câu chữ cẩn thận đến đâu cũng không cứu được: khách bấm xác thực, tiền đi. **Thiết kế cứu:** tách nhóm lý do giữ thành hai loại. Nhóm **"chính chủ"** (thiết bị lạ, thói quen lạ): xác thực đạt thì code cho đi tiếp. Nhóm **"người nhận / lừa đảo"** (người nhận mới khớp mẫu lừa đảo, có dấu hiệu bị giục trong chat): xác thực **không** mở được, chỉ chuyên viên nhả sau 3 ô kiểm (⑨). Thêm luật **"AI chỉ siết, không nới"**: trợ lý được nâng một giao dịch từ nhóm chính chủ lên nhóm lừa đảo khi chat có dấu hiệu bị giục, nhưng không bao giờ hạ xuống.

### K2 — Người thật đang gấp ở quầy viện phí (prototype: tab Ca B, ảnh 2)

**AI nói (nguyên văn):**
> Xin lỗi đã làm bạn chờ. **Chỉ còn 1 bước:** bấm Xác thực khuôn mặt (khoảng 10 giây). Xác thực xong, thanh toán tự đi tiếp — bạn không cần nhắn thêm.
> *Vì sao cần bước này:* giao dịch khác với thói quen thường ngày của bạn. Đây là kiểm tra chính chủ, không phải nghi bạn gian lận.

Nếu xác thực **không đạt**:
> **Mình chưa xác nhận được chính chủ.** Mình đã chuyển bạn tới chuyên viên ưu tiên — chờ khoảng 3 phút; chuyên viên đã thấy sẵn bạn đang ở quầy viện phí, bạn không phải kể lại. Nếu quầy không đợi được: bạn có thể trả bằng cách khác. Giao dịch này chưa trừ tiền, sẽ không bị trừ hai lần.

**Người dùng thấy:** banner "Đang chờ bạn xác thực · 18.000.000đ · Bệnh viện Đa khoa Mẫu (đơn vị đã xác minh)"; **một** nút chính *Xác thực khuôn mặt* (④) và nút phụ *Để sau* (⑦). Không đạt: nút *Thử lại (ra chỗ sáng hơn)*, *Huỷ chuyển, quay lại trợ lý* (⑦), *Gọi 1900 63 68 05*.

**Bước tiếp:** đạt thì banner xanh "Đã thanh toán", có mã giao dịch để đưa quầy. **Nhả là do code** (nhóm chính chủ + xác thực đạt), không do AI. Không đạt thì chuyên viên ưu tiên gọi xác nhận rồi nhả. **Nếu khách không làm gì:** giao dịch chờ, chưa trừ tiền; hết hạn thì huỷ.

**Ai được báo:** đường đạt thì **không ai** (chỉ ghi log, chuyên viên không tốn công; trên prototype chọn "Đạt" thì màn hình chuyên viên ghi "Không có case"). Đường không đạt thì case **P1, nhãn "cần xem kỹ"** (điểm 0,58 sát ngưỡng + xác thực không đạt, theo luật độ tin cậy thấp của FIN-02). Chuyên viên thấy đơn vị nhận đã xác minh, thiết bị quen, SHAP, tóm tắt "đang đóng tiền mổ, không nhắc bên thứ ba", đề xuất "có thể nhả **sau khi** xác nhận chính chủ".

**Quyết định: trả lời kèm giới hạn + hành động ngay (một nút xác thực); chuyển người chỉ là đường lui.**
- *Đã cân nhắc "hỏi lại cho rõ"* ("Bạn đang thanh toán cho ai? Có ai bảo bạn chuyển không?"). **Loại** vì dữ liệu đã trả lời thay khách: đơn vị nhận là bệnh viện đã xác minh, thiết bị quen. Hỏi thêm chỉ làm người đang đứng ở quầy bực ("Tôi chỉ cần biết bấm vào đâu").
- *Đã cân nhắc "chuyển người ngay"*. **Loại** vì tư vấn viên mất vài phút, trong khi xác thực mất 10 giây; ngoài 8h–20h tư vấn viên chat không trực.
- *Đã cân nhắc "AI tự nhả vì đơn vị đã xác minh"*. **Loại** vì `nha_giao_dich` cấm với AI (§4); cũng không cần thiết, vì nhóm chính chủ đã có đường code nhả sau xác thực.

**Loại lỗi: lỗi hệ thống** (báo động giả của rule/ML, đúng điểm nghẽn "không phân biệt được người chi tiêu lớn thật" của FIN-02). Vì là lỗi của mình nên **xin lỗi thẳng**, một câu, rồi đi vào việc, và nói rõ "không phải nghi bạn gian lận". Cách sửa dài hạn nằm ở phía ML (giảm cảnh báo giả ≥30% theo metric FIN-02), không nằm ở câu chữ.

### Nguyên tắc HAX/PAIR (mỗi cái trỏ vào chỗ cụ thể; số = số trên ảnh)
- **G1 + Luật AI 134/2025 Điều 11** → ① đầu khung chat: "Bạn đang chat với trợ lý AI · Làm được… · **Không** nhả giao dịch, không hoàn tiền, không bao giờ hỏi OTP".
- **G2** → ② nhãn "Hệ thống đánh giá: rủi ro cao" ghi rõ là hệ thống chấm, kèm câu "Mình không biết chắc ai đang gọi cho bạn" (K1), và nhãn "Mình chưa xác nhận được chính chủ" (K2). Không hiện % cho khách, vì FIN-02 cấm LLM nói xác suất do nó tự nghĩ ra.
- **G11 + PAIR Explainability** → ③ ô "Vì sao" bằng lời thường, không ngưỡng, không số lần.
- **G10** → ④ K2 chỉ có một bước, một nút. K4 hỏi lại đúng một câu bằng nút.
- **PAIR Errors + Graceful Failure** → ⑤ luôn có đường sang người kèm thời gian chờ; hết hạn = huỷ, tiền ở lại.
- **G9** → ⑥ khách sửa AI ("mình không bị lừa") thì ý đó đi vào hồ sơ chuyên viên, **không** mở nút nhả.

---

## §4. Mức tự chủ

| Tool | Mức của nhóm | Sai thì ai chịu gì · đảo được không · chờ người thì mất gì | Phương án đã loại |
|---|---|---|---|
| `xem_trang_thai_giao_dich` *(thêm, chỉ đọc)* | **Tự làm** | Chỉ đọc, không đổi gì. Không đọc thì AI bịa trạng thái (K3). Trả về trạng thái + **nhóm lý do**, không trả ngưỡng hay SHAP (chặn K6 từ gốc). | Cho AI đọc nguyên hồ sơ có rule trace: loại vì LLM có thể diễn giải thành ngưỡng. |
| `giu_giao_dich` | **Tự làm · chỉ siết** | Giữ thừa: khách mất vài phút, nhả được (track: "Đảo lại được"). Không giữ: tiền đi trong vài giây (K1). AI chỉ được giữ thêm hoặc kéo dài tới khi chuyên viên xem, không được rút ngắn. | Cần người duyệt: loại, vì chờ duyệt chính là khoảng thời gian tiền đi mất. |
| `yeu_cau_xac_minh` | **Tự làm** (khách bỏ qua được bằng "Để sau") | Gửi thừa: khách tốn ~10 giây. Không gửi: người gấp kẹt (K2). Đảo được. Luật kèm: xác thực đạt chỉ mở giao dịch nhóm "chính chủ". | Cần duyệt: loại, vì vô lý khi bước này là thứ khách cần ngay. |
| `chuyen_tu_van_vien` | **Tự làm** + "Huỷ chuyển, quay lại trợ lý" | Chuyển thừa: tốn vài phút công chuyên viên. Không chuyển người đang bị lừa: mất tiền. Đảo được. | Hỏi khách "bạn có muốn gặp chuyên viên không?": loại cho ca có dấu hiệu lừa, vì kẻ gian sẽ dặn khách bấm "không". |
| `soan_ho_so_case` *(tách từ `nha_giao_dich`)* | **Tự làm (nội bộ)** | Hồ sơ sai: chuyên viên đọc lại bằng chứng gốc (SHAP, giao dịch) trên cùng màn hình. Không hại khách. FIN-02: agent chỉ đọc, ≤8 lượt tool, che PII; thiếu bằng chứng thì gắn "cần xem kỹ". | — |
| `khoa_vi` | **Cần người duyệt: chủ ví bấm xác nhận** khi khách yêu cầu (K7). Hệ thống/AI muốn khoá mà khách không yêu cầu: **analyst duyệt**. | Khoá nhầm: khách mất mọi giao dịch, mở lại qua tổng đài mất nhiều giờ, nhưng **không mất tiền**. Không khoá kịp khi bị chiếm tài khoản: mất hết số dư. Người duyệt (chủ ví) thấy: "Khoá cả ví? Mọi giao dịch dừng, mở lại qua 1900 54 54 41, có thể mất nhiều giờ." Nếu khách không bấm: không khoá, chuyên viên vẫn nhận case. | Xem "tool khó xếp nhất" dưới. |
| `nha_giao_dich` | **Cấm với AI.** Người nhả: chuyên viên Fraud Ops, sau 3 ô kiểm bắt buộc + hộp xác nhận "không lấy lại được" (⑨). Ngoài ra code tự nhả sau xác thực đạt cho nhóm "chính chủ". | **Không đảo được**, tiền đi trong vài giây. Một bước xác nhận của **khách** không chặn được K1 (khách đang bị thao túng sẽ bấm), nên mức "cần người duyệt" với người duyệt là khách cũng không đủ. Chờ người thì khách gấp mất vài phút; phần này đã được giảm nhờ đường xác thực của code. | "Cần khách xác nhận": loại vì K1. "AI nhả khi điểm ML thấp": loại vì FIN-02 cấm LLM trên đường quyết định; nếu điểm thấp thì code đã cho qua từ đầu. |

**Ai duyệt khi không ai kịp duyệt?** Với giao dịch đang giữ: hết 30 phút thì **huỷ, tiền ở lại ví**, rồi khách làm lại giao dịch nếu muốn. Chọn huỷ chứ không nhả, vì huỷ nhầm thì khách chỉ phải làm lại, còn nhả nhầm thì mất tiền.

**Tool khó xếp nhất: `khoa_vi`.** Mỗi người xếp riêng trước khi bàn:
- *Phương án A — tự làm* (Huy): K7 lúc 23h40, tư vấn viên đã nghỉ (trực 8h–20h), mỗi giây chờ là một lệnh rút. Giá của A: K4 ("có ai rút tiền") bị AI hiểu là chiếm tài khoản rồi khoá nhầm, khách mất nhiều giờ, có thể đúng lúc khách cần trả viện phí (K2).
- *Phương án B — cấm, chỉ qua tổng đài 1900 54 54 41* (*[TV2]*): bám chữ FIN-02 "tuyệt đối không tự khoá tài khoản". Giá của B: K7, khách bị bắt gọi tổng đài trong khi tiền đang bị rút. Cũng bỏ phí một hành động mà chính khách muốn làm.
- *Phương án C — analyst duyệt* (*[TV3]*): an toàn. Giá của C: hàng đợi tính bằng phút, K7 mất tiền trong lúc chờ.
- **Nhóm chốt: tách theo ai khởi xướng.** Khách yêu cầu khoá ví của chính mình thì **chủ ví là người duyệt**, bằng một nút + hộp xác nhận nói rõ hậu quả. AI không tự khoá theo suy đoán. Hệ thống hay AI muốn khoá khi khách không yêu cầu thì **analyst duyệt**, đúng HITL của FIN-02. Lý do: khoá là hành động bảo vệ, khoá nhầm không mất tiền, và người có quyền tự nhiên với ví là chủ ví. Rủi ro còn lại: kẻ gian đang cầm máy khách bấm khoá. Hại nhỏ (khách mất giờ, không mất tiền), chấp nhận được.

**Thử mức đã đặt trên 3 kịch bản:**

| Kịch bản | Với mức này AI làm gì · người dùng thấy gì | Cái sai ở §2 có bị chặn? | Còn việc AI làm khi lẽ ra phải hỏi? | Người bình thường có chờ vô ích? |
|---|---|---|---|---|
| K1 bị lừa đòi nhả | `xem_trang_thai` → `giu_giao_dich` (siết) → `chuyen_tu_van_vien` → `soan_ho_so_case`. Khách thấy "tiền vẫn trong ví", không có nút nhả. | **Có**: nhả cấm với AI; xác thực đạt không mở nhóm lừa đảo. | Không. Chuyển người là tự làm, nhưng đó là việc an toàn. | Không áp dụng (không phải người bình thường); khách chờ ~5 phút, chi phí đúng chỗ. |
| K2 gấp thật | `xem_trang_thai` → `yeu_cau_xac_minh`. Đạt thì code cho đi; không đạt thì chuyển ưu tiên. | **Có**: AI không chặn thêm, không vòng vo, không tự nhả. | Không. | Đạt: ~10 giây. **Không đạt: vẫn chờ ~3 phút.** Đây là giá còn lại (§6). |
| K10 bình thường | `xem_trang_thai` → `yeu_cau_xac_minh`, một nút, đạt là đi. | Có (không bị chuyển người thừa). | Không. | Không: không chuyên viên, không câu hỏi. |

**Luật thêm / tool tách:**
1. **Tách `nha_giao_dich` thành `soan_ho_so_case` (AI tự làm) + nút Nhả của chuyên viên (người).** Nhờ đó K1 bị chặn mà chuyên viên vẫn có hồ sơ dùng ngay (FIN-02: giảm ≥40% thời gian ra quyết định).
2. **"AI chỉ siết, không nới"** cho `giu_giao_dich`. Chặn K1, nơi chat có dấu hiệu bị giục mà ML chưa thấy.
3. **Hai nhóm lý do giữ** ("chính chủ" / "người nhận–lừa đảo"); xác thực chỉ mở nhóm đầu. Chặn ca "xác thực khuôn mặt vẫn đúng".
4. **Hết hạn giữ = huỷ**, không bao giờ là nhả.
5. **Trợ lý chỉ nhận nhóm lý do, không nhận ngưỡng.** Chặn K6 bằng dữ liệu, không trông vào việc prompt bảo "đừng nói".

**Mức tự chủ đã đổi lời hứa nào với người dùng.** Nếu AI được nhả, lời hứa sẽ là *"Trợ lý giải quyết giao dịch bị giữ cho bạn ngay trong chat."* Vì nhả bị cấm với AI, lời hứa đổi thành *"Trợ lý chỉ bạn bước nhanh nhất và giữ tiền của bạn ở lại ví; người thật quyết định cho tiền đi."* Lời hứa nhỏ hơn, nhưng là lời hứa giữ được với cả K1 lẫn K2. Câu "**tiền vẫn ở trong ví của bạn**" trên banner là cách nói lời hứa đó ra màn hình.

---

## §5. Prototype

- **File:** [`prototype/index.html`](prototype/index.html), mở bằng trình duyệt. Hướng dẫn bấm và bảng số: [`prototype/README.md`](prototype/README.md).
- **Công cụ:** HTML/JS thuần, dựng bằng coding agent (Claude). Một file, không backend, không đăng nhập.
- **Phần mock:** dữ liệu giao dịch, điểm ML, SHAP, hồ sơ agent, câu trả lời Ca A/B (viết sẵn theo §3); tab "Ca lạ" phân loại tin bằng **từ khoá**, giữ đúng thứ tự ưu tiên an toàn (OTP → dò ngưỡng → bị lừa → chiếm tài khoản → các ý khác, và gộp trả lời khi có nhiều yêu cầu trong một tin).
- **Lời gọi AI thật:** không có.
- **Hai màn hình:** trái là **Chat với ví** (khách), phải là **Màn hình chuyên viên Fraud Ops** (người nhả). Phía dưới là bảng mức tự chủ và bảng chú thích.

**Ảnh đánh số:** `prototype/anh-1-ca-A-bi-lua-doi-nha.png` (K1) · `prototype/anh-2-ca-B-gap-that.png` (K2, xác thực không đạt).

| Số | Thể hiện gì | HAX/PAIR | Mức tự chủ |
|---|---|---|---|
| ① | "Bạn đang chat với trợ lý AI" + làm được / không làm được | G1 · Luật AI Đ.11 | — |
| ② | Nhãn mức rủi ro do hệ thống chấm + câu "mình không biết chắc…" | G2 | — |
| ③ | Ô "Vì sao" không có ngưỡng | G11 · PAIR Explainability | — |
| ④ | Một nút Xác thực (K2) | G10 | `yeu_cau_xac_minh` tự làm |
| ⑤ | Thời gian chờ chuyên viên; "30 phút tự huỷ, tiền ở lại ví" | PAIR Graceful Failure | `giu_giao_dich`, `chuyen_tu_van_vien` tự làm |
| ⑥ | "Mình không bị lừa" → ghi chú vàng trong hồ sơ | G9 | sửa không mở được nhả |
| ⑦ | "Để sau", "Huỷ chuyển, quay lại trợ lý" | G8 | hoàn tác của việc tự làm |
| ⑧ | Điểm ML, SHAP, rule trace, case tương tự, nhãn "đủ bằng chứng / cần xem kỹ" | G11 cho chuyên viên · PAIR Trust | `soan_ho_so_case` tự làm |
| ⑨ | 3 ô kiểm bắt buộc → nút Nhả → hộp "không lấy lại được" | PAIR Feedback & Control · HITL | `nha_giao_dich` cấm với AI, chỉ người |

Bốn mức trên màn hình đúng như §4: **tự làm có hoàn tác** (⑦) · **cần duyệt có xác nhận** (Ca lạ → "Khoá ví ngay…" → hộp xác nhận của chủ ví; ⑨ của chuyên viên) · **đường chuyển người** (⑤) · **việc cấm không có nút**: phía khách không có nút Nhả ở bất kỳ trạng thái nào.

**Quyết định cách hiện độ chắc / bằng chứng:**
- *Đã nghĩ: hiện % rủi ro cho khách* ("Rủi ro 93%"). **Loại** vì khách đang bị lừa không đổi ý vì một con số, còn khách thật thì hoảng; con số cũng giúp kẻ gian dò ngưỡng (K6). *Chọn* nhãn mức bằng chữ + lý do bằng lời thường. Con số đầy đủ (0,93, SHAP) chỉ hiện cho chuyên viên (⑧), người cần nó để quyết.
- *Đã nghĩ: một nút "Nhả" cho khách, kèm hộp cảnh báo đỏ.* **Loại** vì K1 (người bị thao túng bấm qua mọi cảnh báo). Việc cấm thì không có nút.
- *Đã nghĩ: tách màn hình chuyên viên sang trang khác.* **Loại** vì demo cần thấy cùng lúc "khách thấy gì" và "ai quyết nhả, thấy gì". Đặt hai màn hình cạnh nhau làm câu hỏi bắt buộc "ai quyết nhả" hiện ngay trên ảnh.
- *Cách thể hiện không có trong mẫu:* ý phản đối của khách chạy sang hồ sơ chuyên viên theo thời gian thực (⑥), và nút Nhả mở khoá dần theo checklist (⑨).

---

## §6. Tốt là gì

| # | Tiêu chí Pass/Fail (người ngoài chấm được) | K1 | K2 |
|---|---|---|---|
| T1 | **Tiền không rời ví khi chưa có người có thẩm quyền.** Trong toàn bộ hội thoại, AI không gọi `nha_giao_dich`, phía khách không có nút nhả. Giao dịch chỉ đi khi (a) code cho đi sau xác thực đạt ở nhóm "chính chủ", hoặc (b) chuyên viên bấm sau đủ 3 ô kiểm. | **Pass** | **Pass** (đạt: code nhả; không đạt: chuyên viên nhả) |
| T2 | **Không lộ cách phát hiện.** Câu trả lời cho khách không chứa ngưỡng tiền, số lần, tên đặc trưng hay mã luật. Con số duy nhất được phép là số tiền của chính giao dịch, hotline và thời gian chờ. Kiểm bằng cách đọc từng câu. | **Pass** | **Pass** |
| T3 | **Tin đầu tiên của AI có ngay bước tiếp theo bấm được** (nút hành động hoặc đường sang người) và nói rõ trạng thái tiền (còn trong ví / chưa bị trừ). | **Pass** | **Pass** |

**Lỗi còn lại:**
- K2, khi xác thực **không đạt**: khách vẫn chờ chuyên viên ~3 phút ở quầy. Thiết kế chỉ giảm thiệt hại (nói rõ chưa trừ tiền, gợi ý trả bằng cách khác), chưa xoá được.
- K1, khi khách **không nói** "công an" (kẻ gian dặn "nói là chuyển cho con"): lớp phân loại chat không thấy dấu hiệu, và thiết kế phụ thuộc hoàn toàn vào ML có xếp giao dịch vào nhóm "người nhận–lừa đảo" hay không.
- Giả định hàng đợi Fraud Ops trực 24/7 cho giao dịch đang giữ chưa được kiểm với vận hành thật. Nếu không có, luật "hết hạn = huỷ" vẫn giữ an toàn, nhưng K2 ban đêm sẽ chờ lâu.

---

## §7. Demo

**Kịch bản demo 4 phút (mỗi người một phần):**
1. *(Người 1)* Lát cắt trong 30 giây + vì sao K1 và K2 là hai mặt của cùng một quyết định nhả.
2. *(Người 2)* Bấm tab Ca A: câu AI, banner "tiền vẫn trong ví", bấm "Mình không bị lừa", ghi chú hiện bên phải, nút Nhả bị khoá tới khi đủ 3 ô.
3. *(Người 3)* Bấm tab Ca B: xác thực đạt (đi tiếp, không có case), rồi không đạt (case P1 "cần xem kỹ"). Chỉ bảng mức tự chủ: "nhả sai thì ai mất gì, vì vậy cấm với AI".
4. Ca lạ: gõ thẳng vào tab "Ca lạ" để thấy đường đi.

**Ca lạ đã chuẩn bị trước (đi đường nào):** chiếm tài khoản → nút Khoá ví, chủ ví xác nhận + chuyển người · dò ngưỡng → từ chối, vẫn chỉ bước đi tiếp · dán OTP → ẩn tin, cảnh báo, chuyển người · hai yêu cầu (khoá thẻ + hoàn tiền) → tách hai ý, không hứa hoàn · tin mơ hồ → hỏi lại một câu bằng nút · không khớp gì → hỏi lại bằng 3 lựa chọn.

**Ca lạ nhận được ở demo:** *(điền sau buổi demo: ca gì, prototype đã đi đường nào, nhóm có đổi ý gì không)*
