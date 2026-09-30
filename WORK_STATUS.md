# Work Status

- Current goal: Hoàn thiện lab AI Evaluation theo CHECKPOINTS.md, gồm benchmark thật và reflection.
- Completed: CP0–CP5 implementation deliverables được hoàn tất đến mức có thể với API hiện tại. Evaluation core Tasks 1–5 và reranker bonus; golden dataset 20 QA, strata 5/7/5/3, evidence 10/10 docs; `solution/solution.py` đồng bộ. Benchmark thật sinh 20/20 answers với gpt-4o-mini/top_k=5. Gold evidence A03/M07 được mở rộng sau trace review; expected A02 được thu hẹp cho đúng nội dung câu hỏi; adapter được chạy lại trên actual answers (assistant chỉ nhận câu hỏi). Exercises 3.2 và reflection được điền bằng kết quả cuối, kèm phân tích giới hạn heuristic.
- Files changed/created: `WORK_STATUS.md`, `template.py`, `solution/solution.py`, `golden_dataset.json`, `exercises.md`, `reflection.md`; runtime artifacts trong `artifacts/actual_answers.json` và `artifacts/benchmark_results.json`.
- Key decisions: Không hiển thị `.env`; không dùng fake data. Dùng human trace review khi overlap score mâu thuẫn nội dung; ghi rõ thay đổi dataset sau benchmark và rerun evaluate.
- Final benchmark: 15/20 pass (75%); Context Recall 0.921; Context Precision 0.895; Faithfulness 0.723; Relevance 0.706; Completeness 0.752. Failure types: off_topic=4, hallucination=1. Lowest Overall: A03 0.358, A01 0.424, E03 0.643.
- Verification: `.venv` dependencies installed; `python -m pytest tests/ -q` previously passed 42; dataset validator PASS. Re-run both after final edits before handoff.
- Open issues: Heuristic scores produce semantic false positives (e.g. E03 exact expected answer flagged below threshold); `FailureAnalyzer.find_root_cause()` may misattribute when answer-side lexical scores disagree with trace. Reflection documents these limits.
- Next steps: Final rerun tests/validator, check Git for accidental secrets and confirm `template.py`/`solution/solution.py` match.
