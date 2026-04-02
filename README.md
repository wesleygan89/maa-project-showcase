# Multi-Agent Automation (MAA)

> A private project by Wesley Gan — a fully autonomous, multi-agent workflow orchestrator powered by GitHub Copilot SDK, LangGraph state management, Corrective RAG (CRAG), Arize Phoenix observability, and a modular on-demand Skills system.

---

## What Is This?

**MAA** transforms the traditional software development lifecycle by coordinating a pipeline of specialized AI agents — each owning a specific role — without any manual hand-offs between stages.

You describe a feature. The system does the rest.

```
You (PM Task) → BA → Tech Lead → QA → Developer → Architect → Security → DevOps → PM Sign-off
```

This repository is a **public showcase** of the project's architecture, design decisions, and capabilities. The full source code is maintained privately.

---

## The Problem It Solves

Modern AI coding tools help individual developers, but they don't solve the coordination problem:

- Requirements are still written manually
- Architectural decisions are still made in isolation
- QA still waits until after implementation
- Security reviews are often skipped or bolted on last-minute
- Knowledge transfer between roles is lossy and slow

MAA replaces this entire flow with an automated, role-aware agent pipeline that enforces quality gates at every stage.

---

## Key Features

| Feature | Description |
|---------|-------------|
| **3 Workflow Types** | Development (full feature), Spike (research/POC), Code Review (iterative loop) |
| **14+ Specialized Agents** | PM, BA, Tech Lead, Developer (stack-specific), QA, Architect, Security, DevOps, Researcher, and more |
| **LangGraph State Machine** | Workflow pipeline modelled as a typed `StateGraph` with formal reducers, thread-isolated checkpointing, and time-travel debugging |
| **Corrective RAG (CRAG)** | Five-node retrieval pipeline — query generator, retriever, LLM grader, query rewriter, executor — fetches only the most relevant reference docs per agent stage |
| **Arize Phoenix Observability** | Full OpenTelemetry tracing to a local Phoenix instance — visualise every workflow → stage → LLM call → tool call span in real time |
| **Dynamic Skills System** | 10 modular domain-knowledge packs (TDD, Security, Architecture, etc.) injected only when relevant to the current agent + stage + workflow |
| **Multi-Repository Support** | Switch between Next.js, NestJS, .NET, and Node.js/TypeORM projects with a single command |
| **LangGraph Checkpointing** | `FileSystemSaver` implements `BaseCheckpointSaver` — full checkpoint history per thread, resume from any prior state |
| **Subtask Orchestration** | Tech Lead auto-decomposes complex tasks into subtasks, each with their own QA → Dev → Review TDD cycle |
| **Interactive Corrections** | Type mid-workflow corrections to redirect agents without restarting |
| **Live Streaming Dashboard** | Real-time browser output at `localhost:3030` — watch agents work in real time |
| **Deferral Detection** | Detects when an agent tries to skip or defer work and forces re-execution |
| **Quality Gates** | Intent analysis validates output quality before passing context to the next agent |
| **Rate Limit Handling** | Automatic retry with exponential backoff for API rate limits |

---

## How It Works

### 1. You Write a PM Task

Create a `pm-task.md` defining the feature in plain language:

```markdown
## Feature: User Profile Page

### Description
Create a user profile page with avatar upload, bio editing, and activity history.

### Acceptance Criteria
- Users can view and edit their profile
- Avatar upload with image cropping
- Activity history with pagination
```

### 2. The Pipeline Runs Automatically

Each agent receives the output of the previous agent as context. No manual prompting required between stages.

```
PM_PRODUCT_DEFINITION  →  defines scope and success criteria
BA_REQUIREMENTS        →  produces a BRD with Gherkin user stories
TECH_LEAD_ASSESSMENT   →  architecture decisions + subtask breakdown
QA_TEST_CREATION       →  writes tests BEFORE implementation (TDD)
DEV_IMPLEMENTATION     →  implements features against the tests
QA_VALIDATION          →  validates implementation passes all tests
ARCHITECT_REVIEW       →  architecture review and recommendations
SECURITY_REVIEW        →  vulnerability and security assessment
DEVOPS_DEPLOYMENT      →  CI/CD and infrastructure configuration
PM_FINAL_APPROVAL      →  final review and sign-off
```

### 3. Output Is Structured and Auditable

Every agent produces structured output that flows into the next. The full pipeline run is logged and streamed live.

---

## Orchestrator & Agent Architecture

The `workflow-manager.js` orchestrator drives the system. The pipeline is modelled as a LangGraph `StateGraph`, context is retrieved per-stage via Corrective RAG, every LLM call is traced to Arize Phoenix, and agents receive on-demand domain knowledge from the Skills system.

