# RAG evaluation report

Golden set: `tests/eval/golden_set.json` — 20 questions, `top_k=5`.

| Setting   | Value                                                                                                    |
| --------- | -------------------------------------------------------------------------------------------------------- |
| Retriever | `PgHybridRetriever (pgvector + Postgres full-text, RRF)`                                                 |
| LLM       | `ollama (qwen3:14b)`                                                                                     |
| Embedder  | `local fastembed (384-d)`                                                                                |
| Tenant    | `e8d78ef2-ade1-4806-ac8d-b97093daf858`                                                                   |
| Corpus    | `cobalt_calibration_service.txt`, `northwind_returns_faq.txt`, `northwind_sla_support.txt`, `sample.txt` |

## Retrieval legs

Which parts of the hybrid search produced candidates, counted from the retriever's own log lines during this run. A question where only the vector leg fired was decided by embeddings alone, so its recall does not exercise the full-text index.

| Leg                           | Questions |
| ----------------------------- | --------- |
| Retrieval calls               | 20        |
| Both legs contributed         | 1 / 20    |
| Vector leg only               | 19 / 20   |
| Full-text leg only            | 0 / 20    |
| Neither leg                   | 0 / 20    |
| Full-text candidates returned | 1         |

## Summary

| Metric                             | Value     |
| ---------------------------------- | --------- |
| Questions                          | 20        |
| Average recall                     | **1.000** |
| Full-recall questions              | 20 / 20   |
| Expected document ranked first     | 19 / 20   |
| Mean rank of the expected document | 1.05      |
| Average answer fact coverage       | 0.908     |
| Questions with a non-empty answer  | 20 / 20   |
| Verdict (avg recall ≥ 0.80)        | **PASS**  |

## Per-question recall

| #   | Question                                                                                 | Expected                         | Retrieved                                                                                  | Rank | Recall | Facts |
| --- | ---------------------------------------------------------------------------------------- | -------------------------------- | ------------------------------------------------------------------------------------------ | ---- | ------ | ----- |
| 1   | How many people does Northwind Logistics employ?                                         | `sample.txt`                     | `sample.txt`, `northwind_sla_support.txt`, `northwind_returns_faq.txt`                     | #1   | 1.00   | 1.00  |
| 2   | What revenue did Northwind book in the 2024 fiscal year, and how much of it was softwar… | `sample.txt`                     | `sample.txt`, `northwind_returns_faq.txt`, `northwind_sla_support.txt`                     | #1   | 1.00   | 1.00  |
| 3   | How many shipments per month does Northwind handle, and by which transport modes?        | `sample.txt`                     | `sample.txt`, `northwind_sla_support.txt`, `northwind_returns_faq.txt`                     | #1   | 1.00   | 1.00  |
| 4   | What accuracy do Northwind contracts promise on the assistant's citations, and how ofte… | `sample.txt`                     | `sample.txt`, `northwind_sla_support.txt`, `northwind_returns_faq.txt`, +1 more            | #1   | 1.00   | 0.50  |
| 5   | What did Northwind commit to for 2026, including the SOC 2 deadline?                     | `sample.txt`                     | `sample.txt`, `northwind_sla_support.txt`, `northwind_returns_faq.txt`                     | #1   | 1.00   | 1.00  |
| 6   | How long do I have to return a hardware device to Northwind?                             | `northwind_returns_faq.txt`      | `northwind_returns_faq.txt`, `northwind_sla_support.txt`, `sample.txt`                     | #1   | 1.00   | 1.00  |
| 7   | Does the Northwind return period start on the order date or on the delivery date?        | `northwind_returns_faq.txt`      | `northwind_returns_faq.txt`, `northwind_sla_support.txt`, `sample.txt`                     | #1   | 1.00   | 1.00  |
| 8   | How quickly must I report a shipment that arrived damaged?                               | `northwind_returns_faq.txt`      | `northwind_returns_faq.txt`, `northwind_sla_support.txt`, `cobalt_calibration_service.txt` | #1   | 1.00   | 1.00  |
| 9   | If my damaged delivery is approved, do I get money back or a new unit?                   | `northwind_returns_faq.txt`      | `northwind_returns_faq.txt`, `northwind_sla_support.txt`, `sample.txt`                     | #1   | 1.00   | 0.50  |
| 10  | Who can approve a refund settlement above twenty thousand euros?                         | `northwind_returns_faq.txt`      | `northwind_returns_faq.txt`, `northwind_sla_support.txt`, `sample.txt`                     | #1   | 1.00   | 1.00  |
| 11  | What monthly uptime does Northwind commit to, and is it higher on the Growth plan?       | `northwind_sla_support.txt`      | `northwind_sla_support.txt`, `sample.txt`, `northwind_returns_faq.txt`                     | #1   | 1.00   | 1.00  |
| 12  | When is the Northwind support desk staffed, and in which timezone?                       | `northwind_sla_support.txt`      | `northwind_sla_support.txt`, `sample.txt`, `northwind_returns_faq.txt`                     | #1   | 1.00   | 1.00  |
| 13  | How quickly must a Severity 1 incident at Northwind get a response?                      | `northwind_sla_support.txt`      | `northwind_sla_support.txt`, `northwind_returns_faq.txt`, `sample.txt`                     | #1   | 1.00   | 1.00  |
| 14  | How is a Northwind service credit calculated when the uptime commitment is missed?       | `northwind_sla_support.txt`      | `northwind_sla_support.txt`, `northwind_returns_faq.txt`, `sample.txt`                     | #1   | 1.00   | 1.00  |
| 15  | What are Northwind's recovery point and recovery time objectives?                        | `northwind_sla_support.txt`      | `northwind_sla_support.txt`, `northwind_returns_faq.txt`, `sample.txt`                     | #1   | 1.00   | 1.00  |
| 16  | How often do the Cobalt Cold Chain logger probes have to be calibrated?                  | `cobalt_calibration_service.txt` | `cobalt_calibration_service.txt`, `northwind_sla_support.txt`, `sample.txt`                | #1   | 1.00   | 1.00  |
| 17  | How long must a temperature mapping study run for a Cobalt pharmacy fridge?              | `cobalt_calibration_service.txt` | `cobalt_calibration_service.txt`, `northwind_sla_support.txt`, `sample.txt`                | #1   | 1.00   | 0.67  |
| 18  | When does a Cobalt temperature mapping study have to be repeated?                        | `cobalt_calibration_service.txt` | `cobalt_calibration_service.txt`, `northwind_sla_support.txt`, `northwind_returns_faq.txt` | #1   | 1.00   | 1.00  |
| 19  | What does an out-of-hours emergency callout cost for a Cobalt unit that has already los… | `cobalt_calibration_service.txt` | `northwind_sla_support.txt`, `cobalt_calibration_service.txt`, `northwind_returns_faq.txt` | #2   | 1.00   | 0.50  |
| 20  | How is the Cobalt alarm threshold set, and what is the rear relay rated for?             | `cobalt_calibration_service.txt` | `cobalt_calibration_service.txt`, `northwind_sla_support.txt`, `sample.txt`                | #1   | 1.00   | 1.00  |

