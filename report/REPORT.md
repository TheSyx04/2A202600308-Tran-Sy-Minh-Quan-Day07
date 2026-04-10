# Báo Cáo Lab 7: Embedding & Vector Store

**Họ tên:** Trần Sỹ Minh Quân
**Nhóm:** 32
**Ngày:** 10/4/2026

---

## 1. Warm-up (5 điểm)

### Cosine Similarity (Ex 1.1)

**High cosine similarity nghĩa là gì?**
> High cosine similarity nghĩa là hai đoạn văn có hướng vector embedding rất gần nhau, tức là nội dung và ngữ nghĩa gần tương tự nhau. Giá trị càng gần 1 thì mức độ tương đồng ngữ nghĩa càng cao, còn càng gần -1 thì ngược lại.

**Ví dụ HIGH similarity:**
- Sentence A: Khách hàng có thể đổi trả sản phẩm trong vòng 30 ngày kể từ ngày mua.
- Sentence B: Chính sách của cửa hàng cho phép hoàn trả hàng trong 30 ngày sau khi mua.
- Tại sao tương đồng: Cả hai câu đều diễn đạt cùng một chính sách là hoàn trả sản phẩm trong 30 ngày, chỉ khác nhau cách dùng từ.

**Ví dụ LOW similarity:**
- Sentence A: Hệ thống cần bật xác thực hai lớp để tăng bảo mật tài khoản.
- Sentence B: Cách làm bánh chuối gồm bước trộn bột, nướng và để nguội.
- Tại sao khác: Hai câu thuộc hai chủ đề hoàn toàn khác nhau nên ngữ nghĩa không liên quan đến nhau.

**Tại sao cosine similarity được ưu tiên hơn Euclidean distance cho text embeddings?**
> Cosine similarity tập trung vào hướng của vector (quan hệ ngữ nghĩa) thay vì độ lớn tuyệt đối, nên ổn định hơn khi so sánh văn bản có độ dài khác nhau. Trong embedding, hướng thường phản ánh nghĩa tốt hơn khoảng cách Euclidean.

### Chunking Math (Ex 1.2)

**Document 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**
> *Trình bày phép tính:* num_chunks = (doc_length - overlap) / (chunk_size - overlap) = (10000 - 50) / (500 - 50) = 22.11 => làm tròn lên 23 chunks
> *Đáp án:* 23 chunks

**Nếu overlap tăng lên 100, chunk count thay đổi thế nào? Tại sao muốn overlap nhiều hơn?**
> Khi overlap tăng lên 100: num_chunks = (10000 - 100) / (500 - 100) = 24.75 => làm tròn lên 25 chunks. Overlap nhiều hơn giúp giữ ngữ cảnh tốt hơn ở ranh giới giữa các chunk, giảm mất ý khi thông tin nằm ở hai đoạn liền.

---

## 2. Document Selection — Nhóm (10 điểm)

### Domain & Lý Do Chọn

**Domain:** Customer support knowledge base / help-center documentation

**Tại sao nhóm chọn domain này?**
> Nhóm chọn domain customer support vì tài liệu dạng FAQ và troubleshooting có cấu trúc rõ ràng và phù hợp với retrieval. Bộ tài liệu này cũng đủ đa dạng để thử các câu hỏi về account, password, billing, refund, rate limit, và cả tài liệu nội bộ cần metadata filtering. Ngoài ra, domain này gần với bài toán RAG thực tế, đó là bài tìm đúng hướng dẫn và tránh lấy nhầm tài liệu internal cho người dùng cuối.

### Data Inventory

