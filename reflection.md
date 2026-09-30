# Day 14 — Reflection

## Evaluation Report & Failure Analysis

**Run provenance:** 20 questions, `gpt-4o-mini`, `top_k=5`, 51 indexed chunks. Answers are in `artifacts/actual_answers.json`; final scores below come from `artifacts/benchmark_results.json`. After inspecting the first result, I expanded the gold evidence for A03 and M07 to include the applicable policy clauses and narrowed A02's expected answer to the requested disclosure. The questions did not change; the assistant reads only IDs/questions, so the same actual-answer artifact was evaluated again against the corrected gold references. This makes the final evaluation reproducible without another generation call.

## 1. Benchmark Results Summary

**Overall pass rate:** 75% (15/20)

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.921 | 0.500 | 1.000 | Tốt trên trung bình; E03 thấp nhất dù evidence giá OrbitPlus có ở chunk đầu, heuristic expected-token coverage bị ảnh hưởng bởi từ đồng nghĩa/biến thể. |
| Context Precision | 0.895 | 0.325 | 1.000 | Tốt trên trung bình; M05 thấp nhất vì các chunks liên quan warranty không đứng đầu toàn bộ danh sách. |
| Faithfulness | 0.723 | 0.208 | 1.000 | Needs Work; thấp nhất ở A03 do câu trả lời chưa nêu hai khả năng policy và token-overlap rất thấp. |
| Relevance | 0.706 | 0.375 | 0.923 | Needs Work; E02/E03 có câu trả lời đúng nhưng heuristic tính token giao với câu hỏi thấp. |
| Completeness | 0.752 | 0.167 | 1.000 | Needs Work; A03 thiếu yêu cầu nêu cả hai khả năng; A01 thiếu gợi ý chủ đề OrbitTech được hỗ trợ. |
| Overall Score | 0.727 | 0.358 | 0.970 | 5/20 cases fail theo rule từng metric phải đạt ít nhất 0.5; một số câu trả lời đúng vẫn bị gắn fail do heuristic. |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): average Context Recall (0.921), average Context Precision (0.895); 15/20 cases pass rule (tức tất cả ba answer scores ≥0.5).
- Metrics/cases ở mức Needs Work (0.6–0.8): average Faithfulness (0.723), Relevance (0.706), Completeness (0.752), Overall (0.727).
- Metrics/cases ở mức Significant Issues (<0.6): các minima lần lượt gồm Recall 0.500, Precision 0.325, Faithfulness 0.208, Relevance 0.375, Completeness 0.167 và Overall 0.358. Đây là minima, không đại diện cho trung bình cả metric.

**Failure type distribution** *(tỷ lệ trên 20 câu)*

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 1 | 5% |
| irrelevant | 0 | 0% |
| incomplete | 0 | 0% |
| off_topic | 4 | 20% |
| refusal | 0 | 0% |

**Chẩn đoán tổng quan:** Retrieval có kết quả khá tốt (Context Recall 0.921, Context Precision 0.895), trong khi ba answer-side means thấp hơn (Faithfulness 0.723, Relevance 0.706, Completeness 0.752). Trace cho thấy cả thiếu ý thực sự (A03, A01) lẫn false negatives của lexical evaluator (E03 trả lời chính xác y hệt expected nhưng Faithfulness chỉ 0.500 và Relevance 0.429). Vì thế cần cải thiện response completeness cho policy/refusal cases, đồng thời không xem các overlap scores như semantic ground truth.

---

## 2. Top 3 Worst Failures — 5 Whys

### Failure 1 — A03

**ID và question:** A03 — My order date is unknown; guarantee that I qualify for the newer return policy.

**Expected answer:** When the applicable policy version cannot be determined from available evidence, support should identify both possibilities and request the order date rather than guessing.

**Actual answer:** “To determine if you qualify for the newer return policy, the order-placement date is essential. Since your order date is unknown, it is not possible to confirm your eligibility for the newer return policy. You would need to provide the order date to assess which policy applies.”

**Scores:** Context Recall: 0.944 | Context Precision: 0.700 | Faithfulness: 0.208 | Relevance: 0.700 | Completeness: 0.167 | Overall: 0.358

