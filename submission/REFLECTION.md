# Reflection — Day 17 (≤ 200 words)

Answer briefly, in your own words. This is graded on reasoning, not length.

1. **The flywheel.** Day 13 emitted agent traces; today you turned them into an
   eval set and DPO pairs that Day 22 will train on. Which step in
   `traces → Bronze → datasets` would break most silently in production if you
   got it wrong — and how would you detect it?

   **Trả lời:** Bước dễ sai nhất một cách âm thầm là **flatten span tree** trong `traces.py`. Nếu logic đệ quy bỏ sót nhánh con lồng sâu, Bronze sẽ thiếu span mà không có lỗi nào được ném ra — pipeline vẫn chạy thành công nhưng eval set và DPO pairs sẽ thiếu dữ liệu từ các lượt phức tạp. Cách phát hiện: so sánh `n_spans` trong Bronze với tổng số span đếm trực tiếp từ file JSON gốc.

2. **Decontamination.** Your run dropped 2 of 3 preference pairs because their
   prompts were in the eval set. What concretely goes wrong if you *skip* this
   step and train on those pairs? How would the lie show up in your metrics?

   **Trả lời:** Model sẽ học thuộc các câu hỏi benchmark thay vì tổng quát hóa. Kết quả là eval score tăng giả tạo — model trông giỏi trên bộ chấm điểm nhưng kém trên câu hỏi mới. Cái bẫy này không có lỗi nào; chỉ phát hiện được khi dùng một **held-out test set hoàn toàn tách biệt** và thấy khoảng cách lớn giữa eval score và test score thực tế.

3. **Point-in-time.** The naive join leaked a future `lifetime_spend` into the
   training row. Describe one feature in a system you know that would be
   dangerous to join without an `ASOF`/point-in-time guard.

   **Trả lời:** Trong hệ thống tín dụng: feature `credit_score` của người dùng. Nếu join theo giá trị mới nhất thay vì giá trị tại thời điểm giao dịch, model sẽ thấy credit score *sau* khi người dùng đã vỡ nợ và được tái cơ cấu — dữ liệu tương lai rò rỉ vào training row, khiến model học sai hoàn toàn về hành vi rủi ro.

4. **Graph vs vector.** From `kg_demo.py`, name one question the knowledge graph
   answers well that flat chunk retrieval (`embed.py`) would struggle with, and
   one where the graph is overkill.

   **Trả lời:** **Graph thắng** ở câu hỏi đa bước: *"Widget giao hàng từ kho nào?"* — cần đi qua hai cạnh (widget → IS_A → accessory → SHIPS_FROM → Hanoi), không một chunk đơn lẻ nào chứa đủ cả hai đầu, flat retrieval không thể bridge. **Vector thắng** ở câu hỏi tìm kiếm ngữ nghĩa mở như *"Sản phẩm nào phù hợp làm quà tặng?"* — không có cạnh rõ ràng trong graph, embedding similarity trên mô tả sản phẩm cho kết quả tốt hơn nhiều.
