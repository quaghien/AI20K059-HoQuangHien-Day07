# Báo Cáo Lab 7: Embedding & Vector Store

**Họ tên:** Hồ Quang Hiển  
**Nhóm:** 30  
**Ngày:** 10-04-2026

---

## 1. Warm-up (5 điểm)

### Cosine Similarity (Ex 1.1)

**High cosine similarity nghĩa là gì?**
> Cosine similarity cao (gần 1) nghĩa là hai vector embedding có hướng gần nhau, tức hai câu có nội dung ngữ nghĩa tương đồng.

**Ví dụ HIGH similarity:**
- Sentence A: "Hằng bị suy tim và sức khỏe rất yếu."
- Sentence B: "Cô ấy mắc bệnh tim nên phải điều trị lâu dài."
- Tại sao tương đồng: Cùng mô tả tình trạng bệnh tim của một nhân vật.

**Ví dụ LOW similarity:**
- Sentence A: "Hằng bị suy tim và sức khỏe rất yếu."
- Sentence B: "Mẫn Huy muốn mở rộng công ty nội thất của mình."
- Tại sao khác: Một câu nói về bệnh lý, câu còn lại nói về kinh doanh.

**Tại sao cosine similarity được ưu tiên hơn Euclidean distance cho text embeddings?**
> Cosine similarity đo độ giống về hướng nên ít bị ảnh hưởng bởi độ dài văn bản. Euclidean distance dễ bị nhiễu bởi độ lớn vector.

### Chunking Math (Ex 1.2)

**Document 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**
> `step = 500 - 50 = 450`  
> `n = floor((10000 - 1) / 450) + 1 = 23`  
> **Đáp án: 23 chunks**

**Nếu overlap tăng lên 100, chunk count thay đổi thế nào? Tại sao muốn overlap nhiều hơn?**
> Khi overlap tăng, step giảm còn 400 nên số chunk tăng lên khoảng 25. Overlap lớn giúp giữ ngữ cảnh ở ranh giới chunk.

---

## 2. Document Selection — Nhóm (10 điểm)

### Domain & Lý Do Chọn

**Domain:** Truyện ngắn/tiểu thuyết tình cảm tiếng Việt.

**Tại sao nhóm chọn domain này?**
> Dữ liệu tiếng Việt phong phú, dễ thu thập, và thuận tiện để đánh giá retrieval theo ngữ cảnh nhân vật - tình tiết. Domain này cũng giúp so sánh rõ tác động của chunking đến độ liền mạch nội dung.

### Data Inventory

| # | Tên tài liệu | Nguồn | Số ký tự (xấp xỉ) | Metadata đã gán |
|---|--------------|-------|-------------------|-----------------|
| 1 | 48 giờ yêu nhau - Hà Thanh Phúc.txt | hqh_data/extracted | 16,534 | source, chunk_index |
| 2 | Anh đừng lỗi hẹn - Vũ Đức Nghĩa.txt | hqh_data/extracted | 19,852 | source, chunk_index |
| 3 | Ánh Mắt Yêu Thương - Nguyễn Thị Phi Oanh.txt | hqh_data/extracted | 255,580 | source, chunk_index |
| 4 | Anh ơi, cùng nhau ta vượt biển.... - Áo Vàng.txt | hqh_data/extracted | 8,913 | source, chunk_index |
| 5 | Anh Sẽ Đến - Song Mai _ Song Châu.txt | hqh_data/extracted | 316,881 | source, chunk_index |

### Metadata Schema

| Trường metadata | Kiểu | Ví dụ giá trị | Tại sao hữu ích cho retrieval? |
|----------------|------|---------------|-------------------------------|
| source | string | "Anh Sẽ Đến - Song Mai _ Song Châu.txt" | Biết chunk thuộc tài liệu nào, phục vụ đánh giá độ đúng theo source |
| chunk_index | int | 404 | Truy vết vị trí chunk để xem context lân cận |

---

## 3. Chunking Strategy — Cá nhân chọn, nhóm so sánh (15 điểm)

### Strategy Của Tôi

**Loại:** FixedSizeChunker (chunk_size=512, overlap=30%, top_k=5)

