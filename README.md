# architecture-diagrams-kubeflow-docs-agent

# GSoC 2026: Agentic RAG on Kubeflow — Architecture Diagrams

---

## 1. Overall System Architecture

High-level view showing how all components connect.

```mermaid
graph TB
    subgraph User Layer
        UI["Chatbot UI<br/>(JS/CSS Widget)"]
    end

    subgraph API Layer
        API["FastAPI Server<br/>(server-https/app.py)"]
    end

    subgraph Agent Layer
        AC["kagent Agent Controller<br/>(K8s CRD)"]
        RL["ReAct Reasoning Loop"]
        AC --> RL
    end

    subgraph Tool Layer - MCP
        MCP["MCP Server<br/>(FastMCP)"]
        T1["search_kubeflow_docs"]
        T2["search_kubeflow_code"]
        T3["search_github_issues"]
        T4["get_reference_architecture"]
        MCP --> T1
        MCP --> T2
        MCP --> T3
        MCP --> T4
    end

    subgraph Storage Layer
        MV["Milvus"]
        C1["Collection:<br/>kubeflow_docs"]
        C2["Collection:<br/>kubeflow_code"]
        C3["Collection:<br/>kubeflow_issues"]
        C4["Collection:<br/>kubeflow_ref_arch"]
        MV --- C1
        MV --- C2
        MV --- C3
        MV --- C4
        MINIO["MinIO<br/>(Object Storage)"]
    end

    subgraph Inference Layer
        LLM["KServe / vLLM<br/>Llama 3.1-8B"]
        EMB["KServe<br/>Embedding Service<br/>(all-mpnet-base-v2)"]
    end

    subgraph Ingestion Layer
        KFP["Kubeflow Pipelines"]
        FULL["Full Pipeline"]
        INCR["Incremental Pipeline"]
        KFP --> FULL
        KFP --> INCR
    end

    subgraph External Sources
        GH["GitHub API<br/>(Repos, Issues, Discussions)"]
        DOCS["Kubeflow Website<br/>(Documentation)"]
    end

    UI -->|"SSE Stream"| API
    API -->|"Query"| AC
    RL -->|"Tool Calls"| MCP
    T1 & T2 & T3 & T4 -->|"Vector Search"| MV
    RL -->|"LLM Inference"| LLM
    MV --- MINIO

    FULL & INCR -->|"Fetch"| GH
    FULL & INCR -->|"Scrape"| DOCS
    FULL & INCR -->|"Embed"| EMB
    FULL & INCR -->|"Index"| MV

    style AC fill:#4A90D9,color:#fff
    style RL fill:#4A90D9,color:#fff
    style MCP fill:#7B68EE,color:#fff
    style LLM fill:#E8A838,color:#fff
    style EMB fill:#E8A838,color:#fff
    style MV fill:#2ECC71,color:#fff
    style KFP fill:#E74C3C,color:#fff
```

---

## 2. Agentic Architecture — ReAct Reasoning Loop

Shows the multi-step reasoning flow where the agent decides whether to retrieve more, refine, or answer.

