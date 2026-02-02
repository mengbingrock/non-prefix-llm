# Research Plan

## Project 1: Build an AI Agent Maximizing Non-Prefix Caching

LMCache uniquely offers non-prefix caching—a capability that provides significant gains where prefix-matching fails and substring matching becomes essential. The goal is to design an agent framework that maximizes these non-prefix cache hit rates.

Current agents use traditional append-only workflows and either forget or summarize context when the length limit is reached. They do not leverage context editing techniques that boost performance—and coincidentally cause prefix matching to fail.

We plan to deprecate the append-only workflow and replace it with these two techniques:

### Technique 1: LMcache to Support Graph Based Agent Memory

Direct Context Editing changed traditional append-only mode and prefix match fails, where opportunity for subtring could be. Build a memory pool similar to the open-source LightMem [[1]](#references) with frequently edited memory, using context editing as demonstrated in MemGPT [[2]](#references). This approach achieves twice the hit rate through substring matching [[3]](#references) [[4]](#references) in cases where prefix matching fails but substring matching succeeds.

LMCache can cache these memory blocks when possible, and they can be retrieved later in blocks during user sessions.

### Technique 2: Cache reasoning based RAG 

Support Cache Hit while performing reasoning. In this PageIndex[[9]](#references), it reasoning based on 'Table of Contents', We could cache the Contents and do prefeching for detailed data. They could use similiar method in the [[10]](#references) to add tool use in reasoning step, to do multiple hop retrival and reasoning.

### Technique 3: Structured Schema for Agent Interactions

Define a structured schema/template for agent interactions, similar to the Prompt Markup Language (PML) in Prompt Cache [[5]](#references). This system will enable:

- Retrieval of cached substrings
- Granular context editing (e.g., single-token parameter updates)
- Avoiding full prefix re-computation

---

## Project 2: Continue CacheBlend Work for Agent Workloads

This is a continuation of Technique 2 above.

Utilize high-level semantic or lexical cues to guide targeted cross-attention recovery, thereby restoring critical inter-module dependencies. This approach addresses the limitations of both:

- **Direct KV cache concatenation** — which neglects cross-attention
- **Heuristics like CacheBlend** [[6]](#references) — which lack semantic awareness

### Exploring Gated Attention

Explore whether Gated Attention [[7]](#references) could further improve CacheBlend. Some studies show that attention sink—mitigated by gated attention—is another factor hurting RAG performance, as noted in CacheClip [[8]](#references).

---

## Background

Standard prefix-based caching relies on the assumption that the beginning of a prompt (the prefix) remains static and identical across requests. However, modern prompt engineering techniques—ranging from optimization to compression—frequently violate this assumption by altering the token sequence, rendering traditional caches ineffective.

---

## References

[1] Fang, J., Deng, X., Xu, H., Jiang, Z., Tang, Y., Xu, Z., Deng, S., Yao, Y., Wang, M., Qiao, S., Chen, H., & Zhang, N. (2025). *LightMem: Lightweight and Efficient Memory-Augmented Generation.* [arXiv:2510.18866](https://arxiv.org/abs/2510.18866)

[2] Packer, C., Wooders, S., Lin, K., Fang, V., Patil, S. G., Stoica, I., & Gonzalez, J. E. (2024). *MemGPT: Towards LLMs as Operating Systems.* [arXiv:2310.08560](https://arxiv.org/abs/2310.08560)

[3] LMCache. (2026). *lmcache-agent-trace* [Source code]. GitHub. [Repository](https://github.com/LMCache/lmcache-agent-trace)

[4] yamazakihirofumi. (2026). *Add RepoAgent trace* (Pull Request #1). GitHub. [PR #1](https://github.com/LMCache/lmcache-agent-trace/pull/1)

[5] Gim, I., Chen, G., Lee, S., Sarda, N., Khandelwal, A., & Zhong, L. (2024). *Prompt Cache: Modular Attention Reuse for Low-Latency Inference.* [arXiv:2311.04934](https://arxiv.org/abs/2311.04934)

[6] Yao, J., Li, K., Zhao, X., Yang, G., Xu, J., & Jin, H. (2024). *CacheBlend: Fast Large Language Model Serving for RAG with Cached Knowledge Fusion.* [arXiv:2405.16444](https://arxiv.org/abs/2405.16444)

[7] Qiu, Z., Wang, Z., Zheng, B., Huang, Z., Wen, K., Yang, S., Men, R., Yu, L., Huang, F., Huang, S., Liu, D., Zhou, J., & Lin, J. (2025). *Gated Attention for Large Language Models: Non-linearity, Sparsity, and Attention-Sink-Free.* [arXiv:2505.06708](https://arxiv.org/abs/2505.06708)

[8] Yang, B., Leng, Q., Zeng, J., & Wu, Z. (2025). *CacheClip: Accelerating RAG with Effective KV Cache Reuse.* [arXiv:2510.10129](https://arxiv.org/abs/2510.10129)

[9] PageIndex, https://pageindex.ai/blog/pageindex-intro

[10] Recall, https://attractive-almandine-935.notion.site/ReCall-Learning-to-Reason-with-Tool-Call-for-LLMs-via-Reinforcement-Learning-1d7aec91e9bb8006ad40f9edbfe2191a
