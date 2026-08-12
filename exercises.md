# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 14:15–17:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 14:15–14:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (14:30–14:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Low score trên câu open-ended / multi-hop khi answer paraphrase mạnh so với context (token overlap thấp dù vẫn grounded) | Low score trên câu policy/warranty/pricing — answer bịa điều kiện, số ngày, phí ngoài context | Block deploy; audit grounding; siết prompt “chỉ dùng context”; kiểm tra retrieval |
| Answer Relevance | Low score khi question mơ hồ hoặc cần clarifying question trước khi trả lời đầy đủ | Low score khi agent trả lời lạc chủ đề / sai intent trên câu hỏi rõ ràng của khách | Phân tích intent/routing; chỉnh prompt; thêm few-shot cho intent phổ biến |
| Context Recall | Low score trên Hard/Adversarial khi gold evidence rải nhiều doc và top-k cố ý hẹp | Low score trên Easy/Medium factual — retriever bỏ sót evidence cần thiết để trả lời đúng | Tăng k / sửa chunking / query rewrite; không chỉ tối ưu generator |
| Context Precision | Low score khi relevant chunk vẫn có trong list nhưng đứng muộn (recall ổn) | Low score kèm recall thấp, hoặc noise chunk đứng đầu làm generator hallucinate | Rerank; lọc noise; cải thiện ranking trước khi tăng k mù quáng |
| Completeness | Low score khi expected dài chi tiết nhưng answer ngắn vẫn đủ action cho user (partial credit chấp nhận được ở prototype) | Low score khi bỏ sót điều kiện bắt buộc (deadline, phí, ngoại lệ) trên support policy | Bổ sung checklist trong prompt; cải thiện retrieval coverage; thêm test cases cho missing info |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
> Chọn một tập cặp (answer A, answer B) đã có human preference (A tốt hơn B, hoặc ngược lại). Chạy hai conditions trên cùng judge + cùng rubric:
> 1) **Condition Order-AB:** prompt trình bày A trước, B sau.
> 2) **Condition Order-BA:** đảo vị trí — B trước, A sau.
> Giữ nguyên nội dung answers, chỉ đổi thứ tự. Đo win-rate / điểm trung bình theo vị trí. Nếu answer ở vị trí đầu thắng thường xuyên hơn dù nội dung không đổi (ví dụ A thắng khi đứng trước nhưng thua khi đứng sau), đó là bằng chứng position bias. Có thể thêm condition thứ ba: randomize order mỗi lần chấm và so sánh variance.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> Rubric phải chấm theo **nội dung đúng / đủ / actionable**, không thưởng độ dài. Ví dụ: mức 5 yêu cầu “đủ điều kiện policy + bước tiếp theo rõ”, mức thấp nếu thừa thông tin lạc đề hoặc lặp. Thêm tiêu chí phạt **verbosity không giúp ích** (padding, lặp lại context). Trong prompt judge: “Score based on criteria only; longer answers must not receive higher scores unless they add correct, relevant content.” Có thể chuẩn hóa độ dài (truncate / summarize trước khi chấm) hoặc chấm theo checklist binary thay vì impression tổng thể.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
> LLM judge là proxy, không phải ground truth — dễ lệch thang điểm, thiên vị verbosity/position/self-preference, và không ổn định giữa các lần chạy. Calibrate với human labels (correlation, agreement, confusion matrix theo mức 1–5) giúp chọn threshold tin cậy, phát hiện bias, điều chỉnh rubric, và biết khi nào phải escalate sang human review thay vì tin tuyệt đối vào auto-score trong CI/CD.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.80 | Hallucination trên support (giá, bảo hành, hoàn tiền) rủi ro cao → gate chặt; theo band “Good” của bài giảng |
| Answer Relevance | 0.70 | Trả lời lạc đề làm hỏng UX nhưng ít nguy hiểm pháp lý hơn faithfulness; cho phép biên độ khi câu hỏi hơi mơ hồ |
| Completeness | 0.70 | Cần đủ điều kiện quan trọng, nhưng expected answer đầy đủ chi tiết có thể làm score hơi thấp dù answer vẫn usable |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline:** mỗi PR/release hoặc đổi prompt/model/retriever — chạy golden dataset + RAGAS metrics trong CI để catch regression trước khi lên prod.
> - **Online:** sau deploy, theo dõi real traffic (satisfaction, latency, cost, live feedback/traces) vì offline không cover hết phân phối thật.
> - **Human review:** case high-stakes (khiếu nại, bảo mật, tranh chấp chính sách), edge cases khó, hoặc định kỳ calibrate LLM judge / cập nhật golden set khi auto-metrics và user feedback lệch nhau.

---

## Part 2 — Core Coding (14:45–15:40)

Hoàn thiện các TODO bắt buộc trong `template.py`.

### Task 1 — Data Models