```mermaid
flowchart TD
    START(["User Query"]) --> PARSE["Parse & Classify Query"]
    PARSE --> DECIDE{"Needs<br/>Retrieval?"}

    DECIDE -->|"No (greeting/simple)"| DIRECT["Direct LLM Response<br/>(No Tool Call)"]
    DECIDE -->|"Yes"| PLAN["Plan: Select Tools<br/>& Formulate Sub-queries"]

    PLAN --> EXEC["Execute Tool Call(s)"]
    EXEC --> RETRIEVE["Retrieve Context<br/>from Milvus Collection(s)"]
    RETRIEVE --> EVAL{"Is Context<br/>Sufficient?"}

    EVAL -->|"Yes"| SYNTH["Synthesize Answer<br/>with Citations"]
    EVAL -->|"No — Refine Query"| REFINE["Refine Search Query<br/>or Decompose Question"]
    EVAL -->|"No — Try Different Tool"| SWITCH["Switch Tool<br/>(docs → code → issues)"]

    REFINE --> EXEC
    SWITCH --> EXEC

    EXEC --> LIMIT{"Max Iterations<br/>Reached? (n=3)"}
    LIMIT -->|"Yes"| FALLBACK["Best-Effort Answer<br/>+ 'Partial Results' Warning"]
    LIMIT -->|"No"| EVAL

    SYNTH --> CITE["Attach Citations<br/>(URLs, file paths, issue #s)"]
    FALLBACK --> CITE
    DIRECT --> OUT(["Stream Response<br/>to User via SSE"])
    CITE --> OUT

    style START fill:#4A90D9,color:#fff
    style OUT fill:#4A90D9,color:#fff
    style EVAL fill:#F39C12,color:#fff
    style DECIDE fill:#F39C12,color:#fff
    style LIMIT fill:#E74C3C,color:#fff
    style SYNTH fill:#2ECC71,color:#fff
    style FALLBACK fill:#E67E22,color:#fff
```

### Sequence Diagram — Multi-Step Query

```mermaid
sequenceDiagram
    participant U as User
    participant API as FastAPI Server
    participant Agent as kagent Controller
    participant LLM as KServe/vLLM
    participant MCP as MCP Server
    participant Milvus as Milvus DB

    U->>API: POST /query (stream=true)
    API->>Agent: Forward query

    Note over Agent: Step 1 — Initial reasoning
    Agent->>LLM: System prompt + user query
    LLM-->>Agent: "I need to search docs for KFP v2 syntax"

    Note over Agent: Step 2 — First tool call
    Agent->>MCP: search_kubeflow_docs("KFP v2 pipeline syntax")
    MCP->>Milvus: Vector search (kubeflow_docs collection)
    Milvus-->>MCP: Top-5 chunks + metadata
    MCP-->>Agent: Retrieved context + source URLs
    API-->>U: SSE event: thinking ("Searching documentation...")

    Note over Agent: Step 3 — Evaluate sufficiency
    Agent->>LLM: "Is this context sufficient?"
    LLM-->>Agent: "Need code examples too"

    Note over Agent: Step 4 — Second tool call
    Agent->>MCP: search_kubeflow_code("kfp v2 pipeline example")
    MCP->>Milvus: Vector search (kubeflow_code collection)
    Milvus-->>MCP: Top-5 code chunks
    MCP-->>Agent: Code context + file paths
    API-->>U: SSE event: tool_call ("Searching source code...")

    Note over Agent: Step 5 — Synthesize
    Agent->>LLM: All context + "Generate cited answer"
    LLM-->>Agent: Final answer with citations
    Agent-->>API: Cited response
    API-->>U: SSE event: answer (streamed response)
    API-->>U: SSE event: done
```

---

## 3. Pluggable Tool Registration Architecture

Shows how new tools can be added without modifying the core agent logic.

```mermaid
graph LR
    subgraph kagent CRD
        AGENT["Agent Resource<br/>(K8s Custom Resource)"]
    end

    subgraph MCP Protocol Layer
        REG["Tool Registry<br/>(FastMCP Server)"]
        SCHEMA["Tool Schema Validator<br/>(JSON Schema)"]
        REG --> SCHEMA
    end

    subgraph Registered Tools
        direction TB
        TOOL1["search_kubeflow_docs<br/>───────────<br/>Collection: kubeflow_docs<br/>Returns: chunks + doc URLs"]
        TOOL2["search_kubeflow_code<br/>───────────<br/>Collection: kubeflow_code<br/>Returns: code + file paths"]
        TOOL3["search_github_issues<br/>───────────<br/>Collection: kubeflow_issues<br/>Returns: issues + links"]
        TOOL4["get_reference_architecture<br/>───────────<br/>Collection: kubeflow_ref_arch<br/>Returns: arch docs + diagrams"]
        TOOL5["🔌 future_tool<br/>───────────<br/>Collection: custom<br/>Implement ToolInterface"]
    end

    AGENT -->|"Declares tools"| REG
    REG --> TOOL1
    REG --> TOOL2
    REG --> TOOL3
    REG --> TOOL4
    REG -.->|"Pluggable"| TOOL5

    subgraph ToolInterface
        direction TB
        IF1["name: str"]
        IF2["description: str"]
        IF3["input_schema: JSONSchema"]
        IF4["execute(params) → ToolResult"]
        IF5["collection_name: str"]
    end

    TOOL5 -.->|"Implements"| ToolInterface

    style AGENT fill:#4A90D9,color:#fff
    style REG fill:#7B68EE,color:#fff
    style TOOL5 fill:#95A5A6,color:#fff,stroke-dasharray: 5 5
    style ToolInterface fill:#F39C12,color:#fff
```

