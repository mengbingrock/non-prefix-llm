
# Mathematical Formulation of Context Management Strategies

Let $W$ be the active **Context Window**, $D_{\text{ext}}$ be **External Data (Disk)**, and $q$ be the user query.

---

## 1. Offload Context (Progressive Addition)

**Strategy**

$$
W \leftarrow W \cup \{ d \in D_{\text{ext}} \mid \text{Relevant}(d, q) \}
$$

The active context is a dynamic subset of external storage, populated only by necessary elements retrieved via protocols or tools.

**Claude & MCP (Graphiti)**

$$
G_{\text{sub}} \subset G_{\text{ext}}
$$

**Letta (Skills)**

$$
W_{\text{RAM}} \leftarrow \text{PageIn}(D_{\text{Disk}})
$$

**Mem0 (Selective)**

$$
W_{\text{new}} = W_{\text{old}} \cup \text{Top}_{k}(\text{Sim}(q, D_{\text{vec}}))
$$

---

## 2. Isolate Context (Domain Partitioning)

**Strategy**

$$
D_{\text{total}} = \bigcup_{i=1}^{n} P_i
\;\Rightarrow\;
\text{Result} = \text{Agg}\big(\{ \text{Process}(P_{i}) \}\big)
$$

The dataset is partitioned into isolated domains $P_{i}$, processed independently and aggregated.

**GraphRAG**

- $P_{i}$ correspond to graph communities  
- Local summaries are generated independently  
- Results are aggregated via Map–Reduce  

---

## 3. Edit Context (Mutable State)

### Letta (MemGPT) — Agentic State Machine

Letta treats memory as an explicit, agent-controlled operating system resource.

**State Definition**

Let $S_{t} = \{B_{\text{human}}, B_{\text{persona}}\}$ be the core memory blocks, and $\pi(S_{t}, I_{t})$ be the agent policy producing an action $a$.

**Update Function**

Memory evolves only when the agent emits an explicit write action:

$$S_{t+1} = \begin{cases} S_{t} \text{ with } B_{i} \leftarrow \text{content} & \text{if } a = {\mathtt{core-memory-replace}}(B_{i}, \text{content}) \\ S_{t} \text{ with } B_{i} \leftarrow B_{i} \oplus \text{content} & \text{if } a = {\mathtt{core-memory-append}}(B_{i}, \text{content}) \\ S_{t} & \text{otherwise} \end{cases}$$

**Operation Semantics**

- **INSERT (Append):** Agent appends new information
- **UPDATE (Replace):** Agent overwrites an existing block
- **DELETE:** Achieved implicitly by replacement with omitted content

---

## 4. Reduce Context (Compression)

**Strategy**

$$
W \leftarrow \text{Compress}(D_{\text{raw}})
\quad \text{such that} \quad
\lvert \text{Compress}(D_{\text{raw}}) \rvert \ll \lvert D_{\text{raw}} \rvert
$$

Information is mapped to a lower-dimensional or summarized representation to minimize token usage.

**ReadAgent**

$$
W \leftarrow \text{Gist}(D_{\text{segment}})
$$

**GraphRAG**

$$
W \leftarrow \{ \text{Summary}(C) \mid C \in \text{Communities} \}
$$
