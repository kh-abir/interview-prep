# 01 — AI-Assisted Engineering: Modern Agentic Workflows

> **Context**: Competitive Advantage (Highlighted on Resume). Covers autonomous coding assistants, Model Context Protocol (MCP), agentic loops, and production AI engineering practices.

---

## 1. The Landscape of Agentic AI Coding Assistants

### 1.1 Assistant Tiers & Architectural Capabilities

```
┌────────────────────────────────────────────────────────────────────────┐
│  Tier 3: Autonomous Multi-Agent Systems                                │
│  - Google Antigravity, Claude Code (Autonomous CLI)                    │
│  - Features: Tool calling, subagent spawning, MCP, shell execution,    │
│    background scheduled workflows, full repo graph context            │
├────────────────────────────────────────────────────────────────────────┤
│  Tier 2: AI-Native Integrated Development Environments (IDEs)          │
│  - Cursor, Windsurf (Codeium Cascade)                                  │
│  - Features: Multi-file edits, codebase embedding indexing, inline     │
│    diff review, prompt-to-refactor loops                               │
├────────────────────────────────────────────────────────────────────────┤
│  Tier 1: Inline Autocomplete & Chat Assistants                         │
│  - GitHub Copilot, Supermaven                                          │
│  - Features: Next-token code prediction, docstring completions         │
└────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Model Context Protocol (MCP) Architecture
**Model Context Protocol (MCP)** is the open standard created by Anthropic that allows AI agents to securely connect to external tools, databases, and enterprise systems over JSON-RPC (stdio / SSE transport).

```
┌─────────────────────────┐
│     AI Coding Agent     │  (e.g., Google Antigravity / Claude Code)
└───────────┬─────────────┘
            │ JSON-RPC 2.0 (stdio / SSE)
            ▼
┌─────────────────────────┐
│       MCP Server        │  (Exposes Tools, Resources, Prompts)
└─────┬─────────┬─────────┘
      │         │
      ▼         ▼
  [GitHub]  [Postgres]
```

---

## 2. The 5-Stage Agentic Development Lifecycle

The hallmark of a Staff-level engineer using AI is **never blindly prompting or accepting raw output**. A disciplined 5-stage loop must be strictly followed:

```
[ Plan ] ──► [ Context curation ] ──► [ Prompt with constraints ] ──► [ Human diff review ] ──► [ Automated test suite ]
```

### 2.1 Stage 1: Plan & Spec-First Engineering
Before prompting an agent to modify code:
1. Define the **Problem Statement** and **Scope Boundary**.
2. Document the **Data Schema** and **API Contract**.
3. Identify non-negotiable **Security & Performance Constraints** (e.g. "Must execute in $O(N)$", "Must not hold locks across external HTTP calls").
4. Use `/plan` commands to force the AI to produce an architectural blueprint before generating implementation diffs.

### 2.2 Stage 2: Context Window Curation
- **Context Pollution**: Dumping 50 unrelated files into the context window degrades model attention, increases hallucinations, and wastes token budget.
- **Precision Context Injection**: Provide only:
  - The exact target file to edit.
  - The interface / type definitions of interacting services.
  - Schema migrations or domain entity classes.
  - The corresponding unit/integration test file.

### 2.3 Stage 3: Human-in-the-Loop Diff Verification
Every line of AI-generated code must be inspected through a critical lens:
- **Did it introduce subtle security bugs?** (e.g., missing tenant scoping in an ORM query, bypass of strong parameters, unsanitized SQL fragments).
- **Did it degrade computational complexity?** (e.g., replacing an indexed hash lookup with an $O(N^2)$ nested loop).
- **Did it introduce hidden memory allocations?** (e.g., loading an unbounded database table into memory with `.all` instead of `.find_each`).

---

## 3. Delegation Matrix: What to Delegate vs. What to Control

| Engineering Task | Delegation Strategy | Rationale |
|---|---|---|
| **Boilerplate & DTOs** | **Full AI Delegation** | Low cognitive complexity; repetitive syntax structures (Zod schemas, TypeScript types, Prisma models). |
| **Unit & Edge-Case Tests** | **AI-Assisted with Human Curation** | AI excels at brainstorming boundary values (empty arrays, negative numbers, unicode, max integers). |
| **Refactoring & Modernization** | **AI-Assisted with Test Harness** | Safe when backed by comprehensive existing test suites. |
| **Distributed Concurrency & Locks** | **Strict Human Ownership** | AI frequently fails to foresee cross-process race conditions, distributed deadlocks, and clock drift nuances. |
| **Financial Ledgers & Double-Entry Math** | **Strict Human Ownership** | Zero room for rounding discrepancies, floating-point flaws, or non-idempotent mutation loops. |
| **Auth & Cryptographic Boundaries** | **Strict Human Ownership** | High risk of security regression or accidental credential leakage. |

---

## 4. AI-Powered Testing & Production Debugging

### 4.1 Property-Based & Edge-Case Test Generation
Prompting an agent to generate property-based tests surfaces edge cases human developers miss:

```typescript
// Prompt: "Generate comprehensive test cases for this money calculation function,
// including floating point precision issues, negative values, and zero division."

