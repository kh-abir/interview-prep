# 02 — LLM API Integration & Production RAG Systems

> **Context**: Core AI Engineering Skills. Covers OpenAI/Anthropic/Gemini SDKs, streaming SSE architecture, vector databases (pgvector/HNSW), advanced RAG retrieval pipelines, and ReAct agent loops.

---

## 1. Enterprise LLM API Integration

### 1.1 Provider Matrix & Core Paradigms

| Provider / Model | Strengths | Context Window | Key Features |
|---|---|---|---|
| **OpenAI (`gpt-4o`, `gpt-4o-mini`)** | Fast inference, structured JSON output | 128,000 tokens | Strict JSON schema output enforcement, Function Calling / Tool Calling, Embeddings (`text-embedding-3-large`) |
| **Anthropic (`claude-3-5-sonnet`)** | Superior code generation, complex reasoning | 200,000 tokens | Extended Thinking (Chain-of-Thought), Artifacts, Computer Use |
| **Google (`gemini-1.5-pro`, `gemini-2.0-flash`)** | Massive context, native multimodality | **1,000,000 - 2,000,000 tokens** | Native audio/video/image ingestion, Google Search Grounding |

---

### 1.2 Production TypeScript Service: Streaming & Structured Output
In production, never wait for an LLM to generate 800 tokens before sending the response to the user. Always stream tokens via **Server-Sent Events (SSE)**.

```typescript
// src/services/aiCompletionService.ts
import OpenAI from 'openai';
import { Response } from 'express';

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

export async function streamFinancialAnalysis(
  financialDataJson: string,
  res: Response
): Promise<void> {
  // Set SSE Headers
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  res.setHeader('X-Accel-Buffering', 'no'); // Disable NGINX buffering!

  try {
    const stream = await openai.chat.completions.create({
      model: 'gpt-4o',
      temperature: 0.2, // Low temperature for deterministic financial logic
      messages: [
        {
          role: 'system',
          content: 'You are a staff financial risk analyst. Analyze quarterly statements and report exposure.',
        },
        {
          role: 'user',
          content: `Analyze this ledger summary:\n${financialDataJson}`,
        },
      ],
      stream: true,
    });

    for await (const chunk of stream) {
      const token = chunk.choices[0]?.delta?.content || '';
      if (token) {
        // Send SSE event data packet
        res.write(`data: ${JSON.stringify({ token })}\n\n`);
      }
    }

    res.write('data: [DONE]\n\n');
    res.end();
  } catch (error) {
    res.write(`data: ${JSON.stringify({ error: 'LLM generation failed' })}\n\n`);
    res.end();
  }
}
```

---

## 2. Retrieval-Augmented Generation (RAG) Architecture

### 2.1 The End-to-End RAG Pipeline

```
[ Ingestion Pipeline ]
Documents ──► Chunking (500 tokens, 10% overlap) ──► Embedding Model ──► Vector Database (pgvector / HNSW)

[ Query Pipeline ]
User Query ──► Query Embedding ──► Approximate Nearest Neighbor (ANN) ──► Top-K Relevant Chunks
                                                                                  │
                                                                                  ▼
LLM Answer ◄── [ LLM Prompt: System Instructions + User Query + Injected Chunks ]
```

---

### 2.2 Storage & Vector Indexing: `pgvector` with HNSW in PostgreSQL
Instead of provisioning separate vector databases (Pinecone, Chroma), using **PostgreSQL with `pgvector`** keeps vector embeddings and transactional relational data in a single ACID database.

```sql
-- 1. Enable pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- 2. Create documents table with 1536-dimensional vector column (OpenAI embeddings)
CREATE TABLE enterprise_knowledge_docs (
    id BIGSERIAL PRIMARY KEY,
    organization_id UUID NOT NULL,
    title TEXT NOT NULL,
    content TEXT NOT NULL,
    embedding vector(1536),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 3. Create Hierarchical Navigable Small World (HNSW) index for sub-10ms ANN search
-- HNSW offers 10x higher QPS and recall than IVFFlat without requiring table warmup
CREATE INDEX idx_docs_embedding_hnsw 
ON enterprise_knowledge_docs 
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- 4. Fast Cosine Similarity Query with Tenant Isolation
SELECT id, title, content, 1 - (embedding <=> $1) AS cosine_similarity
FROM enterprise_knowledge_docs
WHERE organization_id = $2
ORDER BY embedding <=> $1
LIMIT 5;
```