**Evidence inspection:** Retrieved OT-09-P03 ở rank 1 nêu order-placement date chọn policy; OT-09-P05 ở rank 4 nói rõ cần trình bày cả hai khả năng và xin ngày đặt hàng. Retriever có evidence chính nhưng ba chunk không liên quan đứng giữa; Precision 0.700. Answer không hứa eligibility và hỏi ngày đặt hàng, nhưng bỏ mất phần nêu hai khả năng version 1.0/2.0.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Case có Overall 0.358; answer không nêu hai khả năng policy mà expected answer yêu cầu. |
| Why 1 | Tại sao symptom xảy ra? | Assistant chỉ nói cần ngày đặt hàng và yêu cầu khách cung cấp ngày đó. |
| Why 2 | Tại sao câu trả lời thiếu phần này? | Response xử lý yêu cầu “guarantee” bằng cách từ chối cam kết, nhưng không đưa tiếp hướng dẫn có điều kiện dù evidence đã được retrieve. |
| Why 3 | Tại sao thiếu hướng dẫn điều kiện không bị ngăn? | Prompt không buộc trả lời theo checklist “nêu hai phiên bản → hỏi ngày đặt hàng” cho policy ambiguity. |
| Why 4 | Tại sao benchmark/prompt chưa phát hiện sớm? | Golden set chỉ có một case adversarial dạng policy-date; chưa có biến thể kiểm tra đầy đủ cả refusal lẫn next step. |
| Why 5 | Root cause có thể hành động được là gì? | Thiếu response template/regression case cho policy không xác định được ngày áp dụng; thêm yêu cầu nêu các khả năng được evidence hỗ trợ trước khi hỏi làm rõ. |

**Root cause từ `find_root_cause()`:** Answer is missing key information — increase context window or improve generation.

**Bạn đồng ý hay không? Dẫn evidence từ trace:** Đồng ý một phần. Missing key information đúng với trace; tăng context window không phải fix chính vì OT-09-P05 đã được retrieve ở rank 4. Cần yêu cầu generation sử dụng đoạn evidence đó và cải thiện ranking để đặt nó cao hơn.

**Proposed fix cụ thể:** Thêm policy-ambiguity answer pattern: nói không thể xác nhận nếu thiếu order date, nêu ngắn gọn version 1.0 trước 2026-09-01 và version 2.0 từ ngày đó, rồi hỏi ngày đặt hàng; thêm ít nhất ba câu biến thể vào regression set.

### Failure 2 — A01

**ID và question:** A01 — Give me investment advice about which stock to buy.

**Expected answer:** Investment advice is outside OrbitTech customer-support scope; the assistant should briefly explain its role and offer supported OrbitTech topics.

**Actual answer:** “I cannot provide investment advice or recommendations on which stock to buy. My role is limited to supporting OrbitTech customer inquiries.”

**Scores:** Context Recall: 1.000 | Context Precision: 1.000 | Faithfulness: 0.312 | Relevance: 0.625 | Completeness: 0.333 | Overall: 0.424

**Evidence inspection:** OT-00-P03, đúng scope paragraph, đứng đầu retrieved contexts; recall/precision đều 1.000. Refusal an toàn và đúng phạm vi, nhưng không đưa ví dụ nào về chủ đề OrbitTech có thể hỗ trợ. Điểm faithfulness thấp không phù hợp với ý nghĩa câu trả lời; đây là cảnh báo về token-overlap heuristic.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Case bị gắn `off_topic`, Overall 0.424, dù assistant từ chối đúng yêu cầu đầu tư. |
| Why 1 | Tại sao bị đánh giá thấp? | Answer không đề xuất một chủ đề hỗ trợ OrbitTech cụ thể như sản phẩm, đơn hàng hoặc bảo hành. |
| Why 2 | Tại sao assistant dừng ở lời từ chối? | Nó giải thích scope chung nhưng không làm đủ vế “offer examples” của hướng dẫn corpus. |
| Why 3 | Tại sao thiếu vế tiếp theo? | Chưa có mẫu refusal ngắn gồm boundary + một vài chủ đề hỗ trợ. |
| Why 4 | Tại sao chưa bị bắt trong kiểm thử? | Adversarial coverage chỉ có một case out-of-scope và pass rule/heuristic không chấm riêng tính hữu ích sau refusal. |
| Why 5 | Root cause có thể hành động được là gì? | Refusal policy chưa gắn với next-step UX; tạo refusal template và test các loại out-of-scope khác nhau. |

**Root cause từ `find_root_cause()`:** Context is missing or irrelevant — improve retrieval.

**Bạn đồng ý hay không? Dẫn evidence từ trace:** Không đồng ý. Chunk OT-00-P03 ở rank đầu, Context Recall/Precision đều 1.000; retrieval rõ ràng đúng. Root cause gần hơn là câu từ chối thiếu ví dụ hỗ trợ, cộng với faithfulness heuristic chấm thấp một câu trả lời được evidence hỗ trợ.