### Tool Registration Flow

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant MCP as MCP Server
    participant Schema as Schema Validator
    participant Agent as kagent Agent CRD
    participant K8s as Kubernetes API

    Dev->>MCP: Register new tool (name, schema, handler)
    MCP->>Schema: Validate input/output schema
    Schema-->>MCP: ✅ Valid

    Dev->>K8s: Update Agent CRD (add tool reference)
    K8s-->>Agent: Agent config reloaded
    Agent->>MCP: Discover available tools
    MCP-->>Agent: Tool list + schemas

    Note over Agent: New tool available<br/>without code changes to agent core
```

---

## 4. KFP Data Ingestion Pipeline — "Golden Data"

Shows the unified pipeline that handles scraping → cleaning → chunking → embedding → indexing.

```mermaid
flowchart LR
    subgraph Sources ["Data Sources"]
        S1["kubeflow/website<br/>(Markdown docs)"]
        S2["kubeflow/kubeflow<br/>(Python source)"]
        S3["kubeflow/pipelines<br/>(Python source)"]
        S4["kubeflow/katib<br/>(Go source)"]
        S5["GitHub Issues<br/>& Discussions"]
        S6["Reference Platform<br/>Architecture"]
    end

    subgraph Fetch ["1. Fetch (KFP Component)"]
        F1["DataSourceFetcher<br/>(Unified Interface)"]
        F1a["fetch_docs()"]
        F1b["fetch_code()"]
        F1c["fetch_issues()"]
        F1d["fetch_ref_arch()"]
        F1 --> F1a & F1b & F1c & F1d
    end

    subgraph Clean ["2. Clean (KFP Component)"]
        CL["Text Processor"]
        CL1["Strip HTML/Markdown boilerplate"]
        CL2["Normalize whitespace"]
        CL3["Extract metadata<br/>(title, path, repo, date)"]
        CL --> CL1 --> CL2 --> CL3
    end

    subgraph Chunk ["3. Chunk (KFP Component)"]
        CH["Smart Chunker"]
        CH1["Semantic chunking<br/>(respect headings/functions)"]
        CH2["Target: 512 tokens/chunk"]
        CH3["Preserve overlap (50 tokens)"]
        CH4["Attach citation metadata<br/>(URL, line numbers)"]
        CH --> CH1 --> CH2 --> CH3 --> CH4
    end

    subgraph Embed ["4. Embed (KServe Service)"]
        EM["Embedding Service<br/>(all-mpnet-base-v2)"]
        EM1["HTTP API call<br/>(not local install)"]
        EM2["768-dim vectors"]
        EM --> EM1 --> EM2
    end

    subgraph Index ["5. Index (KFP Component)"]
        IX["Milvus Indexer"]
        IX1["Route to correct collection"]
        IX2["Upsert vectors + metadata"]
        IX3["Build HNSW index"]
        IX --> IX1 --> IX2 --> IX3
    end

    S1 & S2 & S3 & S4 & S5 & S6 --> F1
    F1 --> CL
    CL --> CH
    CH --> EM
    EM --> IX
    IX --> MV[("Milvus<br/>4 Collections")]

    style F1 fill:#3498DB,color:#fff
    style CL fill:#9B59B6,color:#fff
    style CH fill:#E67E22,color:#fff
    style EM fill:#E74C3C,color:#fff
    style IX fill:#2ECC71,color:#fff
    style MV fill:#2ECC71,color:#fff