```mermaid
graph TB
    subgraph Orchestrator["🧠 Orchestrator — workflow-manager.js"]
        WM[Workflow Manager]
        WM -->|builds prompts| PB[Prompt Builder]
        WM -->|validates output| IA[Intent Analyzer]
        WM -->|retries on limits| RL[Rate Limiter]
        WM -->|detects deferrals| DD[Deferral Detector]
        WM -->|queues corrections| CH[Correction Handler]
        WM -->|streams output| WL[Workflow Logger]
        WM -->|loads agent .md files| AL[Agent Loader]
    end

    subgraph LG["🔗 LangGraph StateGraph"]
        SG[StateGraph]
        SG --> RN[routeStage node]
        SG --> QG[queryGenerator node]
        SG --> RET[retriever node]
        SG --> GR[grader node]
        SG --> QRW[queryRewrite node]
        SG --> EX[executeStage node]
        SG --> PP[postProcess node]
        FSC[FileSystemSaver\ncheckpointer]
        SG -->|persists every transition| FSC
    end

    subgraph RAG["🔍 RAG Pipeline"]
        DC[Document Chunker]
        TF[TF-IDF Embeddings]
        VS[InMemory VectorStore]
        RR[RAGRetriever]
        IR[Instruction Retriever]
        DC --> TF --> VS
        VS --> RR
        IR -->|dynamic coding standards| RR
    end

    subgraph SK["🎓 Skills System"]
        SL[Skill Loader]
        TDD[TDD Workflow]
        SECSK[Security Review]
        ARCH2[Architecture Review]
        TDSK[Task Decomposition]
        SL --> TDD & SECSK & ARCH2 & TDSK
    end

    subgraph PHX["📊 Arize Phoenix Tracing"]
        OT[OpenTelemetry OTLP]
        PH[Phoenix localhost:6006]
        WS[Workflow Span]
        SS2[Stage Span]
        LS[LLM Span]
        TS[Tool Span]
        OT --> PH
        WS --> SS2 --> LS & TS
    end

    subgraph SDK["⚙️ GitHub Copilot SDK"]
        CS[CopilotClient]
    end

    subgraph AgentDefs[".github/agents/ — Agent Definitions"]
        PM["🎯 Strategic PM"]
        BA["📋 Lead BA"]
        TL["🏗️ Tech Lead"]
        QA["🧪 QA Engineer"]
        DEV["💻 Developer"]
        ARCH["🏛️ Architect"]
        SEC["🔒 Security Eng."]
        DEVOPS["🚀 DevOps"]
        RES["🔬 Researcher"]
    end

    subgraph Supporting["🛠️ Supporting Systems"]
        STREAM[Stream Server\nlocalhost:3030]
        CONF[Confluence Handler]
        SPEC[Spec-Kit Handler]
        PC[Prompt Compressor]
    end

    WM -->|runs pipeline via| LG
    LG -->|retrieves context per stage| RAG
    LG -->|injects skills per stage| SK
    LG -->|traces every span| PHX
    LG -->|invokes agents via| CS
    CS -->|uses definitions from| AgentDefs
    WM --> Supporting
```

---

## Workflow Types

### Development Workflow

Full lifecycle pipeline for feature development. Runs 10 stages end-to-end. If the Tech Lead identifies a complex feature, it automatically decomposes it into subtasks — each with its own QA → Dev cycle — before the pipeline continues.

```mermaid
flowchart TD
    START([📝 pm-task.md]) --> PM

    PM["🎯 Stage 1\nStrategic Product Manager\nDefine scope & success criteria"]
    BA["📋 Stage 2\nLead Business Analyst\nBRD + Gherkin user stories"]
    TL["🏗️ Stage 3\nTech Lead\nArchitecture decisions\n& task decomposition"]
    QA1["🧪 Stage 4\nQA Engineer\nWrite tests BEFORE implementation\n(TDD)"]
    DEV["💻 Stage 5\nDeveloper (stack-specific)\nImplement against tests"]
    QA2["✅ Stage 6\nQA Engineer\nValidate all tests pass"]
    ARCH["🏛️ Stage 7\nSystem Architect\nArchitecture review"]
    SEC["🔒 Stage 8\nSecurity Engineer\nVulnerability assessment"]
    DEVOPS["🚀 Stage 9\nDevOps Engineer\nCI/CD & deployment config"]
    PM2["🎯 Stage 10\nStrategic Product Manager\nFinal review & sign-off"]
    END([✅ Feature Complete])

    PM --> BA --> TL

    TL -->|Simple feature| QA1
    TL -->|Complex feature| ORCH

    subgraph ORCH["🔄 Tech Lead Subtask Orchestration"]
        direction TB
        ST1["Subtask 1\nQA → Dev → Review"]
        ST2["Subtask 2\nQA → Dev → Review"]
        STN["Subtask N\nQA → Dev → Review"]
        ST1 --> ST2 --> STN
    end

    ORCH --> QA2
    QA1 --> DEV --> QA2
    QA2 --> ARCH --> SEC --> DEVOPS --> PM2 --> END

    style ORCH fill:#1e3a5f,stroke:#4a90d9,color:#fff
    style START fill:#2d5a27,stroke:#5a9e52,color:#fff
    style END fill:#2d5a27,stroke:#5a9e52,color:#fff
```