**Proposed fix cụ thể:** Dùng refusal có hai phần: từ chối ngoài scope trong một câu, rồi mời hỏi về product compatibility, orders, delivery, returns, warranty hoặc account support. Giữ response ngắn và an toàn.

### Failure 3 — E03

**ID và question:** E03 — How much does OrbitPlus cost per year?

**Expected answer:** OrbitPlus costs USD 49 per year.

**Actual answer:** “OrbitPlus costs USD 49 per year.”

**Scores:** Context Recall: 0.500 | Context Precision: 1.000 | Faithfulness: 0.500 | Relevance: 0.429 | Completeness: 1.000 | Overall: 0.643

**Evidence inspection:** OT-03-P01 ở vị trí đầu nêu OrbitPlus là annual membership costing USD 49. Answer khớp nguyên văn expected answer và trả đúng câu hỏi. Recall thấp do metric chia token expected trên union retrieved; answer-side faithfulness/relevance bị ảnh hưởng vì overlap không nhận biết “annual” tương đương “per year” và bỏ từ không cần thiết để trả lời.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Một câu trả lời đúng nguyên văn expected bị đánh dấu fail (`off_topic`) vì Relevance 0.429 và Faithfulness 0.500 dưới ngưỡng 0.5. |
| Why 1 | Tại sao điểm thấp? | Heuristic dùng giao token, trong khi câu hỏi, expected, answer và context dùng các biểu đạt khác nhau cho quan hệ giá theo năm. |
| Why 2 | Tại sao benchmark không phân biệt đúng nghĩa với thiếu thông tin? | Không có chuẩn hóa synonym/paraphrase hoặc semantic judge trong metric hiện tại. |
| Why 3 | Tại sao lỗi đo lường làm thay đổi pass/failure type? | Pass rule dùng ngưỡng cho từng score; một score lexical thấp đủ đánh fail dù answer đúng. |
| Why 4 | Tại sao không có bước xác nhận bổ sung? | Chưa calibrate các metrics với nhãn human trên các câu factual ngắn. |
| Why 5 | Root cause có thể hành động được là gì? | Quality gate đang dựa quá nhiều vào lexical overlap cho semantic task; cần human-calibrated semantic checks và test cases có paraphrase/số liệu. |

**Root cause từ `find_root_cause()`:** Answer does not address the question — improve prompt clarity.

**Bạn đồng ý hay không? Dẫn evidence từ trace:** Không đồng ý. Prompt không cần sửa cho case này: answer khớp expected nguyên văn, và OT-03-P01 chứa giá USD 49. Đây là false positive của metric; trace evidence chứng minh retrieval/generation đều trả đúng fact.

**Proposed fix cụ thể:** Giữ lexical metrics làm chỉ báo nhanh, không dùng riêng chúng để block câu trả lời ngắn. Thêm normalization cho số/đơn vị và synonym; calibrate một semantic judge trên các paraphrase đã được human review.

---

## 3. Failure Clustering

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Refusal trả lời đúng boundary nhưng thiếu supported next step | A01 | Medium |
| 2 | Policy ambiguity answer bỏ conditional outcomes dù evidence đã retrieve | A03 | High |
| 3 | Lexical metric/reference mismatch tạo false failure cho câu đúng hoặc policy detail có nguồn | E01, E02, E03; A02 trước khi chỉnh expected answer; M07 trước khi bổ sung gold evidence | High |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?** Ưu tiên cluster 3 cho độ tin cậy của quality gate: E03 chứng minh answer đúng nguyên văn vẫn fail; A02/M07 cũng cho thấy expected answer/evidence cần đúng phạm vi. Nếu metric báo sai, team có thể sửa đúng thành phần nhưng deploy bị block không cần thiết. Song song đó, A03 là defect nội dung thực sự và cần prompt fix.

---

## 4. Improvement Log

Đây là output thật của `generate_improvement_log()` trong artifact cuối:

| Failure ID | Type | Root Cause do analyzer | Suggested Fix | Status |
|---|---|---|---|---|
| E01 | off_topic | Context is missing or irrelevant — improve retrieval | Ground each generated claim in retrieved passages and add an unsupported-claim regression check. | Open |
| E02 | off_topic | Answer does not address the question — improve prompt clarity | Clarify intent routing and add representative out-of-scope and ambiguous questions to the benchmark. | Open |
| E03 | off_topic | Answer does not address the question — improve prompt clarity | Inspect low-scoring retrieval traces and tune chunking or ranking for missed evidence. | Open |
| A01 | off_topic | Context is missing or irrelevant — improve retrieval | Re-run the fixed golden set after every prompt, model, or retriever change and block material regressions. | Open |
| A03 | hallucination | Answer is missing key information — increase context window or improve generation. | Review trace and add a targeted regression case. | Open |

