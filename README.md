# RAG That Doesn't Lie

[![Awesome](https://awesome.re/badge.svg)](https://awesome.re)

> A practical playbook to build RAG systems that don't hallucinate — based on real production failures.

Most RAG systems don't fail because of the model.
They fail because of the pipeline.

After building and benchmarking a production RAG API (61 eval tasks, 14 LLMs), here's what actually breaks — and what works.

---

## Why Most RAG Systems Fail

RAG is supposed to reduce hallucinations.

In reality, most pipelines:
- Retrieve context...
- Then blindly trust the model

Result:
- Missing key facts
- Conflicting sources ignored
- Confident but wrong answers

The model is not the problem. The system is.

---

## 10 Real Failure Modes (From Production)

### 1. Retrieval looks "good" but misses critical chunks

Top-K similarity doesn't mean the *right* chunks are selected. A chunk about "warranty terms" may score lower than a general overview — but it's the one that matters.

### 2. Important data gets dropped by ranking

Numbers, dates, policy limits, entity names — these often rank low in vector similarity because they're short and specific. But they're exactly what the user asked about.

### 3. Model rewrites facts incorrectly

Source says "coverage period: 60 days." Model outputs "coverage period: 30 days." The right chunk was in context. The model just... changed it. This is generation drift, not a retrieval problem.

### 4. Cross-document contradictions ignored

Document A says "unlimited storage." Document B says "50GB limit." Most pipelines pick whichever scores higher. No reconciliation. No flag.

### 5. Chunking destroys meaning

A paragraph split mid-sentence loses context. A table split across chunks becomes garbage. Hierarchical chunking (small for retrieval, large for context) helps — flat chunking doesn't.

### 6. Over-aggressive reranking

Reranking improves the average case but can kill edge cases. We saw a header-penalty reranker drop retrieval by 2 points and increase variance 5x. Calibrated ≠ tweakable.

### 7. Multi-step retrieval adds noise

"Retrieve more, then filter" sounds smart. In practice, parent chunk expansion *reduced* our benchmark by 3 points. More recall ≠ more accuracy.

### 8. No grounding verification

The pipeline generates an answer and returns it. Nobody checks if the answer terms actually appear in the sources. This is where most hallucinations survive.

### 9. Prompt over-reliance

"Only answer from the provided context" is an instruction, not a guarantee. LLMs follow instructions probabilistically. Prompts reduce hallucinations — they don't eliminate them.

### 10. No failure mode

The system always answers — even when it shouldn't. A well-designed pipeline should say "insufficient evidence" instead of guessing. The refusal IS the feature.

---

## What Actually Works

### 1. Treat retrieval as a signal, not truth

Hybrid search (BM25 + vector) catches what either alone misses. BM25 finds exact terms. Vectors find semantics. Together they cover more ground. But never blindly trust the result.

### 2. Force critical chunks (slot-based)

Don't rely on ranking alone. Detect the entities and attributes the query asks about. Force-include chunks that match — regardless of their similarity score. This is constraint-based, not score-based.

### 3. Extract key facts BEFORE the LLM

Pull numbers, dates, percentages, and named values from retrieved chunks. Inject them as a structured "must include" section in the prompt. The model can't forget what's explicitly listed.

### 4. Verify AFTER generation (non-negotiable)

Check the answer against the sources:
- Do answer terms appear in the source text?
- Do numbers match?
- Are there negation conflicts ("never" vs "12 months")?
- Do citations trace back to real sources?

If the answer isn't grounded → reject it. Return "insufficient evidence."

### 5. Accept refusal as success

A correct "I don't know" is infinitely better than a confident wrong answer. Design for refusal. The 17% accuracy gap in our benchmarks = the system correctly not guessing.

---

## Results (Real Benchmarks)

| Metric | Result |
|--------|--------|
| Hallucination rate | **0%** across 61 evaluation tasks |
| Accuracy | **83%** (remaining 17% = correct refusals) |
| LLMs tested | **14** models, 3 runs each |
| Avg latency | **~1.2s** |

Key insight:

> You don't need perfect accuracy to eliminate hallucination.
> You need verification.

Another surprise: the cheapest model (Qwen 3.5 Flash, $0.065/M tokens) performed the same as GPT-4.1 on this pipeline. The pipeline matters more than the model.

---

## What Didn't Work

| Attempt | Result | Lesson |
|---------|--------|--------|
| Multi-step retrieval | Retrieval dropped 3 points | More recall ≠ more accuracy |
| Header penalties in reranker | Retrieval dropped 2 points, variance 5x | Don't touch calibrated scoring |
| Parent chunk expansion | Degraded benchmark | Added noise, not context |
| Switching to GPT-4.1 | Same cross-doc score as Qwen | Pipeline > model |

RAG is a balanced system. Small changes can break it.

---

## Tools & Resources

### Detection

| Tool | What it does | Link |
|------|-------------|------|
| LettuceDetect | Token-level hallucination detection | [GitHub](https://github.com/KRLabsOrg/LettuceDetect) |
| LongTracer | Claim verification against sources | [GitHub](https://github.com/ENDEVSOLS/LongTracer) |
| MiniCheck | Fact-checking, GPT-4 level at 400x lower cost | [GitHub](https://github.com/Liyan06/MiniCheck) |
| RAGAS | Evaluation framework | [GitHub](https://github.com/explodinggradients/ragas) |
| DeepEval | LLM evaluation with hallucination metrics | [GitHub](https://github.com/confident-ai/deepeval) |

### Papers

| Paper | Key finding |
|-------|------------|
| [Lost in the Middle](https://arxiv.org/abs/2307.03172) | LLMs ignore context in the middle of the prompt |
| [MiniCheck](https://arxiv.org/abs/2404.10774) | 770M model matches GPT-4 for fact-checking |
| [LettuceDetect](https://arxiv.org/abs/2502.17125) | 79% F1 on RAGTruth with encoder-only model |
| [FACTS Grounding](https://deepmind.google/blog/facts-grounding-a-new-benchmark-for-evaluating-the-factuality-of-large-language-models/) | Even Gemini fails ~16% of factual claims |
| [Stanford Legal RAG Study](https://hai.stanford.edu/news/hallucination-free-assessing-reliability-leading-ai-legal-research-tools) | Specialized legal RAG tools hallucinate 17-33% |

### Benchmarks

| Benchmark | What it measures | Link |
|-----------|-----------------|------|
| RAGTruth | Hallucination detection accuracy | [Paper](https://arxiv.org/abs/2401.00396) |
| FACTS Grounding | LLM factual accuracy | [DeepMind](https://deepmind.google/blog/facts-grounding-a-new-benchmark-for-evaluating-the-factuality-of-large-language-models/) |
| TruthfulQA | Language model truthfulness | [GitHub](https://github.com/sylinrl/TruthfulQA) |
| HaluEval | Hallucination evaluation | [GitHub](https://github.com/RUCAIBox/HaluEval) |

---

## Want This Without Rebuilding Everything?

We built this entire pipeline into an API:

**[wauldo.com](https://wauldo.com)** — Upload docs, ask questions, get verified answers.

- Native PDF/DOCX upload with quality scoring
- Built-in fact-check endpoint (3 modes)
- Citation verification
- OpenAI-compatible
- [Free tier on RapidAPI](https://rapidapi.com/binnewzzin/api/smart-rag-api)

Full technical breakdown: [How We Achieved 0% Hallucination Rate in Our RAG API](https://dev.to/wauldo/how-we-achieved-0-hallucination-rate-in-our-rag-api-with-benchmarks-4g54)

---

## Contributing

PRs welcome:
- Add failure modes you've encountered
- Share real-world cases with specifics
- Improve techniques with benchmarks
- Add tools or papers

See [contribution guidelines](https://github.com/wauldo/.github/blob/main/CONTRIBUTING.md).

---

## If This Helped

Star the repo — it helps more builders avoid broken RAG systems.

---

## License

[![CC0](https://licensebuttons.net/p/zero/1.0/88x31.png)](https://creativecommons.org/publicdomain/zero/1.0/)