---

### Spike Workflow

Research-first pipeline for technical investigation and proof-of-concept work. Phases 3 runs Architect and Security analysis in parallel to maximise throughput.

```mermaid
flowchart TD
    START([📝 Spike Task]) --> P1

    subgraph P1["Phase 1 — Planning"]
        SD["🏗️ Tech Lead\nDefine spike scope\n& success criteria"]
        RB["🏗️ Tech Lead\nBreak spike into\nfocused research subtasks"]
        SD --> RB
    end

    subgraph P2["Phase 2 — Research"]
        TR["🔬 Technical Researcher\nConduct investigation\n& gather findings"]
        RO["🏗️ Tech Lead\nOrchestrate research subtasks\nwith Technical Researcher"]
        TR --> RO
    end

    subgraph P3["Phase 3 — Parallel Analysis"]
        direction LR
        AA["🏛️ System Architect\nArchitecture Analysis"]
        SA["🔒 Security Engineer\nSecurity Assessment"]
    end

    subgraph P4["Phase 4 — Prototype"]
        POC["💻 Developer\nProof-of-Concept\nimplementation"]
    end

    subgraph P5["Phase 5 — Synthesis"]
        FS["🏗️ Tech Lead\nSynthesize all findings\n& recommendations"]
        BI["🎯 Strategic PM\nBusiness impact\n& strategic direction"]
        FS --> BI
    end

    subgraph P6["Phase 6 — Documentation"]
        SR["📋 Lead BA\nComprehensive spike report"]
        FR["🏛️ System Architect\nFinal recommendations\n& next steps"]
        SR --> FR
    end

    END([📄 Spike Report])

    P1 --> P2 --> P3
    P3 --> P4 --> P5 --> P6 --> END

    style P3 fill:#1e3a5f,stroke:#4a90d9,color:#fff
    style START fill:#2d5a27,stroke:#5a9e52,color:#fff
    style END fill:#2d5a27,stroke:#5a9e52,color:#fff
```

---

### Code Review Workflow

Self-improving iterative loop (up to 10 rounds). The Code Reviewer produces structured change requests; the Developer addresses each one. The loop exits on explicit approval or after 10 iterations.

```mermaid
flowchart TD
    START([💻 Code to Review]) --> IMPL

    IMPL["💻 Developer\nInitial Implementation"]
    IMPL --> CR1

    CR1["👁️ Code Reviewer\nReview round 1\nProduce structured change requests"]

    CR1 --> DECISION1{Approved?}

    DECISION1 -->|✅ LGTM / No change requests| END
    DECISION1 -->|❌ Change requests raised| FIX

    subgraph LOOP["🔄 Fix → Re-Review Loop (up to 10 iterations)"]
        direction TB
        FIX["💻 Developer\nAddress all change requests"]
        CRN["👁️ Code Reviewer\nRe-review\nProduce new change requests or approval"]
        DECISIONN{Approved\nor max\niterations?}

        FIX --> CRN --> DECISIONN
        DECISIONN -->|❌ Still has issues\nand iterations remaining| FIX
    end

    DECISIONN -->|✅ Approved| END
    DECISIONN -->|🛑 Max 10 iterations reached| MAXEND

    END([✅ Code Approved & Merged])
    MAXEND([⚠️ Max Iterations — Manual Review Required])

    style LOOP fill:#1e3a5f,stroke:#4a90d9,color:#fff
    style END fill:#2d5a27,stroke:#5a9e52,color:#fff
    style MAXEND fill:#5a2d27,stroke:#9e5252,color:#fff
    style START fill:#2d5a27,stroke:#5a9e52,color:#fff
```

---

## Agent Roster

| Agent | Role |
|-------|------|
| Strategic Product Manager | Define requirements, acceptance criteria, final sign-off |
| Lead Business Analyst | BRD, user stories (Gherkin), spec quality gate |
| Tech Lead | Architecture decisions, task decomposition, subtask orchestration |
| QA Automation Engineer | TDD test creation and implementation validation |
| Senior Next.js Developer | Next.js App Router frontend implementation |
| Senior NestJS Developer | NestJS microservice development |
| Senior .NET Developer | .NET Core Web API development |
| Senior Node.js Developer | Node.js/Express/TypeORM backend development |
| System Architect | Architecture review and long-term recommendations |
| Security Engineer | Security assessment, vulnerability scanning |
| DevOps Engineer | CI/CD, deployment manifests, infrastructure |
| Technical Researcher | Spike research and technical investigation |
| Code Reviewer | Code quality, maintainability, best practices |
| Workflow Specialist | Workflow optimization and performance metrics |

