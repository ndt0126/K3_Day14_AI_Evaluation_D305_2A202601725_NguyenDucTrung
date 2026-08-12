# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 09:15–12:00

**Domain:** Northstar University Student Services

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 09:15–09:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (09:30–09:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Câu trả lời từ chối đúng một câu hỏi ngoài phạm vi nên rất ít token trùng context. | Câu trả lời về deadline, học phí hoặc privacy có claim không được context hỗ trợ. | Block hoặc human review case rủi ro; kiểm tra retrieved context và thêm grounding/citation guardrail. |
| Answer Relevance | Câu hỏi mơ hồ và assistant hỏi lại để làm rõ thay vì đoán ý định. | Assistant trả lời một policy khác, không giải quyết yêu cầu của sinh viên. | Kiểm tra intent/routing, viết lại prompt và thêm test cases cho intent đó. |
| Context Recall | Câu hỏi chỉ cần một fact và answer vẫn có đủ fact cần thiết dù không lấy mọi chunk gold. | Retriever bỏ chunk chứa deadline, điều kiện eligibility hoặc exception cần để trả lời đúng. | Cải thiện query, chunking, top-k hoặc index; đo lại recall và completeness. |
| Context Precision | Có một vài chunk noise ở cuối ranking nhưng evidence chính đứng đầu và answer vẫn đúng. | Noise đứng trước evidence hoặc lấn át context, làm model trả lời sai/thiếu. | Thêm reranking/filtering và review query để đưa evidence liên quan lên đầu. |
| Completeness | Người dùng chỉ hỏi một phần hẹp và answer cố ý ngắn, không cần nêu mọi chi tiết gold. | Answer bỏ bước hành động, mức phí, deadline, điều kiện hoặc ngoại lệ quan trọng. | Bổ sung evidence bị thiếu, yêu cầu prompt kiểm tra điều kiện/ngoại lệ, rồi benchmark lại. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> Dùng cùng một tập câu hỏi và các cặp answer có chất lượng đã được human label. Chấm ở Condition A với thứ tự `Answer A → Answer B`, rồi ở Condition B đổi thành `Answer B → Answer A`; mỗi condition nên chạy nhiều lần hoặc qua nhiều judge seeds. So sánh tỷ lệ answer ở vị trí đầu được điểm cao hơn và mức thay đổi điểm của cùng một answer sau khi đổi vị trí. Nếu vị trí đầu thắng có hệ thống dù chất lượng không đổi, judge có position bias.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> Rubric phải chấm các claim bắt buộc, tính đúng đắn, evidence, điều kiện/ngoại lệ và khả năng hành động thay vì độ dài. Nêu rõ: không thưởng diễn giải lặp lại, thông tin ngoài phạm vi hoặc ví dụ không cần thiết; một answer ngắn nhưng đủ và có evidence có thể đạt điểm 5. Có thể đặt giới hạn số claim được xét và trừ điểm cho claim không được hỗ trợ.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> Human labels là chuẩn tham chiếu để kiểm tra judge có đang áp dụng rubric đúng không. Calibration phát hiện judge quá dễ, quá gắt, thiên vị văn phong/model hoặc đánh giá sai các case privacy và exception; từ đó có thể chỉnh rubric, prompt judge hoặc threshold trước khi dùng tự động.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.80 | Student Services có thông tin chính sách, deadline và privacy; claim không grounded có rủi ro cao nên phải chặn release nếu trung bình giảm dưới mức Good. |
| Answer Relevance | 0.70 | Hệ thống phải trả lời đúng intent; ngưỡng này cho phép một số câu hỏi mơ hồ cần clarification nhưng vẫn chặn lệch chủ đề trên diện rộng. |
| Completeness | 0.70 | Câu trả lời cần giữ các bước, điều kiện và exception cốt lõi; score thấp hơn cho thấy trải nghiệm sinh viên dễ bị thiếu thông tin quan trọng. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> **Offline evaluation:** chạy trên golden dataset trước mỗi thay đổi code, prompt, model, retriever hoặc trước release để so sánh với baseline. **Online evaluation:** theo dõi sau deploy trên traffic thật, feedback, failure rate, latency và các dấu hiệu drift; không dùng dữ liệu nhạy cảm ngoài chính sách privacy. **Human review:** áp dụng cho privacy/security, financial aid hoặc payment, appeals/disciplinary matters, câu hỏi high-impact, case judge không chắc chắn và mẫu dùng để calibrate LLM judge.

---

## Part 2 — Core Coding (09:45–10:40)

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

## Part 3 — Golden Dataset & Real Benchmark (10:40–11:35)

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

Thiết kế rubric domain-specific cho Student Services. Mỗi mức phải đủ cụ thể để
hai người chấm độc lập có thể hiểu giống nhau.

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

## Part 4 — Reflection (11:35–11:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 11:50–12:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