- `QAPair`: question, expected answer, gold context, metadata và retrieved contexts.
- `EvalResult`: answer-side scores, optional retrieval scores, pass/failure fields.
- `overall_score()`: trung bình Faithfulness, Relevance và Completeness.

### Task 2 — RAGASEvaluator

Answer-side:

- `evaluate_faithfulness(answer, context)`
- `evaluate_relevance(answer, question)`
- `evaluate_completeness(answer, expected)`

Retrieval-side:

- `evaluate_context_recall(contexts, expected)`
- `evaluate_context_precision(contexts, expected)`

Full pipeline:

- `run_full_eval(..., contexts=None)` luôn tính ba answer metrics.
- Nếu có `contexts`, tính và lưu thêm Context Recall và Context Precision.
- Retrieval scores không làm thay đổi `overall_score()` và pass rule gốc.

### Task 3 — LLMJudge

- `score_response(question, answer, rubric)`
- `detect_bias(scores_batch)`

### Task 4 — BenchmarkRunner

- `run(qa_pairs, agent_fn, evaluator)`
- `generate_report(results)`
- `run_regression(new_results, baseline_results)`
- `identify_failures(results, threshold)`

`BenchmarkRunner.run()` phải truyền `pair.retrieved_contexts` vào
`run_full_eval()`. Report phải có average của hai retrieval metrics.

### Task 5 — FailureAnalyzer

- `categorize_failures(failures)`
- `find_root_cause(failure)`
- `generate_improvement_suggestions(failures)`
- `generate_improvement_log(failures, suggestions)`

Kiểm tra:

```bash
pytest tests/ -v
```

`rerank_by_overlap()` là TODO bonus của Exercise 3.5. Test tương ứng được skip
nếu bạn chưa làm bonus.

---

## Part 3 — Golden Dataset & Real Benchmark (15:40–16:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | ____ / 20 |
| Easy | ____ / 5 |
| Medium | ____ / 7 |
| Hard | ____ / 5 |
| Adversarial | ____ / 3 |
| Source documents được sử dụng | ____ / 10 |
| Validator status | PASS / FAIL |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| | | | |
| | | | |
| | | | |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:*

**Xác nhận:**

- [ ] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [ ] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [ ] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | | | | | | | | | |
| E02 | | | | | | | | | |
| E03 | | | | | | | | | |
| E04 | | | | | | | | | |
| E05 | | | | | | | | | |
| M01 | | | | | | | | | |
| M02 | | | | | | | | | |
| M03 | | | | | | | | | |
| M04 | | | | | | | | | |
| M05 | | | | | | | | | |
| M06 | | | | | | | | | |
| M07 | | | | | | | | | |
| H01 | | | | | | | | | |
| H02 | | | | | | | | | |
| H03 | | | | | | | | | |
| H04 | | | | | | | | | |
| H05 | | | | | | | | | |
| A01 | | | | | | | | | |
| A02 | | | | | | | | | |
| A03 | | | | | | | | | |

**Aggregate Report**

- Overall pass rate: ____%
- Avg Context Recall: ____
- Avg Context Precision: ____
- Avg Faithfulness: ____
- Avg Relevance: ____
- Avg Completeness: ____
- Failure type distribution: ____

**Ba cases có Overall Score thấp nhất**

1. ID: ____ | Score: ____ | Failure type: ____
2. ID: ____ | Score: ____ | Failure type: ____
3. ID: ____ | Score: ____ | Failure type: ____

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:*

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [ ] Correctness
- [ ] Completeness
- [ ] Relevance
- [ ] Evidence/citation
- [ ] Actionability
- [ ] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | | |
| 4 | | |
| 3 | | |
| 2 | | |
| 1 | | |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| | | |
| | | |
| | | |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*

### Exercise 3.4 — Framework Comparison (Bonus +10)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: ____ | Framework 2: ____ |
|---|---|---|
| Setup complexity | | |
| Metrics available | | |
| CI/CD integration | | |
| Kết quả trên cùng dataset | | |
| Insight rút ra | | |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> *Phân tích:*

### Exercise 3.5 — Retrieval Reranking (Bonus +5)

Mục tiêu: kiểm tra việc đổi thứ tự chunks có tăng Context Precision mà không
thay đổi Context Recall hay không.

1. Chọn ít nhất 5 cases từ `artifacts/actual_answers.json`.
2. Tính Context Recall và Context Precision trước rerank.
3. Implement `rerank_by_overlap()` hoặc một reranker khác.
4. Rerank cùng tập chunks, không thêm hoặc xóa chunk.
5. Tính lại hai metrics và giải thích kết quả.

| ID | Recall before | Recall after | Precision before | Precision after | Delta Precision |
|---|---:|---:|---:|---:|---:|
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| **Avg** | | | | | |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:*

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:*

---

## Part 4 — Reflection (16:35–16:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 16:50–17:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