| # | Tên tài liệu | Nguồn | Số ký tự | Metadata đã gán |
|---|--------------|-------|----------|-----------------|
| 1 | `account_email_change.md` | OpenAI Help Center | 1999 | `doc_id`, `title`, `category=account`, `audience=customer`, `language=en`, `source`, `last_updated`, `sensitivity=public` |
| 2 | `password_reset_help.md` | OpenAI Help Center | 1815 | `doc_id`, `title`, `category=password`, `audience=customer`, `language=en`, `source`, `last_updated`, `sensitivity=public` |
| 3 | `billing_renewal_failure.md` | OpenAI Help Center | 1789 | `doc_id`, `title`, `category=billing`, `audience=customer`, `language=en`, `source`, `last_updated`, `sensitivity=public` |
| 4 | `refund_request_guide.md` | OpenAI Help Center | 1952 | `doc_id`, `title`, `category=refund`, `audience=customer`, `language=en`, `source`, `last_updated`, `sensitivity=public` |
| 5 | `service_limit_429.md` | OpenAI Help Center | 1867 | `doc_id`, `title`, `category=service_limit`, `audience=customer`, `language=en`, `source`, `last_updated`, `sensitivity=public` |
| 6 | `internal_escalation_playbook.md` | Internal support / handbook reference | 1944 | `doc_id`, `title`, `category=escalation`, `audience=internal_support`, `language=en`, `source`, `last_updated`, `sensitivity=internal` |

### Metadata Schema

| Trường metadata | Kiểu | Ví dụ giá trị | Tại sao hữu ích cho retrieval? |
|----------------|------|---------------|-------------------------------|
| `doc_id` | string | `kb_refund_001` | Định danh duy nhất cho mỗi tài liệu, hữu ích khi quản lý, debug hoặc xóa tài liệu khỏi vector store |
| `title` | string | `How to Request a Refund for a ChatGPT Subscription` | Giúp nhận diện nhanh nội dung tài liệu và trình bày nguồn rõ ràng trong kết quả retrieval |
| `category` | string | `account`, `password`, `billing`, `refund`, `service_limit`, `escalation` | Cho phép lọc theo chủ đề để tăng precision, nhất là khi query thuộc một mảng hỗ trợ cụ thể |
| `audience` | string | `customer`, `internal_support` | Rất quan trọng để tránh trả tài liệu nội bộ cho người dùng cuối và hỗ trợ metadata filtering |
| `language` | string | `en` | Hữu ích khi sau này mở rộng sang tài liệu đa ngôn ngữ hoặc cần giới hạn theo ngôn ngữ người hỏi |
| `source` | string | `https://help.openai.com/en/articles/...` | Giúp truy vết nguồn gốc tài liệu và kiểm tra độ tin cậy của câu trả lời |
| `last_updated` | string | `2026-04-10` | Hữu ích nếu sau này cần ưu tiên tài liệu mới hơn hoặc theo dõi độ tươi của dữ liệu |
| `sensitivity` | string | `public`, `internal` | Hỗ trợ kiểm soát truy cập và giảm nguy cơ retrieve nhầm tài liệu nhạy cảm |

---

## 3. Chunking Strategy — Cá nhân chọn, nhóm so sánh (15 điểm)

### Baseline Analysis

Chạy `ChunkingStrategyComparator().compare()` trên 2-3 tài liệu:

| Tài liệu | Strategy | Chunk Count | Avg Length | Preserves Context? |
|-----------|----------|-------------|------------|-------------------|
| `account_email_change.md` | FixedSizeChunker (`fixed_size`) | 5 | 382.2 | Giữ overlap tốt nhưng có thể cắt giữa ý |
| `account_email_change.md` | SentenceChunker (`by_sentences`) | 11 | 155.8 | Tốt về mạch câu nhưng chunk hơi nhỏ và rời rạc |
| `account_email_change.md` | RecursiveChunker (`recursive`) | 6 | 286.8 | Cân bằng giữa độ dài và ngữ cảnh |
| `refund_request_guide.md` | FixedSizeChunker (`fixed_size`) | 4 | 445.5 | Giữ ngữ cảnh dài tốt nhưng có thể cắt section không tự nhiên |
| `refund_request_guide.md` | SentenceChunker (`by_sentences`) | 7 | 233.4 | Dễ đọc, nhưng đôi lúc tách rời các bullet liên quan |
| `refund_request_guide.md` | RecursiveChunker (`recursive`) | 5 | 327.8 | Khá cân bằng, giữ được section logic |
| `internal_escalation_playbook.md` | FixedSizeChunker (`fixed_size`) | 4 | 441.0 | Giữ nội dung dài, dễ dính cả phần không liên quan |
| `internal_escalation_playbook.md` | SentenceChunker (`by_sentences`) | 5 | 324.2 | Rõ ràng theo câu, phù hợp tài liệu checklist |
| `internal_escalation_playbook.md` | RecursiveChunker (`recursive`) | 5 | 324.2 | Ổn định, giữ được cụm ý theo section nội bộ |

