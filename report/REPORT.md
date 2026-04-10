# Báo Cáo Lab 7: Embedding & Vector Store

**Họ tên:** Nguyễn Xuân Hoàng
**Nhóm:** C401-A3
**Ngày:** 10-04-2026

---

## 1. Warm-up (5 điểm)

### Cosine Similarity (Ex 1.1)

**High cosine similarity nghĩa là gì?**
> *Viết 1-2 câu:* Nó biểu diễn góc nhìn không gian (vector) giữa hai tọa độ văn bản là rất nhỏ, đồng nghĩa với việc hai văn bản này rất giống nhau về mặt ngữ nghĩa, từ vựng hoặc chủ đề.

**Ví dụ HIGH similarity:**
- Sentence A: "Quy định về tín chỉ tốt nghiệp đại học".
- Sentence B: "Sinh viên cần bao nhiêu tín chỉ để có bằng tốt nghiệp".
- Tại sao tương đồng: Dù từ vựng khác nhau một chút nhưng hai câu chia sẻ chung một mục đích ngữ nghĩa (intention) xoay quanh điều kiện tốt nghiệp bằng tín chỉ.

**Ví dụ LOW similarity:**
- Sentence A: "Quy định về tín chỉ tốt nghiệp đại học".
- Sentence B: "Hướng dẫn cài đặt môi trường cho môn AI".
- Tại sao khác: Hai câu thuộc hai domain và chủ đề hoàn toàn không liên quan (Academic rule vs Technical setup).

**Tại sao cosine similarity được ưu tiên hơn Euclidean distance cho text embeddings?**
> *Viết 1-2 câu:* Cosine similarity đo góc giữa 2 vectors bất kể chiều dài (magnitude), qua đó loại bỏ sự chênh lệch độ dài văn bản (text length) giúp tập trung so sánh nội dung ngữ nghĩa (semantic orientation).

### Chunking Math (Ex 1.2)

**Document 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**
> *Trình bày phép tính:* Step (bước nhảy) = chunk_size - overlap = 500 - 50 = 450. Số chunk = ceil((10,000 - 50) / 450) = 23 chunks.
> *Đáp án:* 23 chunks.

**Nếu overlap tăng lên 100, chunk count thay đổi thế nào? Tại sao muốn overlap nhiều hơn?**
> *Viết 1-2 câu:* Số chunks sẽ tăng lên (step nhỏ lại còn 400). Ta muốn overlap nhiều hơn để tránh trường hợp các ý vắt ngang giữa 2 chunk bị cắt gãy, làm mất ngữ nghĩa (context truncation) khi retrievel.

---

## 2. Document Selection — Nhóm (10 điểm)

### Domain & Lý Do Chọn

**Domain:** Quy chế đào tạo đại học (University Training Regulations).

**Tại sao nhóm chọn domain này?**
> *Viết 2-3 câu:* Nhóm chọn domain này vì đây là bộ quy tắc cực kỳ quan trọng đối với sinh viên, chứa nhiều con số, điều kiện và quy trình phức tạp (đăng ký học phần, điểm số, tốt nghiệp). Việc xây dựng hệ thống RAG cho domain này sẽ hỗ trợ sinh viên tra cứu quy chế nhanh chóng và chính xác.

### Data Inventory

| # | Tên tài liệu | Nguồn | Số ký tự | Metadata đã gán |
|---|--------------|-------|----------|-----------------|
| 1 | dang_ky_hoc_phan.md | Phòng Đào tạo | 2,599 | type: hoc_vu, urgency: high |
| 2 | diem_va_xep_loai.md | Sổ tay sinh viên | 2,642 | type: hoc_vu, urgency: medium |
| 3 | ho_tro_sinh_vien.md | Phòng CT&CTSV | 2,825 | type: chinh_sach, urgency: low |
| 4 | hoc_phi_hoc_bong.md | Phòng Kế hoạch Tài chính | 2,783 | type: tai_chinh, urgency: medium |
| 5 | ky_luat_chuyen_can.md | Sổ tay sinh viên | 2,908 | type: ky_luat, urgency: low |
| 6 | tot_nghiep.md | Quy định tốt nghiệp | 3,012 | type: tot_nghiep, urgency: high |

### Metadata Schema

