# Research Plan

## Project 1: Build an AI Agent Maximizing Non-Prefix Caching

Design an agent framework that maximizes cache hit rates, especially in scenarios where prefix-matching fails and substring matching becomes essential.

Current agents use traditional append-only workflows and either forget or summarize context when the length limit is reached. They do not leverage context editing techniques that boost performance—and coincidentally cause prefix matching to fail.

We plan to integrate these two new techniques into the current append-only workflow:

### Technique 1: Dynamic Memory Pool with Context Editing

Build a memory pool similar to the open-source LightMem (Fang et al., 2025) with frequently edited memory, using context editing as demonstrated in MemGPT. This approach achieves twice the hit rate through substring matching (LMCache, 2026; yamazakihirofumi, 2026) in cases where prefix matching fails but substring matching succeeds.

LMCache can cache these memory blocks when possible, and they can be retrieved later in blocks during user sessions.

### Technique 2: Structured Schema for Agent Interactions

Define a structured schema/template for agent interactions, similar to the Prompt Markup Language (PML) in Prompt Cache (Gim et al., 2024). This system will enable:

- Retrieval of cached substrings
- Granular context editing (e.g., single-token parameter updates)
- Avoiding full prefix re-computation

---

## Project 2: Continue CacheBlend Work for Agent Workloads

This is a continuation of Technique 2 above.

Utilize high-level semantic or lexical cues to guide targeted cross-attention recovery, thereby restoring critical inter-module dependencies. This approach addresses the limitations of both:

- **Direct KV cache concatenation** — which neglects cross-attention
- **Heuristics like CacheBlend** — which lack semantic awareness

### Exploring Gated Attention

Explore whether Gated Attention (Qiu et al., 2025) could further improve CacheBlend. Some studies show that attention sink—mitigated by gated attention—is another factor hurting RAG performance, as noted in CacheClip (Yang et al., 2025).

---

## Background

Standard prefix-based caching relies on the assumption that the beginning of a prompt (the prefix) remains static and identical across requests. However, modern prompt engineering techniques—ranging from optimization to compression—frequently violate this assumption by altering the token sequence, rendering traditional caches ineffective.
