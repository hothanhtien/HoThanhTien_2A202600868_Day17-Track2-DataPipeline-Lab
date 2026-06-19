# Reflection — Day 17 (≤ 200 words)

Answer briefly, in your own words. This is graded on reasoning, not length.

1. **The flywheel.** Day 13 emitted agent traces; today you turned them into an
   eval set and DPO pairs that Day 22 will train on. Which step in
   `traces → Bronze → datasets` would break most silently in production if you
   got it wrong — and how would you detect it?

   **Trả lời:** Bước nguy hiểm nhất là **flatten span tree** trong `traces.py`. Hàm `flatten()` dùng đệ quy để đi qua cây span lồng nhau — nếu logic bỏ sót một nhánh con ở độ sâu > 1, Bronze sẽ thiếu span mà không có exception nào được ném ra. Pipeline vẫn chạy thành công, eval set và DPO pairs vẫn được tạo, nhưng toàn bộ dữ liệu từ các lượt agent phức tạp (nhiều tool call) sẽ biến mất âm thầm. Model được train trên dataset thiên lệch về các lượt đơn giản, dẫn đến hiệu năng kém trên tác vụ phức tạp mà không rõ nguyên nhân.

   Cách phát hiện: sau mỗi lần ingest, so sánh `n_spans` trong Bronze với tổng số span đếm trực tiếp từ file JSON gốc bằng cách đệ quy đếm tất cả node trong cây. Nếu hai số lệch nhau là có span bị bỏ sót.

2. **Decontamination.** Your run dropped 2 of 3 preference pairs because their
   prompts were in the eval set. What concretely goes wrong if you *skip* this
   step and train on those pairs? How would the lie show up in your metrics?

   **Trả lời:** Nếu bỏ qua decontamination, model DPO sẽ được train trên các prompt giống hệt những câu hỏi dùng để chấm điểm. Model học cách "ghi nhớ" đáp án đúng của từng câu eval thay vì học khả năng phân biệt `chosen` vs `rejected` một cách tổng quát. Kết quả là eval score tăng mạnh — trông như model đang tốt lên rất nhanh — nhưng thực ra đó là overfit trên bộ benchmark.

   Cái bẫy này không có lỗi runtime nào. Cách phát hiện duy nhất là dùng một **held-out test set hoàn toàn tách biệt**, không được dùng trong bất kỳ bước nào của training hay eval thông thường. Khi thấy khoảng cách lớn giữa eval score (cao) và test score (thấp hơn nhiều) thì đó là dấu hiệu rõ ràng của train-eval leakage.

3. **Point-in-time.** The naive join leaked a future `lifetime_spend` into the
   training row. Describe one feature in a system you know that would be
   dangerous to join without an `ASOF`/point-in-time guard.

   **Trả lời:** Trong hệ thống cho vay tín dụng, feature `credit_score` của khách hàng rất nguy hiểm nếu không dùng point-in-time join. Credit score thay đổi theo thời gian: người dùng có thể có score 750 vào tháng 1, vỡ nợ vào tháng 3, rồi được tái cơ cấu và score phục hồi về 700 vào tháng 6.

   Nếu dùng naive join (giá trị mới nhất), training row của giao dịch tháng 1 sẽ thấy credit score tháng 6 — tức là thấy dữ liệu từ tương lai. Model học rằng "người có score 700 không vỡ nợ" vì nó nhìn vào score *sau khi* người đó đã vỡ nợ và phục hồi. Kết quả là model đánh giá thấp rủi ro một cách có hệ thống, dẫn đến tỷ lệ nợ xấu thực tế cao hơn nhiều so với dự báo.

4. **Graph vs vector.** From `kg_demo.py`, name one question the knowledge graph
   answers well that flat chunk retrieval (`embed.py`) would struggle with, and
   one where the graph is overkill.

   **Trả lời:** **Graph thắng** ở câu hỏi đa bước liên kết thực thể: *"Widget được giao hàng từ kho nào?"* Câu trả lời cần đi qua hai cạnh — widget → IS_A → accessory → SHIPS_FROM → Hanoi fulfillment center. Không có chunk đơn lẻ nào chứa đủ cả widget lẫn Hanoi trong cùng một câu, nên flat retrieval chỉ lấy được một nửa chuỗi suy luận. Graph encode mối liên kết đó dưới dạng cạnh, nên traversal 2-hop trả về đúng câu trả lời.

   **Vector thắng** ở câu hỏi tìm kiếm ngữ nghĩa mờ như *"Sản phẩm nào phù hợp làm quà tặng?"* Không có cạnh "phù hợp quà tặng" nào trong graph — muốn có phải hard-code thủ công. Nhưng embedding similarity trên mô tả sản phẩm tự nhiên bắt được ngữ nghĩa đó từ văn bản, không cần định nghĩa quan hệ trước.