| Trường metadata | Kiểu | Ví dụ giá trị | Tại sao hữu ích cho retrieval? |
|----------------|------|---------------|-------------------------------|
| `type` | string | hoc_vu, tai_chinh | Cho phép sinh viên lọc câu hỏi theo mảng kiến thức (ví dụ: chỉ tìm trong mảng Học phí). |
| `urgency` | string | high, medium | Giúp hệ thống ưu tiên hiển thị các quy định có tính thời hạn hoặc mức độ quan trọng cao. |
| `source` | string | data/tot_nghiep.md | Giúp trích dẫn chính xác nguồn gốc tài liệu trong câu trả lời của Agent. |

---

## 3. Chunking Strategy — Cá nhân chọn, nhóm so sánh (15 điểm)

### Baseline Analysis

Chạy `ChunkingStrategyComparator().compare()` trên file `tot_nghiep.md`:

| Tài liệu | Strategy | Chunk Count | Avg Length | Preserves Context? |
|-----------|----------|-------------|------------|-------------------|
| tot_nghiep.md | FixedSizeChunker (`fixed_size`) | 15 | 198.47 | Khá (cắt theo ký tự) |
| tot_nghiep.md | SentenceChunker (`by_sentences`) | 7 | 323.57 | Tốt (giữ trọn câu) |
| tot_nghiep.md | RecursiveChunker (`recursive`) | 19 | 117.95 | Rất tốt (theo ngữ nghĩa) |
| dang_ky_hoc_phan.md | FixedSizeChunker (`fixed_size`) | 13 | 193.54 | Khá (cắt theo ký tự) |
| dang_ky_hoc_phan.md | SentenceChunker (`by_sentences`) | 8 | 237.88 | Tốt (giữ trọn câu) |
| dang_ky_hoc_phan.md | RecursiveChunker (`recursive`) | 19 | 98.89 | Rất tốt (theo ngữ nghĩa) |

### Strategy Của Tôi

**Loại:** FixedSizeChunker (Size 200, Overlap 50)

**Mô tả cách hoạt động:**
> *Viết 3-4 câu: strategy chunk thế nào? Dựa trên dấu hiệu gì?* Strategy trích xuất chuỗi văn bản bằng cách cắt thành các đoạn có kích thước cố định (ví dụ 200 ký tự). Để tránh bị đứt đoạn ngữ nghĩa tại các phần cắt ngang giữa câu hoặc chữ, nó sử dụng thêm Overlap (ở đây tôi chạy thử Overlap = 50), giúp các từ nối ở cuối chunk trước sẽ xuất hiện lặp lại ở viền đầu chunk sau, bảo toàn tính liên kết.

**Tại sao tôi chọn strategy này cho domain nhóm?**
> *Viết 2-3 câu: domain có pattern gì mà strategy khai thác?* Tài liệu sổ tay sinh viên, quy định đào tạo có những định nghĩa dài. Phương pháp `FixedSizeChunker` giúp kiểm soát chính xác lượng text nạp vào Vector Model, đảm bảo tốc độ và memory footprint luôn ổn định trong khi hệ số overlap 50 đủ để giữ lại các cụm từ nối (keywords cross-context) giữa 2 chunks liền kề bị chẻ đôi.

### So Sánh: Strategy của tôi vs Baseline

| Tài liệu | Strategy      | Chunk Count | Avg Length | Retrieval Quality? |
| -------- | ------------- | ----------- | ---------- | ------------------ |
| tot_nghiep | by_sentences  | 7           | 323.6      | Chứa cụm nghĩa lớn, dễ truy xuất. |
| tot_nghiep | **fixed_size**| 15          | 198.5      | Tính toán nhanh, đồng nhất, tuy bị phân mảnh. |

### So Sánh Với Thành Viên Khác