### Strategy Của Tôi

**Loại:** `RecursiveChunker` (`chunk_size=420`)

**Mô tả cách hoạt động:**
> Tôi dùng `RecursiveChunker` để tách theo thứ tự separator ưu tiên từ mạnh đến yếu (xuống dòng kép, xuống dòng, dấu câu, khoảng trắng). Nếu đoạn còn quá dài, thuật toán tiếp tục đệ quy với separator mức thấp hơn thay vì cắt cứng ngay từ đầu. Cách này giúp chunk thường bám theo ranh giới tự nhiên của tài liệu support (section, bullet, câu), giảm nguy cơ đứt ý. Tôi đặt `chunk_size=420` để chunk không quá ngắn nhưng vẫn đủ nhỏ cho retrieval.

**Tại sao tôi chọn strategy này cho domain nhóm?**
> Tài liệu help-center có cấu trúc rõ theo heading, bullet, và các bước xử lý nên tách đệ quy theo separator sẽ hợp hơn cắt độ dài cố định. So với tách theo câu thuần túy, cách này thường giữ được cụm hướng dẫn hoàn chỉnh trong cùng một chunk. Với domain customer support, chunk giữ mạch thao tác tốt sẽ hữu ích cho truy vấn dạng "làm gì tiếp theo".

**Code snippet (nếu custom):**
```python
# Không dùng custom strategy ở phase này.
# Strategy cá nhân: RecursiveChunker(chunk_size=420)
```

### So Sánh: Strategy của tôi vs Baseline

| Tài liệu | Strategy | Chunk Count | Avg Length | Retrieval Quality? |
|-----------|----------|-------------|------------|--------------------|
| `refund_request_guide.md` | best baseline = `fixed_size` | 4 | 445.5 | Top-1 thường chứa chunk dài, nhiều ngữ cảnh nhưng đôi lúc nhiễu |
| `refund_request_guide.md` | **của tôi** = `recursive (420)` | 5 | 327.8 | Chunk gọn hơn, dễ map theo section; top-3 vẫn có độ phủ ổn |

### So Sánh Với Thành Viên Khác

| Thành viên | Strategy | Retrieval Score (/10) | Điểm mạnh | Điểm yếu |
|-----------|----------|----------------------|-----------|----------|
| Tôi | `RecursiveChunker` (`chunk_size=420`) | 7/10 | Giữ được ngữ cảnh theo section và bullet khá tốt, hợp tài liệu support có cấu trúc rõ ràng | Nếu query quá mơ hồ hoặc quá ngắn thì đôi lúc chunk top-1 vẫn lệch sang tài liệu gần nghĩa hơn |
| Ngô Quang Tăng | `SentenceChunker` (`3 sent/chunk`) | 8.9/10 | Giữ instructions trọn vẹn, semantic units | Chunks không đều |
| Vũ Đức Minh | `RecursiveChunker` (`chunk_size=400`) | 6/10 | Giữ context tốt, phù hợp với tài liệu có section, steps và internal notes | Tạo nhiều chunk hơn và chưa vượt trội rõ rệt về điểm số khi chỉ dùng `_mock_embed` |
| Nguyễn Thế Anh | `FixedSizeChunker` | 8/10 | Giữ ngữ cảnh liên tục, chuẩn hóa chunk | Có thể cắt giữa câu nếu câu dài hơn 500 ký tự |
| Phạm Minh Khôi | `RecursiveChunker` | 8.5/10 | Giữ được ngữ cảnh theo section và bullet tốt, phù hợp cấu trúc doc. | Đôi lúc tạo ra chunk hơi nhiều, cần cấu hình độ dài cẩn thận. |