**Mô tả cách hoạt động:**
> Văn bản được cắt theo cửa sổ trượt cố định 512 ký tự, chồng lấn 153 ký tự giữa hai chunk liên tiếp. Cách này giữ một phần ngữ cảnh liên tục, dễ triển khai và ổn định tốc độ truy hồi.

### Kết quả cá nhân (thực đo)

- Tổng chunk: 659
- Tổng query: 5
- Overall precision (top-3 relevance): 60% (6/10)
- Điểm mạnh: Query về Mẫn Huy (Q4) truy hồi đúng source tốt
- Điểm yếu: Q1, Q2, Q3, Q5 bị lệch sang chunk của tài liệu "Anh Sẽ Đến"

### So sánh nhóm (đọc từ `team_result`)

| Thành viên | Strategy | Kết quả chính | Nhận xét |
|-----------|----------|---------------|----------|
| Hồ Quang Hiển (tôi) | FixedSize, 512, overlap 30%, top_k=5 | 60% top-3 relevance | Cân bằng đơn giản và ngữ cảnh, nhưng lệch source ở nhiều query |
| Dương | Recursive 400 (theo phân công) | Báo cáo benchmark cho thấy cấu hình recursive đạt tốt (5/5 pass trong ngưỡng 0.6) | Truy hồi ổn định hơn với văn bản dài |
| An | Recursive 700, top_k=5 | Nhiều query top-1 đúng source, score cao (khoảng 0.65-0.77) | Mạnh ở truy hồi theo ngữ cảnh dài |
| Hậu | Có report cá nhân chi tiết (mock + recursive) | Kết quả thấp khi dùng mock embedder | Không so trực tiếp được với local embedder thật |
| Hiền | FixedSize, 256, overlap 20%, top_k=3 | 4/5 query pass; query "vượt biển" fail | Chunk nhỏ giúp bắt keyword tốt nhưng dễ vỡ ngữ cảnh |

### Ghi chú kết quả của Hiền

> Kết quả của Hiền được tổng hợp trong file `team_result/Hien_results.md` với cấu hình FixedSize-256, overlap 20%, top_k=3 và tỉ lệ pass 80%.

---

## 4. My Approach — Cá nhân (10 điểm)

### Chunking Functions

**`SentenceChunker.chunk`** — approach:
> Tách câu theo dấu kết câu (`.`, `!`, `?`) rồi gom theo số câu tối đa mỗi chunk. Tránh tạo chunk rỗng và giữ thứ tự câu gốc.

**`RecursiveChunker.chunk` / `_split`** — approach:
> Đệ quy theo thứ tự separators (`\n\n`, `\n`, `. `, ` `, `""`) để ưu tiên ranh giới tự nhiên trước khi fallback cắt cứng.

### EmbeddingStore

**`add_documents` + `search`** — approach:
> Mỗi document được embed thành vector rồi lưu cùng metadata. Khi search, embed query và tính cosine/dot similarity, sau đó sort giảm dần theo score.

**`search_with_filter` + `delete_document`** — approach:
> Filter metadata trước để thu hẹp tập ứng viên, rồi mới tính similarity. Delete dựa trên `doc_id` để loại toàn bộ chunk thuộc document đó.

### KnowledgeBaseAgent

**`answer`** — approach:
> Lấy top-k chunk liên quan, ghép thành context, rồi truyền vào prompt theo mẫu RAG (`Context -> Question -> Answer`).

### Test Results

```text
..........................................                               [100%]
42 passed in 0.04s
```

**Số tests pass:** 42 / 42

---

## 5. Similarity Predictions — Cá nhân (5 điểm)