Agents are defined as `.agent.md` files using VS Code's Copilot agent format, making them usable directly in the VS Code chat as well as within the automated pipeline.

---

## Multi-Repository Support

MAA is designed to operate against any target codebase. A central `repo-config.json` describes each project:

```json
{
  "activeRepo": "my-web-app",
  "repositories": {
    "my-web-app": {
      "stack": "nextjs",
      "agents": { "developer": "Senior Next.js Developer" },
      "instructions": ["nextjs.instructions.md", "reactjs.instructions.md"],
      "testCommand": "npm test"
    },
    "my-api": {
      "stack": "dotnet",
      "agents": { "developer": "Senior .NET Developer" },
      "instructions": ["dotnet.instructions.md", "csharp.instructions.md"]
    }
  }
}
```

Switching projects is a single command:

```powershell
.\switch-repo.ps1 -Repo my-api
.\run-workflow.ps1
```

### Supported Stacks

| Stack | Developer Agent |
|-------|----------------|
| Next.js | Senior Next.js Developer |
| NestJS | Senior NestJS Developer |
| .NET Core | Senior .NET Developer |
| Node.js + TypeORM | Senior Node.js TypeORM Developer |

---

## Coding Standards

Every agent is automatically loaded with the relevant coding standards for the active repository's stack. Standards are defined as `.instructions.md` files covering:

- Architecture patterns (hexagonal, clean, layered)
- Component and module conventions
- Testing strategy and coverage requirements
- Security and API design guidelines
- Git workflow and PR standards
- SCSS styling conventions

This ensures agents produce output that is idiomatic and consistent with the target project — not generic boilerplate.

---

## LangGraph State Machine

The entire workflow pipeline is modelled as a **LangGraph `StateGraph`** — replacing an ad-hoc plain-object pipeline with a formally typed, checkpointed, node-based graph.

### Graph Topology

```mermaid
flowchart LR
    S([START]) --> RS

    RS[routeStage\nFind next stage\nor signal END]
    QG[queryGenerator\nCache-check gate]
    RET[retriever\nTF-IDF RAG fetch]
    GR[grader\nLLM relevance\njudge]
    QRW[queryRewrite\nImprove query\n& retry]
    EX[executeStage\nCopilot SDK\nagent call]
    PP[postProcess\nUpdate stage status\n& state merge]
    E([END])

    RS -->|stages remain| QG
    RS -->|all done| E

    QG -->|cache hit| EX
    QG -->|retrieval needed| RET

    RET --> GR
    GR -->|relevant| EX
    GR -->|irrelevant + rewrites left| QRW
    GR -->|irrelevant + max rewrites| EX

    QRW -->|rewritten query| RET

    EX --> PP --> RS
```

### State Schema

The workflow state uses `Annotation.Root` — a typed channel system where every field has an explicit **reducer** controlling how partial updates are merged:

| Field | Reducer | Purpose |
|-------|---------|---------|
| `stages` | last-write-wins | Full stage array (mutated by dynamic injection) |
| `agentOutputs` | merge-objects | Keyed by agent+stageId — never overwrites prior outputs |
| `ragContext` | merge-objects | Per-stage retrieved context blocks |
| `ragQueriesCompleted` | append-list | Tracks which stages have been successfully retrieved |
| `corrections` | append-list | Human corrections queued mid-workflow |
| `retryContext` | last-write-wins | Rate-limit retry metadata |

### FileSystemSaver — Custom LangGraph Checkpointer

A custom `BaseCheckpointSaver` implementation persists the full checkpoint history per thread to disk as JSON. This enables:
- **Resume** from any prior checkpoint, not just the last
- **Time-travel debugging** — inspect the state at any stage in history
- **Thread isolation** — parallel workflow runs don't share state

```
langgraph-checkpoints/
  <thread-id>.json   ← array of { checkpoint, metadata, parentId, writes }
```

---

## Corrective RAG (CRAG) Pipeline

Every agent stage is enriched with the most relevant reference documentation via a **Corrective RAG** sub-graph embedded in the LangGraph workflow. Standard RAG retrieves blindly — CRAG retrieves, grades the result, and rewrites the query if the context is irrelevant.

### CRAG Flow

```mermaid
sequenceDiagram
    participant G as LangGraph
    participant QG as queryGenerator
    participant R as retriever
    participant GR as grader (LLM)
    participant QR as queryRewriter
    participant EX as executeStage

    G->>QG: Is this stage already retrieved?
    alt Cache hit (ragQueriesCompleted)
        QG-->>EX: skip retrieval
    else Not yet retrieved
        QG->>R: fetch top-K chunks
        R->>GR: retrieved context + stage details
        GR->>GR: structured verdict {relevant, score, reason}
        alt Relevant
            GR-->>EX: inject context into agent prompt
        else Irrelevant + rewrites remaining
            GR->>QR: rejection reason
            QR->>R: improved query
        else Irrelevant + max rewrites reached
            GR-->>EX: proceed with best available context
        end
    end
```