**Strategy nào tốt nhất cho domain này? Tại sao?**
> Trên bộ dữ liệu nhóm, `SentenceChunker` là strategy tốt nhất cho domain customer support vì đạt điểm retrieval cao nhất (8.9/10) và giữ các bước hướng dẫn theo semantic unit rõ ràng. `RecursiveChunker` vẫn là phương án cân bằng, phù hợp khi cần giữ cấu trúc section và bullet ổn định trên nhiều loại tài liệu. `FixedSizeChunker` dễ chuẩn hóa nhưng có rủi ro cắt giữa ý, nên kém tối ưu hơn cho các truy vấn cần instruction liền mạch.

---

## 4. My Approach — Cá nhân (10 điểm)

Giải thích cách tiếp cận của bạn khi implement các phần chính trong package `src`.

### Chunking Functions

**`SentenceChunker.chunk`** — approach:
> Tôi tách câu bằng regex `(?<=[.!?])\s+`. Sau khi tách, tôi dùng hàm `strip()` với từng câu để loại bỏ khoảng trắng thừa. Tiếp theo, tôi gom lại theo nhóm `max_sentences_per_chunk` để tạo chunk cân bằng, tránh chunk quá vụn.

**`RecursiveChunker.chunk` / `_split`** — approach:
> Hàm `chunk()` đóng vai trò điều phối: kiểm tra input rỗng, chọn danh sách separator, rồi gọi `_split()`. Base case là: chuỗi rỗng thì trả `[]`, chuỗi ngắn thì trả luôn `[current_text]`, và khi hết separator thì cắt theo `chunk_size`. Với mỗi separator, tôi ưu tiên ghép các phần vào một buffer để giữ ngữ cảnh, nếu quá dài thì đệ quy xuống separator mức thấp hơn.

### EmbeddingStore

**`add_documents` + `search`** — approach:
> Mỗi `Document` được chuẩn hóa thành một record gồm `id`, `content`, `metadata`, `embedding`,và tôi thêm `metadata.doc_id` để tiện cho việc xóa và lọc sau này. Hàm `add_documents()` tạo embedding cho từng tài liệu rồi lưu vào in-memory store, đồng thời ghi sang ChromaDB nếu có sẵn. Hàm `search()` embed query và tính điểm tương đồng bằng dot product giữa vector query và vector đã lưu, sau đó sắp xếp giảm dần theo `score` và lấy top-k.

**`search_with_filter` + `delete_document`** — approach:
> Tôi lọc metadata trước rồi mới search để đảm bảo kết quả đúng filter. Hàm `search_with_filter()` giữ các record thỏa tất cả cặp key-value trong `metadata_filter`, sau đó mới chạy tính điểm và xếp hạng. Hàm `delete_document()` xóa toàn bộ record có `metadata['doc_id'] == doc_id`, trả về `True/False` tùy vào việc có xóa được hay không, và nếu đang dùng ChromaDB thì xóa đồng bộ theo `ids` tương ứng.

### KnowledgeBaseAgent

**`answer`** — approach:
> Tôi lấy top-k chunk liên quan từ store trước, rồi đánh số từng chunk để tạo phần ngữ cảnh rõ ràng. Prompt gồm 3 phần chính: vai trò trợ lý, khối context đã truy xuất, và câu hỏi người dùng. Sau đó gọi `llm_fn(prompt)` để sinh câu trả lời; nếu không có chunk phù hợp thì chèn thông báo "No relevant context found." để tránh model tự bịa thông tin.

### Test Results

