# Non-Prefix LLM Caching

Standard prefix-based caching relies on the assumption that the beginning of a prompt (the prefix) remains static and identical across requests [[1]](#references). However, modern prompt engineering techniques—ranging from optimization to compression—frequently violate this assumption by altering the token sequence, rendering traditional caches ineffective.

---
## Special Agent Workflow

RepoAgent[[12]](#references), automates documentation generation for code repositories. To fully understand a class, it recursively traverses its superclasses, subclasses, and composite dependencies. This recursive behavior leads to repeated reads of the same class definitions across different call paths.

In benchmarks conducted by LMCache[[11]](#references), prefix-based caching achieved less than 5% hit rate in this scenario, while non-prefix caching reached over 70% hit rate—demonstrating the significant advantage of position-independent caching for recursive agent workflows. 



---

## Context Editing

Examples include Claude Code Context editing APIs, supporting removing thinking block and tool-call results if later turns, and MemGPT (Letta)[[10]](#references), dynamicly changing chat history when user asks to. Recent work on lightweight memory-augmented generation [[7]](#references) and effective KV cache reuse [[8]](#references) further explores these paradigms.

ExpRAG [[13]](#references), a highly optimized RAG pipeline, features dynamic memory that retrieves the top-K most relevant entries based on context. Since the retrieved memories vary unpredictably and appear non-contiguously from the memory pool, prefix caching struggles to find matches. Substring matching with concatenation proves far more effective in this scenario.

**Resources:**
- [Context Editing Documentation](https://platform.claude.com/docs/en/build-with-claude/context-editing)
- [Memory Tool Documentation](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)
- [MemGPT Agent Trace Dataset](https://github.com/LMCache/lmcache-agent-trace/blob/main/memgpt/memgpt1.jsonl)


---

## Prompt Pruning, Filtering & Reusing

Pruning techniques aim to reduce token costs and latency by removing "unimportant" information. However, by deleting tokens from the middle of sentences or paragraphs, they alter the token sequence so that it no longer matches the uncompressed original.

### 1. Self-Information-Based Content Filtering (Selective Context)

Proposed by Li et al. [[2]](#references), this method uses "self-information" (a measure of surprisal) to evaluate the informativeness of lexical units. Content with low self-information is deemed redundant and filtered out.

Similar to LLMLingua, this creates a discontinuous token sequence. Since standard KV caches require the entire preceding sequence to be identical to validly reuse attention states, the deletion of even a single "uninformative" token early in the prompt invalidates the cache for all subsequent tokens.

### 2. LLMLingua

LLMLingua [[4]](#references) employs a smaller, efficient model (like GPT-2 or LLaMA-7B) to calculate the perplexity of each token in a prompt. It then removes tokens with low perplexity—those the model considers predictable or redundant.

Because this removal happens globally across the prompt (including instructions and context), the resulting sequence is a "holey" version of the original. Even if the semantic meaning is preserved, the exact token sequence is fundamentally changed, causing a "cache miss" for any system relying on exact string matching.

### 3. Prompt Cache

Prompt Cache [[3]](#references) addresses these limitations by abandoning the "prefix match" paradigm in favor of a modular, explicit declaration system:

- **Explicit Schema vs. String Matching:** Instead of scanning the raw text for overlapping strings, Prompt Cache uses a Prompt Markup Language (PML) where users explicitly define reusable segments as "modules" (e.g., `<module name="system-message">`).

- **Position Independence:** The system pre-computes attention states for these named modules and stores them independently of their position. When a prompt requests a module, the system performs an exact name lookup to retrieve the cached states, rather than trying to match a text prefix.

- **Handling Variations:** This allows "discontinuous" attention states to be reused. Even if the user inserts new text (like optimized DSPy instructions) between two cached modules, Prompt Cache can stitch the cached states together with the new text, whereas a prefix cache would fail the moment the new text appeared.

---

## Prompt Enhancement & Optimization

Automated optimization frameworks treat prompts as mutable variables rather than static instructions. By iteratively refining the text to maximize performance, they introduce variations that break the strict character-for-character alignment required for prefix caching.

### 1. TextGrad

TextGrad [[6]](#references) applies gradient-based feedback (similar to backpropagation) to text, treating the prompt as a learnable parameter. It iteratively modifies specific segments of the prompt—including system instructions and problem definitions—based on error signals from the model.

Because these "textual gradient" updates can insert or replace tokens at the very beginning of the prompt to correct reasoning errors, the prefix changes with every optimization step, preventing the reuse of previously cached attention states.

### 2. DSPy

DSPy [[5]](#references) moves away from manual string manipulation, instead "compiling" high-level declarative signatures into optimized prompts. The framework automatically optimizes instructions and few-shot examples (demonstrations) to maximize a specific metric.

This compilation process is dynamic; DSPy may reorder instructions, swap out few-shot examples, or rephrase the task description entirely between runs. Since the "prefix" in DSPy is a generated artifact of the optimization process rather than a static string, it rarely remains stable enough for traditional prefix caching to remain effective.

---


## References

[1] Google for Developers. (2025). *Optimize inference speed with prefix caching.*

[2] Li, Y., et al. (2023). *Unlocking Context Constraints of LLMs: Enhancing Context Efficiency of LLMs with Self-Information-Based Content Filtering.* [arXiv:2304.12102](https://arxiv.org/abs/2304.12102)

[3] Gim, I., et al. (2024). *Prompt Cache: Modular Attention Reuse for Low-Latency Inference.* [arXiv:2311.04934v2](https://arxiv.org/abs/2311.04934v2)

[4] Jiang, H., et al. (2023). *LongLLMLingua: Accelerating and Enhancing LLMs in Long Context Scenarios via Prompt Compression.* [arXiv:2310.06839](https://arxiv.org/abs/2310.06839)

[5] Khattab, O., et al. (2023). *DSPy: Compiling Declarative Language Model Calls into Self-Improving Pipelines.*

[6] Tworkowski, S., et al. (2025). *Integrating TextGrad-style Prompt Optimization with Memory-Driven Self-Evolution.* [arXiv:2508.18749](https://arxiv.org/abs/2508.18749)

[7] Fang, J., Deng, X., Xu, H., Jiang, Z., Tang, Y., Xu, Z., Deng, S., Yao, Y., Wang, M., Qiao, S., Chen, H., & Zhang, N. (2025). *LightMem: Lightweight and efficient memory-augmented generation.* [arXiv:2510.18866](https://arxiv.org/abs/2510.18866)

[8] Yang, B., Leng, Q., Zeng, J., & Wu, Z. (2025). *CacheClip: Accelerating RAG with effective KV cache reuse.* [arXiv:2510.10129](https://arxiv.org/abs/2510.10129)

[9] Qiu, Z., Wang, Z., Zheng, B., Huang, Z., Wen, K., Yang, S., Men, R., Yu, L., Huang, F., Huang, S., Liu, D., Zhou, J., & Lin, J. (2025). Gated attention for large language models: Non-linearity, sparsity, and attention-sink-free. arXiv preprint. https://doi.org/10.48550/arXiv.2505.06708

[10] LMCache. (2026). lmcache-agent-trace [Source code]. GitHub. https://github.com/LMCache/lmcache-agent-trace/blob/main/memgpt_result/memgpt1_chunk_size_16.png

[11] yamazakihirofumi. (2026). Add RepoAgent trace (Pull Request #1) [Source code]. GitHub. https://github.com/LMCache/lmcache-agent-trace/pull/1

[12] OpenBMB. (2024). RepoAgent (Version 0.2.0) [Computer software]. GitHub. https://github.com/OpenBMB/RepoAgent

[13] Wei, T., Sachdeva, N., Coleman, B., He, Z., Bei, Y., Ning, X., Ai, M., Li, Y., He, J., Chi, E. H., Wang, C., Chen, S., Pereira, F., Kang, W., & Cheng, D. Z. (2025). Evo-Memory: Benchmarking LLM agent test-time learning with self-evolving memory (arXiv:2511.20857). arXiv. https://arxiv.org/abs/2511.20857