```

### Full vs Incremental Pipeline

```mermaid
flowchart TD
    TRIGGER{"Pipeline<br/>Trigger"}
    TRIGGER -->|"Scheduled / Manual"| FULL["Full Pipeline"]
    TRIGGER -->|"Webhook / Cron"| INCR["Incremental Pipeline"]

    FULL --> FF["Fetch ALL sources"]
    FF --> FC["Clean & Chunk ALL"]
    FC --> FE["Embed ALL chunks"]
    FE --> FI["Drop & Rebuild<br/>Milvus Collections"]

    INCR --> IC["Check last_updated<br/>timestamps"]
    IC --> ID{"Changed<br/>files?"}
    ID -->|"Yes"| IF["Fetch CHANGED only"]
    ID -->|"No"| SKIP["Skip — No Action"]
    IF --> ICL["Clean & Chunk changed"]
    ICL --> IE["Embed changed chunks"]
    IE --> IU["Upsert to Milvus<br/>(no full rebuild)"]

    style FULL fill:#E74C3C,color:#fff
    style INCR fill:#2ECC71,color:#fff
    style SKIP fill:#95A5A6,color:#fff
```

---

## 5. KServe Scale-to-Zero Architecture

Shows how the LLM inference scales down when idle and handles cold starts.

```mermaid
stateDiagram-v2
    [*] --> Scaled_to_Zero: No traffic

    Scaled_to_Zero --> Cold_Start: Incoming request
    Cold_Start --> Pod_Scheduling: K8s scheduler assigns node
    Pod_Scheduling --> Model_Loading: vLLM loads Llama 3.1-8B weights
    Model_Loading --> Ready: Model loaded into GPU
    Ready --> Serving: Process requests
    Serving --> Ready: Request completed
    Ready --> Scale_Down_Timer: No traffic for N minutes
    Scale_Down_Timer --> Scaled_to_Zero: Timer expires

    note right of Cold_Start
        Target: < 60s total
        Pod scheduling: ~5s
        Model loading: ~30-45s
        First inference: ~2s
    end note

    note right of Serving
        Warm latency:
        Simple query: < 3s
        Tool call query: < 8s
    end note
```

### KServe Component Diagram

```mermaid
graph TB
    subgraph KServe Infrastructure
        IS["InferenceService<br/>(Llama 3.1-8B)"]
        SR["ServingRuntime<br/>(vLLM)"]
        KPA["Knative Pod Autoscaler<br/>(scale-to-zero enabled)"]
        IS --> SR
        KPA -->|"Manages"| IS
    end

    subgraph Embedding Service
        ES["InferenceService<br/>(all-mpnet-base-v2)"]
        ESR["ServingRuntime<br/>(sentence-transformers)"]
        EKPA["Knative Pod Autoscaler<br/>(min replicas: 1)"]
        ES --> ESR
        EKPA -->|"Manages"| ES
    end

    subgraph GPU Node
        GPU["NVIDIA GPU (>= 16GB)"]
        SR -->|"Uses"| GPU
    end

    API["FastAPI Server"] -->|"/v1/completions"| IS
    MCP["MCP Server"] -->|"/v1/completions"| IS
    KFP["KFP Pipeline"] -->|"/v1/embeddings"| ES

    style IS fill:#E8A838,color:#fff
    style ES fill:#E8A838,color:#fff
    style KPA fill:#4A90D9,color:#fff
    style EKPA fill:#4A90D9,color:#fff
    style GPU fill:#E74C3C,color:#fff
