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
| Faithfulness | Adversarial / out-of-scope: assistant correctly refuses and has little lexical overlap with gold policy text (e.g. medical diagnosis request). | In-scope answer invents deadlines, fees, GPA cutoffs, or refund rates not in retrieved/gold context. | Block deploy if in-scope hallucination. Add grounding/citation guardrail; do not treat OOS refusals as faithfulness bugs. |
| Answer Relevance | Correct policy answer that paraphrases without repeating question tokens (word-overlap heuristic under-scores). | Answer addresses a different intent (tuition vs scholarship, grade appeal vs service complaint). | Alert on paraphrase false-negatives; investigate routing/prompt if intent mismatch. Add intent check or human spot-check. |
| Context Recall | Out-of-scope / prompt-injection cases where expected answer is a refusal and retriever need not pull full policy corpus. | Easy factual lookup misses the one document that contains the date/amount (e.g. Fall add/drop deadline). | If recall is low on Easy/Medium in-scope items: improve chunking, query rewrite, or top-k. Do not raise k blindly on adversarial items. |
| Context Precision | Recall is high and relevant chunks appear, but some noise sits later in the ranked list. | Relevant evidence is buried while unrelated chunks rank first, so the generator sees noise first. | Rerank (Exercise 3.5) or tighten retrieval. Precision-only drops are usually alert, not block, if recall stays high. |
| Completeness | Minor optional detail missing (office name wording) while dates, amounts, and conditions are present. | Missing a governing condition/exception (census vs after-census refund, scholarship probation vs conduct loss, policy v1 vs v2). | Iterate generation prompt / context window. Treat missing exceptions on money/deadline questions as critical, not cosmetic. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
>
> Dùng cùng một pair (Answer A, Answer B) trên cùng question + rubric, chỉ đổi thứ tự trình bày cho judge.
>
> - **Condition 1 — A first:** prompt = `[Answer A] then [Answer B]`. Record scores.
> - **Condition 2 — B first:** prompt = `[Answer B] then [Answer A]`. Record scores.
> - **Optional control:** randomize order across N pairs and report how often the first-listed answer wins.
>
> **Detection rule:** position bias exists if the first-listed answer systematically scores higher even when content is unchanged (mean score_first − score_second is significantly > 0, or win-rate of position 1 ≫ 50%). If A beats B in Condition 1 but B beats A in Condition 2, the judge is following order, not quality.
>
> Run the swap on both a clear-winner pair and a near-tie pair so the effect is not hidden by a huge quality gap.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
>
> Rubric phải chấm **claim quality**, không chấm độ dài:
> 1. Mỗi mức 1–5 nêu điều kiện cụ thể (correct dates/fees, exceptions, evidence, safety) — không dùng “thorough / detailed / comprehensive” như tín hiệu điểm cao.
> 2. **Explicit penalty:** extra sentences that do not add a grounded policy claim do not raise the score; unsupported extra claims lower Faithfulness/Correctness.
> 3. **Reward concision when complete:** a short answer that covers all required conditions can score 5; a long answer missing an exception cannot.
> 4. Cap or ignore stylistic filler (repeated restatements of the question).
> 5. In pairwise judging, tell the judge to ignore length and compare only rubric dimensions.
>
> Example for Student Services: “USD 420 per credit, due by the regular registration deadline, 5-day grace period” scores higher than a long essay that omits the grace-period vs deadline distinction.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
>
> LLM judges inherit model biases (position, verbosity, self-preference) and may disagree with domain experts on high-stakes Student Services cases (privacy refusals, policy versions, false premises). Calibration against a small human-labeled set:
> - measures agreement (correlation / exact-match on 1–5);
> - reveals systematic leniency or severity;
> - lets you adjust rubric wording, temperature, or ensemble before using the judge as a CI gate.
>
> Without human calibration, a high automatic judge score can still mean “sounds like the judge model,” not “correct Northstar policy.” Calibration is especially needed when the generator and the judge share a model family.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.70 | Hallucinated fees, deadlines, or GPA rules can cause real student harm. Below 0.70 on in-scope items = block deploy. Adversarial refusals are reviewed separately so a correct OOS refuse is not treated as a faithfulness regression. |
| Answer Relevance | 0.60 | Intent mismatch (e.g. answering tuition when asked about scholarship renewal) is serious but slightly more recoverable than invented facts. 0.60 blocks clearly off-intent answers while allowing paraphrase that the word-overlap heuristic under-scores. |
| Completeness | 0.60 | Missing a governing exception (after-census refund = 0%, late-add fee non-refundable) is a policy failure. 0.60 blocks large gaps; minor wording misses can alert rather than block if Faithfulness stays ≥ 0.70. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
>
> - **Offline evaluation** (golden dataset + RAGAS-style metrics / this lab pipeline): every code, prompt, chunking, or retriever change; before demo/launch; as a CI quality gate (`run_regression` vs last baseline). Cheap, repeatable, no live students.
> - **Online evaluation** (traces, user feedback, TruLens/Langfuse-style monitoring): after deploy, on real portal traffic — latency, refusal rate, thumbs-down, escalation to Registrar/IT. Catches distribution shift that the 20-QA golden set cannot cover.
> - **Human review:** high-stakes or ambiguous cases — grade appeals, privacy/security, medical leave, suspected prompt injection, and any item where automatic metrics disagree or Faithfulness is borderline. Also used to calibrate the LLM-as-a-Judge rubric before trusting it in CI.
>
> Production flow: offline gate must pass → deploy with online monitoring → sample high-stakes traces for human review → feed new failures back into the golden dataset.

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