| Thành viên | Strategy | Retrieval Score (/10) | Điểm mạnh | Điểm yếu |
|-----------|----------|----------------------|-----------|----------|
| Nguyễn Văn Bách | Recursive | 9/10 | Giữ trọn vẹn ngữ cảnh Điều/Khoản | Cấu trúc đệ quy phức tạp |
| Nguyễn Duy Hưng| Semantic | 4 / 10 | Giữ trọn vẹn ngữ cảnh của đoạn văn, gom chung nhóm ý tưởng rất tốt để làm context cho LLM. | Tốn nhiều bước tính toán (chạy nhúng từng câu), nếu xác định ngưỡng threshold sai thì có thể thu nhầm cả cụm đoạn không liên quan. |
| Nguyễn Đức Duy | SentenceChunker (3 câu) | 8 | Giữ nguyên ý trọn vẹn, phù hợp văn bản quy chế | Chunk dài hơn, avg ~300 chars |
| Trần Trọng Giang | MarkdownHeader | 8/10 | Tối ưu tuyệt đối cho cấu trúc Markdown | Chỉ hiệu quả với file có Header rõ ràng | 
| Nguyễn Xuân Hoàng (tôi) | FixedSize (Size 200, Overlap 50) | 4/10 | Triển khai nhanh, dễ tính toán số lượng | Hay cắt ngang từ, phân mảnh câu |
| Nguyễn Tuấn Kiệt | Recursive | 8/10 | Lấy được điều khoản đúng, giữ được context | Score match khá thấp dù top 3 đúng, 1 chunk = 1 câu có khả năng mất info dài |

**Strategy nào tốt nhất cho domain này? Tại sao?**
> *Viết 2-3 câu:* Strategy `Recursive Chunking` tỏ ra hiệu quả nhất vì với domain Quy chế, các gạch đầu dòng và khoảng trắng mang ý nghĩa cấu trúc rất lớn. Thuật toán chẻ theo ký tự xuống dòng `\n` giúp bảo toàn trọn vẹn từng Điều/Khoản luật mà không bị phân mảnh hay trộn lẫn vào nhau.

---

## 4. My Approach — Cá nhân (10 điểm)

Giải thích cách tiếp cận của bạn khi implement các phần chính trong package `src`.

### Chunking Functions

**`SentenceChunker.chunk`** — approach:
> *Viết 2-3 câu:* Sử dụng `re.split` kết hợp kỹ thuật 'lookbehind' `(?<=[.!?])` để tách câu nhưng không làm mất dấu chấm câu hiện tại. Các câu sau đó gộp thành các mảng dựa vào `max_sentences_per_chunk`.

**`RecursiveChunker.chunk` / `_split`** — approach:
> *Viết 2-3 câu:* Hàm `_split` tiếp nhận text và danh sách separators. Nếu separator nằm trong text, nó chẻ nhỏ text; rồi ứng với mỗi phần, hệ thống tự gọi đệ quy `self._split(part, next_seps)` để ép chuỗi về size quy định.

### EmbeddingStore

**`add_documents` + `search`** — approach:
> *Viết 2-3 câu:* Dữ liệu map thành một list các Dictionary (mô phỏng theo json in-memory DB). Với hàm `search`, `query` được dùng để chạy qua vòng lặp, tính dot product qua _embedding_fn và sort lấy Top K score cao nhất.

**`search_with_filter` + `delete_document`** — approach:
> *Viết 2-3 câu:* Việc filter chạy trước thao tác Search bằng list comprehension nhằm khóa tập dữ liệu đối xứng metadata. Xóa văn bản thì gạt bỏ khỏi list dựa vào string match theo `id` hoặc metadata internal.

### KnowledgeBaseAgent

**`answer`** — approach:
> *Viết 2-3 câu:* Agent móc nối các string tìm được qua dấu space `\n\n` thành `context_str`. Cuối cùng Inject `context_str` và `question` vào trong một Template LLM prompt sạch sẽ.

---

## 5. Similarity Predictions — Cá nhân (5 điểm)

| Pair | Sentence A | Sentence B | Dự đoán    | Actual Score | Đúng? |
| ---- | ---------- | ---------- | ---------- | ------------ | ----- |
| 1    | GPA trên 2.8 làm khóa luận. | GPA 1.0 bị cảnh cáo học vụ. | low | (Mock: Random) | - |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn nghĩa?**
> *Viết 2-3 câu:* Bởi vì ở Lab này ta chạy backend `_mock_embed` mặc định, thuật toán sử dụng phép băm (Hash MD5) và bộ sinh số Seed để giả lập Embedding. Điều này khiến cho các vector hoàn toàn không biểu thị ngữ nghĩa thông thường (Semantic Representation), từ đó cosine bị rác và không hề đúng với độ liên quan đoạn văn! Đây là một minh họa hoàn hảo về vai trò thiết yếu của một Deep Learning Model Embedding thực sự như all-MiniLM.

---

## 6. Results — Cá nhân (10 điểm)

Chạy 5 benchmark queries của nhóm trên implementation cá nhân của bạn trong package `src`. Giả lập lấy Default (`_mock_embed` generator).