```
=============================== test session starts ================================
platform win32 -- Python 3.11.9, pytest-9.0.3, pluggy-1.6.0 -- c:\Users\minhq\OneDrive\Tài liệu\Vin\2A202600308_lab_1\2A202600308-Tran-Sy-Minh-Quan-Day07\.venv\Scripts\python.exe
cachedir: .pytest_cache
rootdir: C:\Users\minhq\OneDrive\Tài liệu\Vin\2A202600308_lab_1\2A202600308-Tran-Sy-Minh-Quan-Day07
collected 42 items                                                                  

tests/test_solution.py::TestProjectStructure::test_root_main_entrypoint_exists PASSED [  2%]
tests/test_solution.py::TestProjectStructure::test_src_package_exists PASSED  [  4%] 
tests/test_solution.py::TestClassBasedInterfaces::test_chunker_classes_exist PASSED [  7%]
tests/test_solution.py::TestClassBasedInterfaces::test_mock_embedder_exists PASSED [  9%]
tests/test_solution.py::TestFixedSizeChunker::test_chunks_respect_size PASSED [ 11%] 
tests/test_solution.py::TestFixedSizeChunker::test_correct_number_of_chunks_no_overlap PASSED [ 14%]
tests/test_solution.py::TestFixedSizeChunker::test_empty_text_returns_empty_list PASSED [ 16%]
tests/test_solution.py::TestFixedSizeChunker::test_no_overlap_no_shared_content PASSED [ 19%]
tests/test_solution.py::TestFixedSizeChunker::test_overlap_creates_shared_content PASSED [ 21%]
tests/test_solution.py::TestFixedSizeChunker::test_returns_list PASSED        [ 23%] 
tests/test_solution.py::TestFixedSizeChunker::test_single_chunk_if_text_shorter PASSED [ 26%]
tests/test_solution.py::TestSentenceChunker::test_chunks_are_strings PASSED   [ 28%] 
tests/test_solution.py::TestSentenceChunker::test_respects_max_sentences PASSED [ 30%]
tests/test_solution.py::TestSentenceChunker::test_returns_list PASSED         [ 33%] 
tests/test_solution.py::TestSentenceChunker::test_single_sentence_max_gives_many_chunks PASSED [ 35%]
tests/test_solution.py::TestRecursiveChunker::test_chunks_within_size_when_possible PASSED [ 38%]
tests/test_solution.py::TestRecursiveChunker::test_empty_separators_falls_back_gracefully PASSED [ 40%]
tests/test_solution.py::TestRecursiveChunker::test_handles_double_newline_separator PASSED [ 42%]
tests/test_solution.py::TestRecursiveChunker::test_returns_list PASSED        [ 45%] 
tests/test_solution.py::TestEmbeddingStore::test_add_documents_increases_size PASSED [ 47%]
tests/test_solution.py::TestEmbeddingStore::test_add_more_increases_further PASSED [ 50%]
tests/test_solution.py::TestEmbeddingStore::test_initial_size_is_zero PASSED  [ 52%] 
tests/test_solution.py::TestEmbeddingStore::test_search_results_have_content_key PASSED [ 54%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_have_score_key PASSED [ 57%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_sorted_by_score_descending PASSED [ 59%]
tests/test_solution.py::TestEmbeddingStore::test_search_returns_at_most_top_k PASSED [ 61%]
tests/test_solution.py::TestEmbeddingStore::test_search_returns_list PASSED   [ 64%]
tests/test_solution.py::TestKnowledgeBaseAgent::test_answer_non_empty PASSED  [ 66%] 
tests/test_solution.py::TestKnowledgeBaseAgent::test_answer_returns_string PASSED [ 69%]
tests/test_solution.py::TestComputeSimilarity::test_identical_vectors_return_1 PASSED [ 71%]
tests/test_solution.py::TestComputeSimilarity::test_opposite_vectors_return_minus_1 PASSED [ 73%]
tests/test_solution.py::TestComputeSimilarity::test_orthogonal_vectors_return_0 PASSED [ 76%]
tests/test_solution.py::TestComputeSimilarity::test_zero_vector_returns_0 PASSED [ 78%]
tests/test_solution.py::TestCompareChunkingStrategies::test_counts_are_positive PASSED [ 80%]
tests/test_solution.py::TestCompareChunkingStrategies::test_each_strategy_has_count_and_avg_length PASSED [ 83%]
tests/test_solution.py::TestCompareChunkingStrategies::test_returns_three_strategies PASSED [ 85%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_filter_by_department
 PASSED [ 88%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_no_filter_returns_all_candidates PASSED [ 90%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_returns_at_most_top_k PASSED [ 92%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_reduces_collection_size PASSED [ 95%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_returns_false_for_nonexistent_doc PASSED [ 97%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_returns_true_for_existing_doc PASSED [100%]

================================ 42 passed in 0.09s ================================ 
```