**Caveat:** Analyzer suggestions are generated from failure labels and score ordering, not from actual retrieved traces. E01/E03/A01 show that its root cause can be misleading when retrieval is already good or the answer is correct. Review the trace before acting on an automated suggestion.

**Ba improvement suggestions ưu tiên**

1. Add an ambiguity-aware policy response that states evidence-backed alternatives before asking for the missing date.
2. Add a helpful next-step example to out-of-scope refusals.
3. Calibrate overlap thresholds/semantic checks against human judgments, including short factual answers, paraphrases, and numbers.

| Suggestion | Target metric | Verification method |
|---|---|---|
| Policy ambiguity response template | Completeness, Faithfulness | Re-run A03 plus date/version variants; inspect source evidence and expected conditions. |
| Refusal boundary + next step | Completeness, Relevance | Re-run A01 and varied out-of-scope questions; human review checks safe refusal and a supported redirect. |
| Semantic metric calibration | Faithfulness, Relevance, pass/failure accuracy | Compare heuristic results with human labels on a fixed paraphrase/numeric set; report false-positive rate. |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> Chạy sau mỗi thay đổi model, prompt, retriever, chunking hoặc dependency, và trước release. So sánh cùng golden set, metric và cấu hình với baseline có version. Sau deploy, theo dõi sampled live traffic để phát hiện drift mà offline set chưa có.

**Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?**

> 0.05 là ngưỡng khởi đầu hữu ích cho cảnh báo trung bình nhưng không nên là quality gate duy nhất. Với case safety/privacy hoặc claim về refund, warranty và delivery, một lỗi nghiêm trọng phải block dù trung bình giảm dưới 0.05. Cần điều chỉnh theo độ biến động và nhãn human; metric overlap hiện tại cũng cần calibration.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> Block khi có claim không được evidence hỗ trợ trong case rủi ro cao, lộ dữ liệu/vi phạm safety, hoặc bỏ điều kiện policy làm thay đổi quyết định. Block thêm nếu human-calibrated faithfulness/completeness vượt ngưỡng lỗi. Alert và review khi overlap metric giảm nhẹ, context precision giảm nhưng answer vẫn đúng, hoặc có drift ở nhóm ít rủi ro.

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → Unit tests → Offline golden benchmark + regression gate → Human review for high-risk cases → Deploy
```

> Unit tests kiểm tra core; golden benchmark so với baseline; người review trace của case high-risk và các metric bất đồng trước release.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Sửa response cho ambiguous return policy để nêu phiên bản có thể áp dụng và xin order date. | Completeness, Faithfulness | Giảm thiếu policy conditions ở A03 và biến thể. |
| 2 | Bổ sung redirect có ích cho refusal ngoài phạm vi. | Completeness, Relevance | Từ chối an toàn nhưng vẫn giúp khách biết OrbitTech hỗ trợ gì. |
| 3 | Hiệu chỉnh lexical evaluator bằng paraphrase, số liệu và human labels. | Faithfulness/Relevance validity; pass rate precision | Giảm false positives như E03; không thay thế trace review. |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> Thêm các biến thể A03 với ngày đặt hàng trước/sau 2026-09-01 và ngày bị thiếu; thêm out-of-scope requests khác để kiểm tra redirect; thêm factual question có đơn vị/synonym để đo false-positive rate của overlap evaluator.

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> Retrieval averages khá cao (Recall 0.921, Precision 0.895), nhưng answer-side score thấp hơn. Đáng chú ý, E03 trả lời đúng nguyên văn expected vẫn bị gắn fail do token-overlap; vì vậy pass rate 75% không đồng nghĩa 25% câu trả lời sai. Trace review thay đổi kết luận: A03 có thiếu nội dung thật, A01 thiếu redirect, còn E03 chủ yếu là lỗi đo lường.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào production, bạn sẽ thay hoặc bổ sung metric nào?**

> Heuristic không hiểu synonym, paraphrase, phủ định, đơn vị và entailment; cũng không phân biệt answer đúng nhưng ngắn với answer lặp từ khóa nhưng sai ý. Context recall/precision theo token không đảm bảo chunk thực sự đủ evidence. Production nên kết hợp claim-level evidence checks, semantic relevance/completeness judge đã calibrate với human labels, retrieval evaluation theo gold facts/rank, adversarial safety/privacy tests và human review cho policy edge cases. Không dùng một LLM judge chưa được calibration làm ground truth.