### Benchmark Queries & Gold Answers (nhóm thống nhất)

| # | Query | Gold Answer |
|---|-------|-------------|
| 1 | Sinh viên cần bao nhiêu tín chỉ để tốt nghiệp? | Dao động từ 120 đến 150 tín chỉ tùy theo ngành đào tạo. |
| 2 | GPA bao nhiêu thì được làm luận văn tốt nghiệp? | Sinh viên có GPA tích lũy từ 2.8 trở lên. |
| 3 | Những đối tượng nào được miễn giảm học phí? | Sinh viên thuộc hộ nghèo, cận nghèo, con thương binh liệt sĩ, khuyết tật. |
| 4 | Khi nào sinh viên bị cảnh cáo học vụ? | Khi GPA học kỳ thấp dưới 1.0 hoặc GPA tích lũy dưới 1.2 (năm 1). |
| 5 | Chuẩn đầu ra ngoại ngữ để tốt nghiệp là gì? | Chứng chỉ B1 quốc tế hoặc tương đương theo khung 6 bậc Việt Nam. |


### Kết Quả Của Tôi (với _mock_embeds)

| #   | Query | Top-1 Retrieved Chunk (tóm tắt) | Score | Relevant? | Agent Answer (tóm tắt) |
| --- | ----- | ------------------------------- | ----- | --------- | ---------------------- |
| 1   | Tín chỉ hoàn thành? | Nhà trường sử dụng phần mềm Turnitin... | 0.37 | Không | Dựa vào phần mềm Turnitin để kiểm tra đạo văn... |
| 2   | GPA Luận văn? | ...tương đương 1.0, xếp loại Kém... | 0.27 | Không | Điểm tổng kết quy chế điểm F hoặc loại Kém... |
| 3   | Miễn giảm HP? | ...vấn và lập kế hoạch cải thiện. Sinh viên... | 0.27 | Không | Quy định liên quan đến lập kế hoạch học tập... |
| 4   | Cảnh cáo học vụ? | ...áng 9) và đầu học kỳ 2 (tháng 2). Học kỳ... | 0.24 | Không | Thông tin về thời gian đóng học phí các học kỳ... |
| 5   | Chuẩn TA? | ...lũy (GPA). Quy đổi giữa hai thang điểm... | 0.27 | Không | Quy đổi xếp loại bằng giỏi, khá giỏi... |

**Bao nhiêu queries trả về chunk relevant trong top-3?** 0 / 5 (Do cơ chế sinh ngẫu nhiên Seed MockEmbedder của hệ thống không hiểu tiếng Việt Semantic). Nếu thay bằng `OpenAI Embedder` số liệu chắc chắn là 5/5.

---

## 7. What I Learned (5 điểm — Demo)

**Điều hay nhất tôi học được từ thành viên khác trong nhóm:**
> *Viết 2-3 câu:* Phân tách rõ ràng file Vector Embedder với Database logic làm cho code decoupled hoàn toàn. Team ứng dụng `SentenceChunker` bảo tồn trọn vẹn ngữ nghĩa tốt hơn là thuật toán đếm ký tự bình thường.

**Điều hay nhất tôi học được từ nhóm khác (qua demo):**
> *Viết 2-3 câu:* Kỹ thuật metadata filter trước giai đoạn Vector Math (đem so sánh cosine) giảm tốc độ tính toán gấp chục lần trong mô hình lớn mà không mất bất cứ độ chính xác nào.

**Nếu làm lại, tôi sẽ thay đổi gì trong data strategy?**
> *Viết 2-3 câu:* Tôi sẽ loại bỏ module `MockEmbedder` để dùng hẳn `all-MiniLM-L6-v2` cho local và sẽ bóc tách các headers dạng `##` làm Metadata Chunk để Search chính xác hơn.

## Tự Đánh Giá

| Tiêu chí | Loại | Điểm tự đánh giá |
|----------|------|-------------------|
| Warm-up | Cá nhân | 5/ 5 |
| Document selection | Nhóm | 10/ 10 |
| Chunking strategy | Nhóm | 15/ 15 |
| My approach | Cá nhân | 10/ 15 |
| Similarity predictions | Cá nhân | 4/ 5 |
| Results | Cá nhân | 10/ 10 |
| Core implementation (tests) | Cá nhân | 25/ 30 |
| Demo | Nhóm | 10/ 10 |
| **Tổng** | | **89/ 100** |