**Số tests pass:** 42 / 42

---

## 5. Similarity Predictions — Cá nhân (5 điểm)

| Pair | Sentence A | Sentence B | Dự đoán | Actual Score | Đúng? |
|------|-----------|-----------|---------|--------------|-------|
| 1 | I need to change my email address. | I need to change my email address. | high | 1.000 | Yes |
| 2 | Reset my password. | I want to reset my password. | high | -0.064 | No |
| 3 | Clear cache and cookies before retrying. | Clear browser cache and cookies before retrying. | high | 0.055 | No |
| 4 | Use exponential backoff for 429 errors. | Use exponential backoff for 429 errors. | high | 1.000 | Yes |
| 5 | Contact the bank if the renewal failed. | I want to bake banana bread with flour and eggs. | low | 0.039 | Yes |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn nghĩa?**
> Điều bất ngờ nhất là một số cặp câu gần nghĩa theo cảm giác người đọc lại cho điểm rất thấp hoặc thậm chí âm, trong khi các câu giống hệt nhau mới cho điểm 1.0. Điều này cho thấy kết quả similarity phụ thuộc rất mạnh vào chất lượng embedding; nếu embedding không thật sự nắm được ngữ nghĩa thì cosine similarity chỉ phản ánh quan hệ giữa vector đầu ra chứ không đảm bảo đúng nghĩa. Với `_mock_embed`, similarity dùng được để test kỹ thuật, nhưng không thể xem như đo ngữ nghĩa thật.

---

## 6. Results — Cá nhân (10 điểm)

Chạy 5 benchmark queries của nhóm trên implementation cá nhân của bạn trong package `src`. **5 queries phải trùng với các thành viên cùng nhóm.**

### Benchmark Queries & Gold Answers (nhóm thống nhất)

| # | Query | Gold Answer |
|---|-------|-------------|
| 1 | How can a customer change the email address on their OpenAI account? | A customer can change their email from Settings > Account on ChatGPT Web if the account supports email management. This is not supported for phone-number-only accounts, Enterprise SSO accounts, or some enterprise-verified personal accounts. After the change, the user is signed out and must log in again with the new email. |
| 2 | What should a customer do if they do not receive the password reset email? | The customer should check the spam/junk folder, confirm they are checking the same inbox used during signup, and verify there is no typo in the email address. If the account was created only with Google, Apple, or Microsoft login, password recovery must be done through that provider instead. |
| 3 | What are the recommended steps when a ChatGPT Plus or Pro renewal payment fails? | The customer should clear browser cache and cookies, contact the bank to check for blocks or security flags, verify billing and card details, and confirm the country or region is supported. If the payment still fails, they should contact support through the Help Center chat widget. |
| 4 | How should a customer handle a 429 Too Many Requests error? | A 429 error means the organization exceeded its request or token rate limit. The recommended solution is exponential backoff: wait, retry, and increase the delay after repeated failures. The customer should also reduce bursts, optimize token usage, and consider increasing the usage tier if needed. |
| 5 | When should an active customer emergency be escalated, and who should be contacted first? | Escalation should be considered when the emergency lasts more than 3 hours without clear resolution, involves multiple simultaneous customer issues, blocks critical outside work, or requires broader coordination. A Support Manager On-call should be consulted, and the account CSM should usually be contacted first as the escalation DRI. |

### Kết Quả Của Tôi