`Rank` is the 1-based position of the first expected document in the retrieved list — the chunk the prompt leads with. `Recall` is the fraction of expected documents retrieved; `Facts` is the fraction of a question's `expected_facts` found in the generated answer (see `app/rag/eval.py` for what that does and does not prove).

## Latency (Stage 10 spans, milliseconds)

| Stage      | n   | p50    | p95    | max    | p50 time-to-first | cancelled | aborted |
| ---------- | --- | ------ | ------ | ------ | ----------------- | --------- | ------- |
| `llm_ttft` | 20  | 1319.2 | 2004.9 | 4648.5 | 637.6             | 0         | 0       |
| `retrieve` | 20  | 7.0    | 7.5    | 7.8    | —                 | 0         | 0       |

Percentiles are nearest-rank over the spans of this run: p95 is a duration this system actually produced, not an interpolation. `retrieve` is the real embedder plus both Postgres legs; `llm_ttft` covers token streaming, so its total is paced by the answer length rather than by the model alone.

## Notes

* Recall is the metric this stage is about: the fraction of expected documents hybrid retrieval returned inside the top chunks.
* Answer fact coverage is substring matching over a normalised answer, so it catches a wrong or empty answer and nothing subtler.
* Latency comes from the Stage 10 spans of this run, so the table and the log lines describe the same work.
* `llm_ttft` times token streaming, so its total is paced by the answer length as much as by the model (ollama (qwen3:14b)).
* Run at 2026-09-27T21:09:30+00:00 against tenant `e8d78ef2-ade1-4806-ac8d-b97093daf858` (`rag-eval`), which is reset on every run; the corpus is the four files in `tests/fixtures/`.
* **Finding:** the full-text leg returned no candidates for 19 of 20 questions, so most of this recall is vector-only. `plainto_tsquery` ANDs every token of a question, and a multi-term question rarely has all of its words in one chunk; the RRF fusion is therefore running with one leg. Fixing that is a retrieval change, not an eval change, so it is recorded in `PROGRESS.md` as the Stage 10 follow-up rather than patched here.