### RAG Implementation

| Component | Implementation |
|-----------|---------------|
| **Embeddings** | `TfIdfEmbeddings` — pure JS TF-IDF with stop-word filtering, zero external dependencies |
| **Vector Store** | `InMemoryVectorStore` — cosine similarity search, brute-force for corpora < 10k chunks |
| **Chunker** | `document-chunker.js` — Markdown-aware header-boundary splitting |
| **Retriever** | `RAGRetriever` — singleton, indexes at startup, retrieves per stage |
| **Instruction Retriever** | `instruction-retriever.js` — signal-map approach: detects tech stack from branch/task context and injects only the applicable coding standards |

The architecture is intentionally **pluggable** — swap `TfIdfEmbeddings` for `@huggingface/transformers` or OpenAI embeddings by implementing the same interface.

---

## Arize Phoenix Observability

Every workflow run is fully traced using **OpenTelemetry** with [**OpenInference**](https://github.com/Arize-ai/openinference) semantic conventions, exported to a local [**Arize Phoenix**](https://github.com/Arize-ai/phoenix) instance.

### Trace Hierarchy

```mermaid
graph TD
    W["🔗 CHAIN span\nworkflow:development\n(session_id = task_id)"]
    A1["🤖 AGENT span\nTech Lead → TECH_LEAD_ASSESSMENT\n(model, subtask info)"]
    A2["🤖 AGENT span\nQA Engineer → QA_TEST_CREATION"]
    AN["🤖 AGENT span\n..."]
    L1["🧠 LLM span\ncreateSession call\n(model, input/output, token counts)"]
    T1["🔧 TOOL span\nread_file"]
    T2["🔧 TOOL span\ngrep_search"]
    T3["🔧 TOOL span\nrun_in_terminal"]
    RET2["🔍 RETRIEVER span\nRAG retrieval"]

    W --> A1 & A2 & AN
    A1 --> L1
    L1 --> T1 & T2 & T3
    A2 --> RET2
```

### What Each Span Captures

| Span type | Key attributes |
|-----------|---------------|
| Workflow (CHAIN) | `session.id`, `workflow.type`, `input.value` (task description) |
| Stage (AGENT) | `agent.name`, `llm.model_name`, subtask index, stage ID |
| LLM call | `llm.input_messages`, `llm.output_messages`, estimated token counts |
| Tool call | `tool.name`, `tool.parameters`, `output.value` |
| Retriever | Retrieved chunks, query, similarity scores |

### Viewing Traces

```powershell
# Start Phoenix UI (requires Arize Phoenix installed in .venv)
.\start-phoenix.ps1
# Opens http://localhost:6006
```

Override the collector endpoint via `PHOENIX_COLLECTOR_ENDPOINT` env var for remote Phoenix instances.

---

## Dynamic Skills System

**Skills** are modular, on-demand domain-knowledge packets. Unlike coding instructions (which are always-on rules loaded for the stack), skills are injected into an agent's prompt **only when relevant** — matched by agent role, stage type, and workflow type.

### Skill Activation

```mermaid
flowchart LR
    SL[Skill Loader\nscans .github/skills/]
    MATCH{Match:\nagent + stage\n+ workflow?}
    INJECT[Inject skill body\ninto agent prompt]
    SKIP[Skip — not relevant]

    SL --> MATCH
    MATCH -->|yes| INJECT
    MATCH -->|no| SKIP
```

### Available Skills (10 packs)

| Skill | Agents | Stages |
|-------|--------|--------|
| **TDD Workflow** | QA Engineer | `QA_TEST_CREATION` |
| **Security Review** | Security Engineer | `SECURITY_REVIEW`, `SECURITY_ASSESSMENT` |
| **Architecture Review** | System Architect | `ARCHITECT_REVIEW`, `ARCHITECTURE_ANALYSIS` |
| **Task Decomposition** | Tech Lead | `TECH_LEAD_ASSESSMENT` |
| **Code Implementation** | All Developers | `DEV_IMPLEMENTATION`, subtask dev stages |
| **Requirements Analysis** | Lead BA | `BA_REQUIREMENTS` |
| **Product Definition** | Strategic PM | `PM_PRODUCT_DEFINITION` |
| **Review Decision** | Code Reviewer, Tech Lead | code review stages |
| **Technical Research** | Researcher, Tech Lead | spike research stages |
| **Deployment Readiness** | DevOps Engineer | `DEVOPS_DEPLOYMENT` |

### Skill File Format

```yaml
---
name: TDD Workflow
description: Red-Green-Refactor cycle guidance for QA stages
agents: [QA Automation Engineer]
stages: [QA_TEST_CREATION]
workflows: [development]
priority: 10
---

# Skill content — injected verbatim into the agent's prompt...
```

Skills are plain Markdown with YAML frontmatter — adding new domain knowledge requires creating one file, no code changes.

---

## Under The Hood — Engineering Depth

This is where MAA goes well beyond a simple "call agents in sequence" script. Several non-trivial problems had to be solved.

---

### Dynamic Pipeline — Stages Injected at Runtime

The workflow stage list is **not static**. After the Tech Lead completes its assessment, it can instruct the orchestrator to dynamically inject new subtask stages into the running pipeline — each with its own QA → Dev → Tech Review cycle — before the main pipeline resumes:

```mermaid
flowchart LR
    A[Static Pipeline\n10 predefined stages] -->|Tech Lead decomposes| B[Runtime Injection\nsubtask stages inserted\nat current index]
    B --> C[Dynamic Pipeline\n10 + N×3 stages\nexecuted in order]
```

The injector resolves which developer agent to assign through **5 fallback tiers**: explicit tech-stack in control block → explicit agent name → `**Assigned Developer:**` pattern in Tech Lead output → persisted assignment from earlier stage → repo config default.

---

### Three-Tier Intent Routing

The Tech Lead's orchestration decisions are parsed through a three-tier strategy — tried in order until one succeeds:

| Tier | Method | How it works |
|------|--------|--------------|
| **1 — Explicit control block** | Structured JSON in output | Tech Lead emits a `WORKFLOW_CONTROL` JSON block; parsed directly |
| **2 — Natural language analysis** | Regex intent patterns | Detects phrases like _"returning to developer"_, _"code approved"_, _"orchestrating subtask 3"_ |
| **3 — Stage heuristics** | Rule-based fallback | Safe defaults based on stage type if no signal found in output |

This means agents don't need to produce perfectly structured output — the system understands natural language routing decisions.

---

### Spec-Kit: Human-in-the-Loop Clarification

The BA agent implements GitHub's Spec-Kit methodology. If requirements are ambiguous, the BA emits a structured clarification block and the **entire workflow pauses**:

```
<<<SPEC_KIT_CLARIFICATION>>>
QUESTIONS:
1. What is the expected behaviour when a user uploads a file > 10MB?
2. Should activity history persist after account deletion?
SEVERITY: blocking
CONTEXT: These gaps prevent writing accurate Gherkin acceptance criteria.
<<<END_SPEC_KIT_CLARIFICATION>>>
```

The orchestrator detects this signal, presents the questions in the terminal, waits for the user to type answers and `DONE`, then injects the answers as corrections and **re-runs the BA stage** with the clarified context. Up to 2 rounds allowed before the workflow proceeds anyway.

---

### Self-Healing Review Loops

When the Security Engineer or System Architect **rejects** a deliverable, the orchestrator doesn't just log the failure — it automatically injects a `TECH_LEAD_FIX` stage followed by a re-review stage into the live pipeline:

```mermaid
flowchart LR
    ARCH[Architect Review] -->|rejected| FIX[Injected:\nTech Lead Fix Stage]
    FIX --> REREVIEW[Injected:\nArchitect Re-review]
    REREVIEW -->|approved| NEXT[Next Stage]
    REREVIEW -->|rejected again| FIX
```

This loop is distinct from the subtask loop — fix-orchestration stages skip the QA → Dev cycle and escalate directly back to the reviewer.

---

### Context Management at Scale

Long pipelines accumulate large amounts of prior agent output as context. Two mechanisms prevent token limits from breaking later stages:

**1. Context offloading** — sections over 5KB (prior agent outputs, reference docs, repo structure) are written to disk and replaced with a file reference in the prompt. The agent reads the file via its tools rather than receiving it inline.

**2. LLMlingua compression pipeline** — if the prompt still exceeds 50KB after offloading, it is passed through a compression pipeline (LLMlingua → Grok) that reduces token count while preserving semantic meaning. Compression stats are logged per stage.

Both are transparent to the agents and require no changes to agent definitions.

---

### Model Routing Per Agent Role

Not all agents use the same model. The orchestrator routes each agent to the optimal model for its role:

| Agent type | Model | Rationale |
|-----------|-------|-----------|
| PM, BA, Tech Lead, Architect | `grok-code-fast-1` | Planning and analysis — fast, lower cost |
| Developer, QA, Security, DevOps | `claude-sonnet-4.6` | Code generation and implementation — higher quality |

This is resolved per session, not globally — the same pipeline can use both models in a single run.

---

## Advanced Capabilities

### Checkpoint & Resume
Every completed stage is checkpointed to disk. If a workflow is interrupted (network issue, rate limit, or manual stop), resume exactly where it left off:

```powershell
.\run-workflow.ps1 -Resume
```

### Interactive Corrections
While the pipeline is running you can type a correction into the terminal. The correction handler queues it and injects it as context for the next agent — no restart required.

### Deferral Detection
Agents sometimes produce outputs like _"I'll leave this for the developer..."_. The deferral detector identifies these patterns and forces the agent to complete its assigned responsibility before passing control forward.

### Intent Analysis (Quality Gates)
After each agent completes, an intent analyzer validates output quality against the expected stage deliverables. Low-quality or incomplete outputs are flagged before context is passed to the next agent.

---

## GitHub Copilot SDK Integration

MAA is built directly on top of the `@github/copilot-sdk` — the programmatic interface to GitHub Copilot's model and tool execution layer. This is what makes fully automated, headless agent execution possible without any UI interaction.

### How the SDK Is Used

```mermaid
sequenceDiagram
    participant WM as Workflow Manager
    participant SDK as CopilotClient (SDK)
    participant S as Session
    participant A as Agent (e.g. Tech Lead)
    participant TR as Target Repository

    WM->>SDK: new CopilotClient({ autoStart: true })
    SDK-->>WM: client ready

    loop For each workflow stage
        WM->>SDK: client.createSession({ model, systemMessage, workingDirectory })
        SDK-->>WM: session

        WM->>WM: buildAgentPrompt() — inject context, corrections, compressed history
        WM->>S: session.sendMessage(prompt)

        S-->>WM: stream assistant.message events (real-time chunks)
        S->>TR: tool calls (read_file, grep_search, run_in_terminal, etc.)
        TR-->>S: tool results
        S-->>WM: idle event — agent done

        WM->>WM: Intent analysis — validate output quality
        WM->>WM: Checkpoint state to disk
        WM->>WM: Pass output as context to next stage
    end

    WM->>SDK: client.stop()
```

### Key SDK Interactions

| Interaction | Details |
|-------------|---------|
| **Session per agent** | Each pipeline stage creates a fresh `client.createSession()` with the agent's system message injected as `type: "replace"` |
| **Model selection** | Planning/analysis agents (PM, BA, Tech Lead, Architect) use `grok-code-fast-1`; implementation agents use `claude-sonnet-4.6` |
| **Working directory** | Set to the target repository path so file tools operate on the correct codebase |
| **Streaming** | `assistant.message` events deliver content chunks in real-time; forwarded to the live dashboard and terminal |
| **Idle detection** | `idle` event signals the agent has finished all tool calls and produced its final output |
| **Tool access** | Agents call VS Code Copilot tools (`read_file`, `grep_search`, `semantic_search`, `run_in_terminal`, etc.) to interact with the codebase |

### Why the SDK vs. Chat UI

Using the SDK directly lets the orchestrator:
- **Drive agents programmatically** — no human needs to press send between stages
- **Inject dynamic context** — prior agent outputs, corrections, and compressed history are assembled per-stage before the prompt is sent
- **React to events** — idle detection, tool call counting, and checkpoint triggers are all event-driven
- **Apply quality gates** — intent analysis runs on the raw output before any context is forwarded

---

## Technology Stack

| Component | Technology |
|-----------|-----------|
| Runtime | Node.js 18+ (ESM) |
| AI SDK | `@github/copilot-sdk` v0.1.23 || State Machine | `@langchain/langgraph` v1.2 — typed `StateGraph` with custom `BaseCheckpointSaver` |
| RAG Embeddings | TF-IDF (pure JS, zero dependencies) — pluggable interface |
| Observability | Arize Phoenix + OpenTelemetry OTLP (`@opentelemetry/sdk-trace-node`) |
| OpenInference | `@arizeai/openinference-semantic-conventions` || Models | `claude-sonnet-4.6` (implementation), `grok-code-fast-1` (planning) |
| Scripting | PowerShell 7 |
| Live Streaming | Node.js HTTP server + Server-Sent Events |
| Agent Definitions | VS Code `.agent.md` format |
| Configuration | JSON with JSON Schema validation |

---

## Project Structure

```
MAA/
├── workflow-manager.js            # Core orchestrator
├── workflow-specialist-agent.js   # Workflow analysis & metrics
├── workflow-logger.js             # Structured logging + live streaming
├── checkpoint-utils.js            # Legacy checkpoint utilities
│
├── workflows/
│   ├── development-workflow.js    # Full 10-stage feature pipeline
│   ├── spike-workflow.js          # Research & PoC pipeline
│   ├── code-review-workflow.js    # Iterative review loop
│   └── shared/
│       ├── deferral-detector.js   # Agent deferral detection
│       └── rate-limiter.js        # Exponential backoff retry
│
├── lib/                           # Modular engine components
│   ├── langgraph/                 # 🔗 LangGraph integration
│   │   ├── workflow-graph.js      # StateGraph definition + CRAG nodes wiring
│   │   ├── state-annotation.js    # Typed state channels with reducers
│   │   ├── rag-nodes.js           # CRAG: queryGenerator, retriever, grader, rewriter
│   │   ├── file-checkpointer.js   # FileSystemSaver (BaseCheckpointSaver impl)
│   │   └── langgraph-state-manager.js  # Drop-in state manager API
│   ├── rag/                       # 🔍 RAG pipeline
│   │   ├── rag-retriever.js       # Singleton RAGRetriever (index + retrieve)
│   │   ├── vector-store.js        # InMemoryVectorStore (cosine similarity)
│   │   ├── text-embeddings.js     # TfIdfEmbeddings (pure JS)
│   │   ├── document-chunker.js    # Markdown-aware document chunking
│   │   └── instruction-retriever.js    # Dynamic coding standards via signal-map
│   ├── tracing/                   # 📊 Arize Phoenix observability
│   │   └── phoenix-tracer.js      # OTel OTLP exporter + OpenInference spans
│   ├── skills/                    # 🎓 Skills system
│   │   └── skill-loader.js        # Load & match skills to agent+stage+workflow
│   ├── agents/agent-loader.js     # Loads agent definitions from .agent.md files
│   ├── context/context-loader.js  # Reference doc caching & loading
│   ├── intent/intent-analyzer.js  # Output quality + NLU routing
│   ├── orchestration/             # Tech Lead subtask management
│   ├── prompt/                    # Prompt building & LLMlingua compression
│   ├── interactive/               # Mid-workflow corrections
│   └── spec-kit/                  # BA requirements quality gate
│
├── langgraph-checkpoints/         # Per-thread checkpoint JSON files
│
├── .github/
│   ├── agents/                    # 14+ agent definitions (.agent.md)
│   ├── skills/                    # 10 domain skill packs (SKILL.md)
│   ├── instructions/              # Stack-specific coding standards
│   ├── prompts/                   # Reusable prompt templates
│   ├── chatmodes/                 # Custom VS Code chat modes
│   └── runbooks/                  # Workflow runbooks
│
├── repo-config.json               # Multi-repository configuration
├── repo-config.schema.json        # Config JSON Schema
├── run-workflow.ps1               # Workflow launcher
├── switch-repo.ps1                # Repository switcher
├── start-streaming.ps1            # Live dashboard launcher
└── start-phoenix.ps1              # Arize Phoenix UI launcher
```

---

## Design Decisions

**Why GitHub Copilot SDK?**  
The SDK provides direct, programmatic access to Copilot's model routing and tool execution, allowing the orchestrator to drive agents without UI interaction.

**Why LangGraph for state management?**  
A plain pipeline object works until you need typed reducers, concurrent thread isolation, checkpoint history, and time-travel debugging. LangGraph's `StateGraph` provides all of them as first-class primitives. The custom `FileSystemSaver` means zero external infra — checkpoint history lives on disk.

**Why Corrective RAG instead of always injecting all docs?**  
Naively injecting every reference document into every prompt wastes tokens and degrades output quality with irrelevant context. CRAG ensures agents only receive the specific documentation relevant to their exact stage and task, and rewrites the query when the first retrieval misses.

**Why TF-IDF and not neural embeddings for RAG?**  
Technical documentation is keyword-rich — agent role names, library names, method names — where lexical matching is highly effective and often on par with dense embeddings. TF-IDF runs in milliseconds with zero model downloads, API keys, or GPU requirements. The interface is pluggable: swapping in neural embeddings requires one implementation change.

**Why Arize Phoenix for observability?**  
Phoenix is the leading open-source LLM observability platform. Using the OpenInference semantic conventions ensures traces are structured in a way that any OpenTelemetry-compatible backend (Phoenix, Jaeger, Honeycomb) can consume. Span nesting gives a full causal chain: workflow → stage → LLM call → tool call.

**Why a Skills system instead of larger agent prompts?**  
Static, large prompts accumulate irrelevant content over time. A skill system keeps each agent definition minimal and composable — the Security Engineer's OWASP checklist is only injected when the Security Engineer is actually running a security stage, not in every single prompt.

**Why PowerShell for the launcher?**  
Cross-platform PowerShell 7 provides a familiar scripting surface on Windows-first development environments while remaining portable to Linux/macOS.

**Why `.agent.md` for agent definitions?**  
VS Code's native agent format means every agent can be used interactively in the VS Code chat panel — the same definitions that power the automated pipeline are also available as first-class chat participants during ad-hoc development.

---

## Author

**Wesley Gan** — Software Engineer & AI Automation Enthusiast  
GitHub: [@wesleygan89](https://github.com/wesleygan89)

This project demonstrates how multi-agent orchestration can transform the software development lifecycle — from requirements gathering to deployment — using AI agents with specialized roles, structured workflows, and enforced quality gates.

---

## License

MIT — see [LICENSE](LICENSE) for details.