```

---

## 6. Citation Flow — End-to-End Traceability

Shows how citations are tracked from ingestion through to the final response.

```mermaid
flowchart LR
    subgraph Ingestion Time
        DOC["Source Document<br/>kubeflow.org/docs/pipelines/overview"]
        DOC --> META["Extract Metadata"]
        META --> STORE["Store in Milvus"]
    end

    subgraph Metadata Schema
        direction TB
        M1["source_url: 'https://kubeflow.org/...'"]
        M2["source_type: 'docs' | 'code' | 'issue'"]
        M3["repo: 'kubeflow/pipelines'"]
        M4["file_path: 'docs/pipelines/overview.md'"]
        M5["line_start: 45"]
        M6["line_end: 89"]
        M7["last_updated: '2026-03-15'"]
        M8["chunk_id: 'kfp-overview-003'"]
    end

    META --> M1 & M2 & M3 & M4 & M5 & M6 & M7 & M8

    subgraph Query Time
        Q["User Query"] --> SEARCH["Vector Search"]
        SEARCH --> RESULTS["Top-K Results<br/>+ Full Metadata"]
        RESULTS --> LLM["LLM Generates Answer"]
        LLM --> CITE["Citation Formatter"]
    end

    STORE -->|"Indexed with metadata"| SEARCH

    subgraph Final Output
        ANS["Answer Text"]
        C1["📄 [1] Kubeflow Pipelines Overview<br/>kubeflow.org/docs/pipelines/overview"]
        C2["💻 [2] pipeline_utils.py:L45-89<br/>github.com/kubeflow/pipelines/..."]
        C3["🐛 [3] Issue #4521: Pipeline caching<br/>github.com/kubeflow/pipelines/issues/4521"]
    end

    CITE --> ANS
    CITE --> C1 & C2 & C3

    style DOC fill:#3498DB,color:#fff
    style CITE fill:#2ECC71,color:#fff
    style ANS fill:#4A90D9,color:#fff
```

---

## 7. Kubernetes Deployment Topology

Shows where everything runs on the cluster.

```mermaid
graph TB
    subgraph Kubernetes Cluster
        subgraph Namespace: kubeflow
            KFP_SVC["Kubeflow Pipelines<br/>(Pipeline orchestration)"]
        end

        subgraph Namespace: docs-agent
            API_POD["FastAPI Server Pod<br/>(Deployment, 2 replicas)"]
            MCP_POD["MCP Server Pod<br/>(Deployment, 1 replica)"]
            AGENT_CRD["kagent Agent<br/>(Custom Resource)"]
        end

        subgraph Namespace: milvus
            MV_POD["Milvus Standalone<br/>(StatefulSet)"]
            MINIO_POD["MinIO<br/>(StatefulSet)"]
            ETCD_POD["etcd<br/>(StatefulSet)"]
        end

        subgraph Namespace: kserve
            LLM_IS["InferenceService: Llama<br/>(scale-to-zero)"]
            EMB_IS["InferenceService: Embeddings<br/>(always-on)"]
        end

        subgraph Namespace: istio-system
            GW["Istio Gateway"]
            VS["VirtualService"]
        end
    end

    INTERNET["Internet / User"] -->|"HTTPS"| GW
    GW --> VS
    VS -->|"/api/*"| API_POD
    API_POD --> AGENT_CRD
    AGENT_CRD --> MCP_POD
    MCP_POD --> MV_POD
    AGENT_CRD --> LLM_IS
    KFP_SVC --> EMB_IS
    KFP_SVC --> MV_POD
    MV_POD --> MINIO_POD
    MV_POD --> ETCD_POD

    style GW fill:#4A90D9,color:#fff
    style API_POD fill:#2ECC71,color:#fff
    style MCP_POD fill:#7B68EE,color:#fff
    style MV_POD fill:#2ECC71,color:#fff
    style LLM_IS fill:#E8A838,color:#fff
    style EMB_IS fill:#E8A838,color:#fff
    style AGENT_CRD fill:#4A90D9,color:#fff
```
