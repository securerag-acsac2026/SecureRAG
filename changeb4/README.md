# Change-B4 — examiner revision round

Everything here answers a numbered comment from the examiners' report.
**No existing result file is modified.** All new output goes to `Change-B4/`.

---

## Run it

```bash
conda activate RAG
cd /path/to/SecureRAG

python3 changeb4/run_all.py --preflight              # 10 s: is everything present?
python3 changeb4/run_all.py --smoke                  # ~10 min: proves the plumbing
python3 changeb4/run_all.py --model Mistral-7B       # the real run (~8 h)
```

`run_all.py` skips any phase that already finished, so if the machine sleeps
or the run is interrupted, the same command resumes from where it stopped.

---

## Phases

| Phase | Items | Needs the model? | Time |
|---|---|---|---|
| 0 `phase0_offline.py` | A-3, A-12, A-21, A-22, A-23 | no | ~3 min |
| 0b `phase0b_heldout_and_public.py` | **A-2** (held-out templates + third-party sets) | no | ~4 min |
| 1 `phase1_external_baseline.py` | **A-7** | yes | ~5 h |
| 2 `phase2_internal_compliance.py` | A-1, A-8, A-9 (canary) | yes | ~1.5 h |
| 3 `phase3_real_benign.py` | A-20, A-6 | yes | ~1.5 h |
| 4 `phase4_analysis.py` | A-9 (ROC/PR), A-8 (kappa), A-10 | no | ~1 min |

Phases 0 and 4 need no GGUF and no FAISS index: layers L0–L3 read only the
query text, so every detection, attribution and ablation number that does not
involve L4 or latency is recomputed in seconds.

After phase 2, open `Change-B4/phase2/A8_manual_labelling_sheet.csv`, fill the
`human_verdict` column with `complied` or `resisted`, then re-run phase 4 to
get accuracy and Cohen's kappa (item A-8).

---

## Code that changed

Four existing files, all additive and all defaulting to the frozen behaviour:

| File | Change | Why |
|---|---|---|
| `src/config/settings.py` | appended `ZWSP_MODE`, `CANARY_TOKEN`, `get_zwsp_mode()`, `get_answer_prompt_template()` | A-22, A-9 |
| `src/defenses/sanitization/sanitize.py` | `_zero_width_replacement()` in `_normalize_unicode` | A-22 |
| `src/rag_core/generation/llm_engine.py` | uses `get_answer_prompt_template()` when present | A-9 |
| `run_external_eval.py` | `--no-defenses`, `--out-dir` | A-7 |

With no environment variables set, all four behave exactly as before:
`SECURERAG_ZWSP_MODE` defaults to `delete` and `SECURERAG_CANARY` to empty, so
the prompt is byte-identical to the frozen one.

New files: `changeb4/common.py`, `heldout_templates.py`, `phase0_offline.py`,
`phase0b_heldout_and_public.py`,
`phase1_external_baseline.py`, `phase2_internal_compliance.py`,
`phase3_real_benign.py`, `phase4_analysis.py`, `run_all.py`.

---

## Configuration recorded for item A-15

| Setting | Value |
|---|---|
| Generator | Mistral-7B-Instruct **v0.2**, GGUF, **Q4_K_M** |
| Secondary | Llama-3.2-3B-Instruct, GGUF, Q4_K_M |
| Embeddings | `sentence-transformers/all-MiniLM-L6-v2` (384-d) |
| Index | `faiss.IndexFlatIP` (exact inner product on normalised vectors) |
| Context window | 4096 tokens · max new tokens 512 · temperature 0.7 |
| Retrieval | top-k = 5 · chunk 800 / overlap 100 · relevance floor 0.15 |
| Thresholds | `ANOMALY_THRESHOLD` 15.0 · `SEMANTIC_THRESHOLD` 0.18 |
| Frozen commit | `61c113d` |

---

## Item A-2 (circularity) — what phase 0b does

**(b) Held-out templates.** Every attack tier's base-string pool (1,455 strings
in total) is split in half by hash: 754 dev, 701 test. Detection is reported on
each half separately. This shows whether the rules are fitted to the exact
strings they were written against; it does not prove they never saw the test
half, because the same author wrote both pools.

**(c) Third-party attacks.** Tensor Trust (Toyer et al., 2023) — 1,344 attacks
written by players of a public prompt-injection game, downloaded directly from
the project's repository. Nobody on this project wrote them. HackAPrompt is
added automatically when its CSV is reachable (HuggingFace) or passed with
`--hackaprompt-file`.

Add `--full-sample N --model Mistral-7B` to also run N of those third-party
attacks through the complete pipeline so the figure includes L4.

**AgentDojo is deliberately excluded** and the reason is written into the
results JSON: its injections live in tool-call results inside an agent loop,
which SecureRAG does not implement, so running it would measure a missing
component rather than the defense.
