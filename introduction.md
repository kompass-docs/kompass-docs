
## 1. Product Overview, Vision, and Market Positioning

### 1.1 Product Vision

AI Studio is an enterprise platform that enables organizations to build, deploy, and operate AI agent systems at scale. The platform provides unified capabilities for:

- **Multi-Agent Orchestration**: Coordinate complex workflows involving multiple AI agents using sequential, parallel, hierarchical, graph, and conditional execution patterns
- **Prompt Lifecycle Management**: Version, promote, and render prompt templates with enterprise-grade governance and caching
- **LLM Operations**: Standardized gateway for LLM calls, embeddings, and cost tracking with structured observability
- **Component-Based Execution**: Graph-based execution of reusable components (API, LLM, agent, guardrail) for rapid application development

The platform is designed to abstract away the complexity of integrating multiple AI frameworks (LangChain, CrewAI, OpenAI, Ollama), managing prompt versions, orchestrating multi-agent workflows, and tracking costs and performance across AI operations.

### 1.2 Market Positioning

AI Studio positions itself as an **enterprise AI orchestration platform** that bridges the gap between:

- **AI Framework Fragmentation**: Organizations struggle to standardize on a single AI framework while needing to leverage best-in-class tools. AI Studio provides a unified abstraction layer over LangChain, CrewAI, OpenAI, Ollama, and custom frameworks.
- **Prompt Management Complexity**: Teams need version control, A/B testing, and production promotion for prompts, but existing solutions are either too lightweight or too complex. AI Studio provides enterprise-grade prompt versioning with Redis caching and MongoDB persistence.
- **Multi-Agent Coordination**: Building systems that require multiple agents to collaborate requires custom orchestration logic. AI Studio provides declarative workflow patterns (sequential, parallel, hierarchical, graph, conditional) that reduce boilerplate and increase reliability.
- **Observability Gaps**: AI operations lack standardized tracing, cost tracking, and performance monitoring. AI Studio provides structured trace ingestion (ClickHouse), session aggregation, and metadata-driven cost analysis.

**Target Market Segments:**
- **Enterprise Software Companies**: Building AI-powered features into existing products
- **AI/ML Platform Teams**: Providing internal AI infrastructure to product engineering teams
- **Consulting and Professional Services**: Delivering custom AI solutions to clients
- **Financial Services, Healthcare, Legal**: Regulated industries requiring audit trails, version control, and governance

### 1.3 Platform Architecture Overview

AI Studio is architected as a **distributed microservices platform** with three primary runtime services:

1. **Workflow Engine (DotAgent)**: FastAPI service providing agent/workflow CRUD, execution orchestration, component/flow execution, global variables, guardrails, MCP tool discovery, and WebSocket streaming
2. **Prompt Engine**: FastAPI service providing prompt CRUD, versioning, production promotion, template rendering, trace ingestion/querying, and upstream LLM/projects proxying
3. **LLM Engine**: FastAPI service providing LLM chat/text completions, embeddings, LLM judge scoring, LiteLLM registry proxy, and structured trace emission

**Shared Infrastructure:**
- **PostgreSQL/SQLite**: Primary persistence for agents, workflows, executions, components, flows, global variables, guardrails, and migration tracking
- **MongoDB**: Document store for prompts (with embedded versions), tags, model configs, and system prompts
- **Redis**: Production prompt version caching and session state
- **ClickHouse**: Analytical store for traces, cost aggregation, and migration tracking

### 1.4 Key Differentiators

1. **Framework Agnostic**: Single API surface for LangChain, CrewAI, OpenAI, Ollama, and custom frameworks
2. **Declarative Workflows**: Define multi-agent workflows as configuration (YAML or API), not code
3. **Enterprise Prompt Governance**: Version control, production promotion, and Redis caching for high-throughput prompt retrieval
4. **Unified Observability**: Structured traces in ClickHouse with session aggregation and cost tracking
5. **Component Reusability**: Build once, execute in multiple flows via component registry
6. **Streaming Execution**: WebSocket and SSE support for real-time execution status and "thinking logs"

### 1.5 Platform Boundaries and Integration Points

**What AI Studio Provides:**
- Agent/workflow definition and execution
- Prompt template storage, versioning, and rendering
- LLM gateway with tracing and cost estimation
- Component/flow execution engine
- Global variable management with credential encryption
- Guardrail execution
- MCP tool discovery
- Memory/embedding integration
- Trace ingestion and querying

**What AI Studio Does Not Provide (Out of Scope):**
- **Authentication/Authorization**: No inbound auth middleware; must be enforced via API gateway or reverse proxy
- **Rate Limiting/Quotas**: No built-in rate limiting; must be enforced externally
- **Multi-Tenant Isolation**: `project_id` is used for filtering/scoping, not security boundaries
- **First-Party SDKs**: Only HTTP APIs are provided (no official client libraries)
- **Guaranteed Trace Delivery**: Trace ingestion is async/best-effort; failures are logged but not surfaced to callers
- **CI/CD Pipelines**: No built-in deployment automation or versioning policy beyond route prefixes (`/api/v1/...`)

**External Dependencies:**
- Upstream LiteLLM service (for LLM calls, embeddings, registry operations)
- Upstream Projects service (optional, for project metadata)
- PostgreSQL (or SQLite for development)
- MongoDB (for prompts, tags, model configs)
- Redis (for prompt caching)
- ClickHouse (for traces and migrations)

---