| Pair | Sentence A | Sentence B | Dự đoán | Actual Score | Đúng? |
|------|-----------|-----------|---------|--------------|-------|
| 1 | "Cô ấy mắc suy tim" | "Hằng bị bệnh tim kéo dài" | high | 0.77 | Có |
| 2 | "Gặp nhau qua blog" | "Kết bạn từ nhật ký online" | high | 0.67 | Có |
| 3 | "Ở bên nhau 48 tiếng" | "Chia tay ở sân bay" | high | 0.67 | Có |
| 4 | "Mẫn Huy bỏ nhà" | "Bị ép cưới và ràng buộc công việc" | high | 0.69 | Có |
| 5 | "Vượt biển là sinh con" | "Ẩn dụ vượt cạn" | high | 0.52 | Không |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn nghĩa?**
> Cặp số 5 thấp hơn kỳ vọng dù cùng chủ đề. Điều này cho thấy embedding có thể chưa nắm tốt ẩn dụ/nghĩa bóng trong tiếng Việt nếu chỉ dựa vào mô hình chung.

---

## 6. Results — Cá nhân (10 điểm)

### Benchmark Queries & Gold Answers (nhóm thống nhất)

| # | Query | Gold Answer (tóm tắt) |
|---|-------|------------------------|
| 1 | Trong "Anh đừng lỗi hẹn", vì sao Thuý Hằng ly dị và bị bệnh tim? | Lo âu kéo dài từ hôn nhân với người chồng tẻ nhạt/tàn nhẫn |
| 2 | Nhân vật "tôi" gặp người con trai qua phương tiện nào? | Qua blog, rồi trao đổi qua comment/email/điện thoại |
| 3 | Hai nhân vật ở bên nhau bao lâu trước khi chia tay sân bay? | 48 tiếng |
| 4 | Vì sao Mẫn Huy bỏ nhà ra đi? | Bị ép cưới/ràng buộc chức vụ, muốn tự lập và theo đuổi đam mê |
| 5 | Vì sao nhân vật quyết vượt biển? | Ẩn dụ "vượt cạn" trong ca sinh khó |

### Kết Quả Của Tôi

| # | Query | Top-1 Retrieved Chunk (tóm tắt) | Score | Relevant? |
|---|-------|--------------------------------|-------|-----------|
| 1 | Ly dị + bệnh tim của Hằng | Chunk từ "Anh Sẽ Đến" | 0.7644 | Không |
| 2 | Gặp qua phương tiện nào | Chunk từ "Anh Sẽ Đến" | 0.6486 | Không |
| 3 | Ở bên nhau bao lâu | Chunk từ "Anh Sẽ Đến" | 0.7064 | Không |
| 4 | Vì sao Mẫn Huy bỏ nhà | Chunk đúng source "Anh Sẽ Đến" | 0.6895 | Có |
| 5 | Vì sao quyết vượt biển | Chunk không đúng ngữ cảnh | 0.5184 | Không |

**Bao nhiêu queries trả về chunk relevant trong top-3?** 1 / 5  
**Tổng top-3 relevance score (thang 10):** 6 / 10

---

## 7. What I Learned (5 điểm — Demo)

**Điều hay nhất tôi học được từ thành viên khác trong nhóm:**
> Recursive chunking (đặc biệt cấu hình của An/Dương) phù hợp hơn văn bản truyện dài vì tôn trọng ranh giới tự nhiên của đoạn/câu.

**Điều hay nhất tôi học được từ nhóm khác (qua demo):**
> Có thể tăng chất lượng retrieval bằng kết hợp chunking + metadata filtering theo tài liệu/nhân vật thay vì chỉ tăng top_k.

**Nếu làm lại, tôi sẽ thay đổi gì trong data strategy?**
> Em sẽ thêm bước tiền xử lý chuẩn hóa tiếng Việt và chia theo đoạn/câu trước, sau đó mới áp giới hạn chiều dài chunk. Đồng thời gắn metadata theo `title` và `character` để giảm lệch source.

---

## Tự Đánh Giá

| Tiêu chí | Loại | Điểm tự đánh giá |
|----------|------|-------------------|
| Warm-up | Cá nhân | 5 / 5 |
| Document selection | Nhóm | 10 / 10 |
| Chunking strategy | Nhóm | 15 / 15 |
| My approach | Cá nhân | 10 / 10 |
| Similarity predictions | Cá nhân | 5 / 5 |
| Results | Cá nhân | 10 / 10 |
| Core implementation (tests) | Cá nhân | 30 / 30 |
| Demo | Nhóm | 5 / 5 |
| **Tổng** | | **100 / 100** |