| # | Query | Top-1 Retrieved Chunk (tóm tắt) | Score | Relevant? | Agent Answer (tóm tắt) |
|---|-------|--------------------------------|-------|-----------|------------------------|
| 1 | How can a customer change the email address on their OpenAI account? | Top-1 lấy từ `password_reset_help.md` (đoạn `Settings > Account`); top-3 có `account_email_change.md` | 0.2707 | Top-1: Không, Top-3: Có | Trả lời có thể chạm đúng một phần thao tác settings nhưng thiếu các điều kiện không hỗ trợ đổi email |
| 2 | What should a customer do if they do not receive the password reset email? | Top-1 lệch sang `service_limit_429.md`; top-3 không chứa tài liệu password | 0.3569 | Top-1: Không, Top-3: Không | Trả lời dễ sai trọng tâm vì retrieval không đưa đúng nguồn reset-password |
| 3 | What are the recommended steps when a ChatGPT Plus or Pro renewal payment fails? | Top-1 lệch sang `password_reset_help.md`; top-3 có `billing_renewal_failure.md` | 0.1953 | Top-1: Không, Top-3: Có | Trả lời có thể đúng một phần nếu model tận dụng top-3, nhưng grounding ở top-1 chưa tốt |
| 4 | How should a customer handle a 429 Too Many Requests error? | Top-1 và top-3 đều lệch sang `account_email_change.md` | 0.1707 | Top-1: Không, Top-3: Không | Trả lời không bám được nguồn 429 trong lần chạy này |
| 5 | When should an active customer emergency be escalated, and who should be contacted first? | Top-1 đúng `internal_escalation_playbook.md`; top-3 đều đúng tài liệu nội bộ | -0.0163 | Top-1: Có, Top-3: Có | Trả lời đúng hướng escalation: điều kiện >3 giờ, nhiều vấn đề đồng thời, và liên hệ CSM trước |

**Bao nhiêu queries trả về chunk relevant trong top-3?** 3 / 5

---

## 7. What I Learned (5 điểm — Demo)

**Điều hay nhất tôi học được từ thành viên khác trong nhóm:**
> Tôi học được từ bạn Ngô Quang Tăng rằng tách theo câu (`SentenceChunker`) có thể cho grounding tốt hơn trong các câu hỏi dạng hướng dẫn từng bước, vì mỗi chunk thường gọn và dễ khớp ý. Cách bạn ấy ưu tiên semantic unit thay vì cố giữ chunk dài giúp giảm trường hợp trả lời lan man. Điều này giúp tôi nhìn rõ hơn việc chọn strategy phải gắn với kiểu câu hỏi, không chỉ dựa vào số chunk.

**Điều hay nhất tôi học được từ nhóm khác (qua demo):**
> Qua demo nhóm khác, tôi học được cách dùng metadata filter chặt hơn theo `category` và `audience` trước khi search để giảm nhiễu ở top-1. Họ cũng làm rõ nguồn hỗ trợ cho từng câu trả lời (trích chunk nào), nên phần grounding thuyết phục hơn. Tôi thấy đây là điểm rất thực tế khi triển khai RAG cho tài liệu có cả public và internal.

**Nếu làm lại, tôi sẽ thay đổi gì trong data strategy?**
> Nếu làm lại, tôi sẽ chuẩn hóa lại dữ liệu theo các đoạn ngắn mang tính "atomic instruction" (ví dụ mỗi bước xử lý tách rõ) để giảm nhầm lẫn giữa các tài liệu gần nghĩa như password và account settings. Tôi cũng sẽ bổ sung metadata chi tiết hơn như `intent` (reset_password, change_email, refund, rate_limit) để filter theo ý định trước khi tính similarity. Ngoài ra, tôi sẽ thử kết hợp query rewriting để tăng khả năng match đúng tài liệu ở top-1.

---

## Tự Đánh Giá

| Tiêu chí | Loại | Điểm tự đánh giá |
|----------|------|-------------------|
| Warm-up | Cá nhân | 5 / 5 |
| Document selection | Nhóm | 10 / 10 |
| Chunking strategy | Nhóm | 13 / 15 |
| My approach | Cá nhân | 10 / 10 |
| Similarity predictions | Cá nhân | 4 / 5 |
| Results | Cá nhân | 7 / 10 |
| Core implementation (tests) | Cá nhân | 30 / 30 |
| Demo | Nhóm | 4 / 5 |
| **Tổng** | | **83 / 100** |
