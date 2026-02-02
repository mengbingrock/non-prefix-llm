# Comparative Analysis: Agent Memory Architectures (Mem0, Zep, Letta)

This document compares the architectural approaches of **Mem0**, **Zep**, and **Letta** (formerly MemGPT) regarding how they manage, store, and retrieve memory for AI agents.

## 1. Architectural Overview Graph

The following diagram illustrates the fundamental structural differences between the three systems:

```text
       ZEP (Temporal Graph)           LETTA (Stateful OS)             MEM0 (Universal Layer)
   [Focus: Evolution over Time]   [Focus: Context Management]    [Focus: Personalization/Speed]
             |                                 |                                |
  +----------v-----------+         +-----------v-----------+        +-----------v-----------+
  |    Ingest Stream     |         |    Context Window     |        |   Interaction Pairs   |
  | (Unstructured Data)  |         |      ("RAM")          |        |   (User + Assistant)  |
  +----------+-----------+         +-----------+-----------+        +-----------+-----------+
             |                                 |                                |
   [Extraction & Updates]          [Paging & Function Call]          [Extraction & Tool Call]
             |                                 |                                |
  +----------v-----------+         +-----------v-----------+        +-----------v-----------+
  |  Temporal Knowledge  |         |      Core Memory      |        |   Adaptive Memory     |
  |        Graph         | <-----> |        Blocks         |        |   (Vector + Graph)    |
  |                      |  (Swap) |   [Human] [Persona]   |        |                       |
  |  (Node)-[t1]->(Node) |         +-----------+-----------+        |  [User] [Session]     |
  |    |             |   |                     |                    |                       |
  |  (Node)-[t2]->(Node) |         +-----------v-----------+        +-----------+-----------+
  +----------+-----------+         |    Archival Storage   |                    |
             |                     |       ("Disk")        |                    |
      [Hybrid Search]              +-----------------------+           [Semantic Search]
 (Graph + Vector + Keyword)                                            (Vector Similarity)

---

## 2. Detailed Breakdown

### A. Zep: The Temporal Knowledge Graph
Zep utilizes a Temporal Knowledge Graph architecture (powered by Graphiti). It is designed to handle dynamic data that changes over time, distinguishing it from static RAG approaches.
• Structure: It builds a graph where nodes represent entities and edges represent relationships. Crucially, it uses a Bi-Temporal Data Model, tracking both when an event occurred (Valid Time) and when the data was recorded (Transaction Time).
• Management: Zep supports real-time incremental updates. As facts change or are superseded, the graph updates to reflect the new state without requiring a full re-computation of the graph.
• Retrieval: It employs a Hybrid Retrieval strategy, combining semantic embeddings, keyword search (BM25), and graph traversal to find relevant context.
• Use Case: Best for long-running agents where facts evolve (e.g., a user moves to a new city, changing the "lives_in" relationship) and reasoning about the sequence of events is critical.

### B. Letta (MemGPT): The Operating System
Letta (formerly MemGPT) treats the LLM as a processor and memory as an Operating System. It manages a hierarchy of memory to overcome fixed context windows.
• Structure: Memory is divided into:
    ◦ Main Context (RAM): The active context window, containing Core Memory Blocks. These are explicitly defined sections like human (facts about the user) and persona (agent instructions).
    ◦ External Context (Disk): Unlimited off-context storage, such as "Archival Memory" (conversation history) and "Recall Memory".
• Management: The agent manages its own memory via Function Calling (tool use). It can explicitly edit its Core Memory blocks or "page" information in from Archival Memory when the context window is full.
• Retrieval: Retrieval is "self-directed." The agent decides when to pull data from "Disk" into "RAM" based on the current conversation flow.
• Use Case: Ideal for "Stateful Agents" that need to maintain a consistent persona and persist specific instructions or core facts across very long sessions.

### C. Mem0: The Universal Memory Layer
Mem0 provides a Multi-Level Memory architecture designed for efficiency and personalization. It offers two variants: a vector-based standard model and a graph-based Mem0g.
• Structure:
    ◦ Mem0 (Standard): Uses a vector database to store "dense natural language" memories. It segregates memory into User, Session, and Agent scopes to enable adaptive personalization.
    ◦ Mem0g (Graph): Extends the standard model by structuring data as a graph G=(V,E,L) (Nodes, Edges, Labels) to capture complex relationships.
• Management:
    ◦ Extraction Phase: An LLM extracts salient facts (or entities/triplets in Mem0g) from message pairs.
    ◦ Update Phase: A "Conflict Detector" and "Update Resolver" determine whether to ADD, UPDATE, or DELETE memories based on semantic similarity and contradictions.
• Retrieval:
    ◦ Mem0: Uses dense vector search to retrieve top-k relevant memories.
    ◦ Mem0g: Uses a dual strategy: Entity-Centric (subgraph search) and Semantic Triplet (vector matching of relationships).
• Performance: Mem0 claims to be 91% faster and use 90% fewer tokens than full-context approaches, optimizing for production latency.

---

## 3. Comparison Matrix

| Feature | Zep | Letta (MemGPT) | Mem0 / Mem0g |
|---------|-----|----------------|---------------|
| Core Metaphor | Temporal Knowledge Graph | Operating System (RAM vs. Disk) | Adaptive Personalization Layer |
| Data Structure | Bi-Temporal Graph (Nodes/Edges) | Hierarchical Blocks (Core, Archival) | Vector Store (Mem0) or Graph (Mem0g) |
| Update Mechanism | Real-time, incremental graph updates | Explicit Agent-directed edits (Tool call) | Extraction pipeline with Conflict Detection |
| Retrieval Logic | Hybrid: Graph + Semantic + Keyword | Paging: Swap in/out of context window | Dual: Entity traversal + Semantic Search |
| Key Metric | "Deep Memory Retrieval" & Temporal Reasoning | Self-editing capabilities & Persistence | Latency (p95) & Token Efficiency |
| Primary Strength | Reasoning over how facts change over time | Maintaining complex state & Persona | Production speed & User/Session scoping |

---

## 4. Summary
• Choose Zep if your agent needs to understand the history of data (e.g., "Where did I live last year vs. now?") and requires a graph structure that updates incrementally.
• Choose Letta if you are building an agent that needs to behave like an OS, managing its own context window limits and maintaining a strict, editable persona.
• Choose Mem0 if you need a low-latency, production-ready layer that simply "remembers" user preferences and facts across sessions to personalize the experience without complex setup.