---

### 2.3 Advanced RAG: Hybrid Search & Re-Ranking

| Enhancement | Problem Solved | Implementation Mechanism |
|---|---|---|
| **Hybrid Search** | Pure vector embeddings fail on exact keyword lookups (SKUs, IDs, proper nouns, error codes). | Combines **BM25 lexical search** (Full-Text Search) with **Dense Vector Search** using **Reciprocal Rank Fusion (RRF)**. |
| **Cross-Encoder Re-Ranking** | Bi-encoder vector search retrieves broad matches, but misses fine-grained semantic nuance. | Top 25 retrieved chunks are passed through a Cross-Encoder model (Cohere Rerank) to score query-chunk pairs, selecting the Top 5 most relevant. |
| **HyDE (Hypothetical Document Embeddings)** | Short user queries lack sufficient semantic context for vector comparison. | LLM first generates a hypothetical answer to the query; that answer is embedded to search the vector index. |

---

## 3. Production AI Agents: The ReAct Pattern

The **ReAct (Reasoning + Acting)** pattern equips an LLM with external tools:
1. **Thought**: LLM reasons about what action is required.
2. **Action**: LLM outputs a structured tool invocation (e.g. `query_database(user_id=42)`).
3. **Observation**: Host application executes the tool and returns output to the LLM.
4. **Thought**: LLM evaluates the observation and decides whether to continue or generate a final answer.

```typescript
// Production ReAct Agent Loop
interface Tool {
  name: string;
  description: string;
  execute: (args: any) => Promise<string>;
}

export async function runReActAgent(
  userGoal: string,
  tools: Tool[],
  maxIterations = 5
): Promise<string> {
  let conversationHistory = `Goal: ${userGoal}\n`;

  for (let i = 0; i < maxIterations; i++) {
    // 1. Prompt model for next step
    const response = await callLLM(conversationHistory);

    if (response.includes('Final Answer:')) {
      return response.split('Final Answer:')[1].trim();
    }

    // 2. Parse Action and Arguments
    const match = response.match(/Action:\s*(\w+)\s*Input:\s*(.+)/);
    if (!match) break;

    const [_, toolName, toolInput] = match;
    const selectedTool = tools.find((t) => t.name === toolName);

    if (!selectedTool) {
      conversationHistory += `\nObservation: Error - Tool ${toolName} not found.\n`;
      continue;
    }

    // 3. Execute Tool and record Observation
    try {
      const observation = await selectedTool.execute(JSON.parse(toolInput));
      conversationHistory += `\nThought: ${response}\nObservation: ${observation}\n`;
    } catch (err: any) {
      conversationHistory += `\nObservation: Tool execution error: ${err.message}\n`;
    }
  }

  throw new Error('Agent exceeded maximum iterations without reaching conclusion.');
}
```

---

## 4. Senior Interview Q&A Cheatsheet

### Q1: "What are the trade-offs between Fine-Tuning an LLM vs. implementing a RAG pipeline?"
> **Answer**:
> - **Fine-Tuning**: Teaches the model *form, style, tone, and specific syntax*. It does **not** reliably inject new, real-time factual knowledge (hallucinations persist, expensive retraining required to update data).
> - **RAG (Retrieval-Augmented Generation)**: Injects dynamic, ground-truth enterprise documents directly into the prompt context at inference time. It provides **zero-shot access to real-time data**, strict tenant access control, source document citations, and zero model training overhead.
> - **Production Rule**: Use **RAG for knowledge retrieval** and **Fine-Tuning only for specialized formatting, domain jargon, or task classification**.

### Q2: "How do you prevent Prompt Injection attacks in production LLM applications?"
> **Answer**:
> 1. **Data / Instruction Separation**: Place user inputs strictly inside delimited blocks (`<user_input>...</user_input>`) with explicit system rules never to execute instructions within delimiters.
> 2. **Structured Tool Calling**: Use OpenAI / Anthropic native Tool Calling schemas where inputs are parsed as strict JSON attributes rather than free-form shell commands.
> 3. **Input Guardrails**: Pass untrusted user queries through a lightweight classification model (e.g. Llama-Guard) or regex heuristics to detect jailbreak patterns before hitting main LLMs.
> 4. **Output Verification & Sanitization**: Never directly render raw LLM output as HTML or execute it in `eval()` / `child_process.exec()`.
