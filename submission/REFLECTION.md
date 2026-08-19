# Reflection — Lab 19

**Tên:** Tô Thái Dương
**Cohort:** A20-K3
**Path đã chạy:** lite

---

## Câu hỏi (≤ 200 chữ)

> Trên golden set 50 queries, mode nào thắng ở loại query nào (`exact` /
> `paraphrase` / `mixed`), và tại sao? Khi nào bạn **không** dùng hybrid
> (i.e. khi nào pure BM25 hoặc pure vector là lựa chọn đúng)?

Trên 50 câu hỏi, hybrid có Precision@10 trung bình cao nhất (78,6%), nhưng
không phải lúc nào nó cũng thắng. Với nhóm `exact`, BM25 và hybrid cùng đạt
96,7% vì từ khóa trong câu hỏi xuất hiện trực tiếp trong tài liệu. Ở nhóm
`mixed`, hybrid đạt 100%, cao hơn BM25 (97,0%) và vector (98,5%), do nó kết hợp
được cả tín hiệu từ khóa lẫn ngữ nghĩa.

Điều mình không ngờ là ở nhóm `paraphrase`, BM25 vẫn cao nhất (33,3%), còn
vector chỉ đạt 24,0%. Theo mình, nguyên nhân chính là model
`bge-small-en-v1.5` thiên về tiếng Anh nên chưa biểu diễn tốt câu diễn đạt lại
bằng tiếng Việt. Mình sẽ dùng BM25 thuần khi truy vấn chứa mã, tên riêng hoặc
thuật ngữ chính xác để giảm chi phí và độ trễ. Vector thuần phù hợp hơn khi từ
khóa giữa câu hỏi và tài liệu khác nhau nhiều, nhưng chỉ khi model embedding
thực sự phù hợp với ngôn ngữ và dữ liệu.

---

## Điều ngạc nhiên nhất khi làm lab này

Mình khá bất ngờ vì semantic search không tự động tốt hơn BM25. Kết quả cho
thấy chọn đúng model embedding quan trọng không kém việc chọn thuật toán tìm kiếm.

---

## Bonus challenge

- [ ] Đã làm bonus (xem `bonus/`)
- [ ] Pair work với: Không