import { calculateProratedRefund } from './billingService';

describe('calculateProratedRefund Edge Cases', () => {
  it('handles standard mid-month cancellation correctly', () => {
    const refund = calculateProratedRefund({
      totalPaidCents: 3000,
      daysUsed: 10,
      totalDaysInBillingCycle: 30,
    });
    expect(refund).toBe(2000); // Exact integer math
  });

  it('prevents IEEE-754 floating point penny rounding errors', () => {
    // 33.333% of 1000 cents should round down safely, never returning NaN or decimal cents
    const refund = calculateProratedRefund({
      totalPaidCents: 1000,
      daysUsed: 1,
      totalDaysInBillingCycle: 3,
    });
    expect(Number.isInteger(refund)).toBe(true);
  });

  it('rejects invalid inputs with negative or inverted date intervals', () => {
    expect(() =>
      calculateProratedRefund({
        totalPaidCents: 1000,
        daysUsed: 35, // daysUsed > totalDays
        totalDaysInBillingCycle: 30,
      })
    ).toThrow('Invalid billing cycle interval');
  });
});
```

### 4.2 Production Post-Mortem Incident Triage
When a critical SEV-1 incident strikes:
1. Extract the **Error Message**, **Backtrace**, and **Last 100 Structured Log Lines**.
2. Query the agent with the relevant service code attached.
3. Prompt: *"Analyze this stack trace against the attached controller and database query. Identify the exact root cause, state whether this is an infrastructure or application bug, and generate a minimal zero-downtime hotfix patch."*

---

## 5. Senior Interview Q&A Cheatsheet

### Q1: "How has your daily engineering velocity changed with agentic AI tools, and how do you ensure code quality doesn't degrade?"
> **Answer**:
> "Agentic AI tools have roughly doubled my feature delivery speed by eliminating mechanical friction: writing repetitive migrations, generating boilerplate CRUD endpoints, creating mock fixtures, and drafting initial test suites.
> However, speed without discipline causes technical debt. I safeguard quality through three non-negotiables:
> 1. **Spec-first discipline**: I write the architecture, database constraints, and API schema contracts myself before initiating agent code generation.
> 2. **Exhaustive diff review**: I audit every generated line for memory allocation bloat, N+1 query patterns, and authorization boundaries.
> 3. **Regression testing**: No AI-generated code is ever merged without passing an automated CI suite with strict linter and type-checker gates."

### Q2: "What is an MCP server and how does it extend an AI assistant's capabilities in a production codebase?"
> **Answer**:
> "MCP (Model Context Protocol) provides a standardized, secure bridge between an LLM agent and external tools or environments. Instead of relying only on static training data or copy-pasted text, an MCP server equips the agent with real-time capabilities: running database queries, querying GitHub pull requests, inspecting Kubernetes pod logs, or running shell test suites directly. It transforms the AI from a passive text completion engine into an active, verified feedback loop."
