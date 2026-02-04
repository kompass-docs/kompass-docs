## 4. Functional Requirements and End-to-End User Workflows

### 4.1 Functional Requirements Overview

AI Studio's functional requirements are organized into six primary capability areas:

1. **Agent Management**: Create, update, delete, and execute AI agents with framework-agnostic configuration
2. **Workflow Orchestration**: Define and execute multi-agent workflows using sequential, parallel, hierarchical, graph, and conditional patterns
3. **Prompt Lifecycle Management**: Version, promote, and render prompt templates with enterprise governance
4. **Component and Flow Execution**: Build reusable components and execute graph-based flows
5. **LLM Operations**: Standardized gateway for LLM calls, embeddings, and cost tracking
6. **Observability and Tracing**: Structured trace ingestion, querying, and cost aggregation

Each capability area includes detailed functional requirements, API contracts, data models, validation rules, and end-to-end workflows.

---

### 4.2 Agent Management (Part 1)

#### 4.2.1 Functional Requirements

**FR-AGENT-1: Agent Creation**
- **Requirement**: Users must be able to create AI agents with framework-specific configurations
- **API**: `POST /api/agents`
- **Input Schema**:
  - `name` (required, string): Unique agent identifier
  - `framework` (required, enum): One of `['langchain', 'autogen', 'crewai', 'openai', 'ollama', 'custom']`
  - `system_prompt` (required, string): System prompt for the agent
  - `description` (optional, string): Human-readable description
  - `llm_config` (required, object): LLM configuration with model, temperature, maxTokens, etc.
  - `capabilities` (optional, array): List of capability strings (e.g., `["conversation"]`)
  - `tools` (optional, array): List of tool names (must be unique)
  - `tags` (optional, array): Tags for categorization
  - `memory` (optional, object): Memory configuration
  - `framework_config` (optional, object): Framework-specific configuration
  - `max_iterations` (optional, integer): Maximum agent iterations
  - `timeout` (optional, integer): Execution timeout in seconds
  - `status` (optional, enum): `"active"` or `"inactive"` (default: `"active"`)
  - `owner` (optional, string): Owner identifier (default: `"system"`)
  - `category` (optional, string): Category for organization (default: `"general"`)

- **Behavior**:
  - Server injects LLM defaults from environment variables (`LITELLM_PROVIDER`, `LITELLM_BASE_URL`, `LITELLM_KEY`) into `llm_config` if not provided
  - Validates framework against allowlist
  - Ensures tool names are unique
  - Persists agent to `aistudio_agents` table in PostgreSQL
  - Returns agent object with generated `agent_id`

- **Error Handling**:
  - `400`: Invalid framework, duplicate tool names, missing required fields
  - `500`: Database errors, unexpected exceptions

**FR-AGENT-2: Agent Retrieval**
- **Requirement**: Users must be able to retrieve agent definitions by ID
- **API**: `GET /api/agents/{agent_id}`
- **Response**: Agent object with all configuration fields
- **Error Handling**:
  - `404`: Agent not found

**FR-AGENT-3: Agent Listing**
- **Requirement**: Users must be able to list all agents with optional filtering
- **API**: `GET /api/agents`
- **Query Parameters**:
  - `framework` (optional): Filter by framework
  - `status` (optional): Filter by status
  - `category` (optional): Filter by category
  - `tags` (optional): Filter by tags
- **Response**: Array of agent objects

**FR-AGENT-4: Agent Update**
- **Requirement**: Users must be able to update existing agent configurations
- **API**: `PUT /api/agents/{agent_id}`
- **Input Schema**: Same as agent creation (all fields optional except those that cannot be changed)
- **Behavior**:
  - Updates agent in database
  - Preserves agent ID and creation timestamp
  - Updates `updated_at` timestamp
- **Error Handling**:
  - `404`: Agent not found
  - `400`: Invalid configuration

**FR-AGENT-5: Agent Deletion**
- **Requirement**: Users must be able to delete agents
- **API**: `DELETE /api/agents/{agent_id}`
- **Behavior**:
  - Deletes agent from database
  - Note: Does not validate if agent is referenced by workflows (orphaned references may exist)
- **Error Handling**:
  - `404`: Agent not found

**FR-AGENT-6: Agent Execution**
- **Requirement**: Users must be able to execute agents with input context
- **API**: `POST /api/agents/{agent_id}/execute`
- **Input Schema**:
  - `input` (required, string or object): Input context for the agent
  - `stream` (optional, boolean): Enable streaming response (default: `false`)
- **Response**:
  - `output` (string): Agent response
  - `metadata` (object): Execution metadata (latency, tokens, etc.)
- **Behavior**:
  - Loads agent configuration from database
  - Creates runtime agent via adapter registry (`AgentRegistry.create_agent`)
  - Executes agent with input context
  - Returns response and metadata
- **Error Handling**:
  - `404`: Agent not found
  - `500`: Execution errors, adapter errors

**FR-AGENT-7: Agent Execution with Streaming Logs**
- **Requirement**: Users must be able to execute agents with real-time "thinking logs"
- **API**: `POST /api/agents/{agent_id}/execute-with-logs`
- **Response**: Server-Sent Events (SSE) stream with events:
  - `event: thinking`: Agent thinking/intermediate steps
  - `event: final_answer`: Final agent response
  - `event: error`: Execution errors
- **Behavior**:
  - Uses framework-specific callbacks (LangChain, CrewAI) to capture thinking logs
  - Streams events as they occur during execution
- **Error Handling**:
  - `404`: Agent not found
  - `500`: Execution errors

**FR-AGENT-8: Agent Configuration Retrieval**
- **Requirement**: Users must be able to retrieve agent configuration schema
- **API**: `GET /api/agents/config`
- **Response**: Configuration schema/metadata for agent creation

**FR-AGENT-9: Agent Tools Listing**
- **Requirement**: Users must be able to list available tools for agents
- **API**: `GET /api/agents/tools`
- **Response**: List of available tool definitions
- **Behavior**:
  - Returns tools from `dotagent/adapters/tools/` (LangChain tools, CrewAI tools)
  - May include MCP-discovered tools if MCP service is configured

#### 4.2.2 End-to-End Workflow: Creating and Executing an Agent

**Workflow Steps:**

1. **Create Agent**
   ```
   POST /api/agents
   {
     "name": "research-assistant",
     "framework": "langchain",
     "system_prompt": "You are a research assistant that provides concise, accurate information.",
     "llm_config": {
       "model": "gpt-4",
       "temperature": 0.7
     },
     "capabilities": ["conversation", "research"],
     "tools": ["web_search", "document_reader"],
     "status": "active"
   }
   ```
   **Response**: `201 Created` with agent object including `agent_id`

2. **Verify Agent Creation**
   ```
   GET /api/agents/{agent_id}
   ```
   **Response**: Agent object with all fields

3. **Execute Agent**
   ```
   POST /api/agents/{agent_id}/execute
   {
     "input": "What are the latest developments in quantum computing?"
   }
   ```
   **Response**: 
   ```json
   {
     "output": "Recent developments in quantum computing include...",
     "metadata": {
       "latency_ms": 1250,
       "tokens_used": 450
     }
   }
   ```

4. **Execute with Streaming Logs (Optional)**
   ```
   POST /api/agents/{agent_id}/execute-with-logs
   {
     "input": "Analyze this document and summarize key points."
   }
   ```
   **Response**: SSE stream with thinking logs and final answer

5. **Update Agent (Optional)**
   ```
   PUT /api/agents/{agent_id}
   {
     "system_prompt": "You are an expert research assistant...",
     "llm_config": {
       "model": "gpt-4-turbo",
       "temperature": 0.5
     }
   }
   ```
   **Response**: Updated agent object

6. **Delete Agent (Optional)**
   ```
   DELETE /api/agents/{agent_id}
   ```
   **Response**: `204 No Content`

**Workflow Duration**: < 2 minutes for complete workflow

**Success Criteria**:
- Agent created and persisted successfully
- Agent execution returns valid response
- Streaming logs capture intermediate steps (if enabled)
- Agent update preserves existing configuration where not overridden

---

### 4.3 Workflow Orchestration (Part 2)

#### 4.3.1 Functional Requirements

**FR-WORKFLOW-1: Workflow Creation**
- **Requirement**: Users must be able to create multi-agent workflows with declarative configuration
- **API**: `POST /api/workflows`
- **Input Schema**:
  - `name` (required, string): Unique workflow identifier
  - `description` (optional, string): Human-readable description
  - `pattern` (required, enum): One of `['sequential', 'parallel', 'hierarchical', 'graph', 'conditional']`
  - `agents` (required, array): List of agent IDs referenced in steps
  - `steps` (required, array): List of workflow steps (minimum 1 step required)
    - `agent` (required, string): Agent ID (must exist in `agents` array)
    - `task` (required, string): Task description/prompt for the agent
    - `depends_on` (optional, array): List of step indices this step depends on (for graph patterns)
    - `condition` (optional, string): Conditional expression (for conditional patterns)
  - `visual_data` (optional, object): UI visualization metadata
  - `timeout` (optional, integer): Workflow execution timeout in seconds
  - `max_retries` (optional, integer): Maximum retry attempts (default: 0)
  - `error_handling` (optional, enum): `'stop'`, `'continue'`, or `'retry'` (default: `'stop'`)

- **Validation Rules**:
  - At least one step required
  - Each step's `agent` must exist in `agents` array
  - For graph patterns: `depends_on` must form a valid DAG (no circular dependencies)
  - For parallel patterns: Requires ≥ 3 steps (first, ≥1 middle, last)
  - For hierarchical patterns: First step is the manager

- **Behavior**:
  - Persists workflow to `aistudio_workflows` table in PostgreSQL
  - Validates agent references exist in database
  - Returns workflow object with generated `workflow_id`

- **Error Handling**:
  - `400`: Invalid pattern, missing steps, invalid agent references, circular dependencies
  - `500`: Database errors

**FR-WORKFLOW-2: Workflow Retrieval**
- **Requirement**: Users must be able to retrieve workflow definitions by ID
- **API**: `GET /api/workflows/{workflow_id}`
- **Response**: Workflow object with all configuration fields
- **Error Handling**:
  - `404`: Workflow not found

**FR-WORKFLOW-3: Workflow Listing**
- **Requirement**: Users must be able to list all workflows
- **API**: `GET /api/workflows`
- **Response**: Array of workflow objects

**FR-WORKFLOW-4: Workflow Update**
- **Requirement**: Users must be able to update existing workflow configurations
- **API**: `PUT /api/workflows/{workflow_id}`
- **Input Schema**: Same as workflow creation (all fields optional)
- **Behavior**:
  - Updates workflow in database
  - Preserves workflow ID and creation timestamp
  - Updates `updated_at` timestamp
- **Error Handling**:
  - `404`: Workflow not found
  - `400`: Invalid configuration

**FR-WORKFLOW-5: Workflow Deletion**
- **Requirement**: Users must be able to delete workflows
- **API**: `DELETE /api/workflows/{workflow_id}`
- **Behavior**:
  - Deletes workflow from database
  - Note: Does not delete associated executions (execution history is preserved)
- **Error Handling**:
  - `404`: Workflow not found

**FR-WORKFLOW-6: Workflow Execution (Async)**
- **Requirement**: Users must be able to execute workflows asynchronously with execution tracking
- **API**: `POST /api/executions`
- **Input Schema**:
  - `workflow_id` (required, string): Workflow ID to execute
  - `context` (required, object): Initial context/input for the workflow
  - `stream` (optional, boolean): Enable WebSocket streaming (default: `false`)

- **Behavior**:
  - Creates execution record in `aistudio_executions` table with status `"pending"`
  - Executes workflow in background thread pool (`ThreadPoolExecutor`)
  - Updates execution status to `"running"` when execution starts
  - Executes workflow via `Orchestrator.execute_workflow()`
  - Persists execution results, logs, and errors
  - Updates execution status to `"completed"` or `"failed"`
  - If `stream=true`, publishes status updates via WebSocket (`/ws/execution/{execution_id}`)

- **Response**:
  - `execution_id` (string): Execution identifier
  - `status` (string): Initial status (`"pending"`)

- **Error Handling**:
  - `404`: Workflow not found
  - `400`: Invalid context, missing required fields
  - `500`: Execution errors

**FR-WORKFLOW-7: Execution Status Retrieval**
- **Requirement**: Users must be able to retrieve execution status and results
- **API**: `GET /api/executions/{execution_id}`
- **Response**:
  - `execution_id` (string)
  - `workflow_id` (string)
  - `status` (enum): `"pending"`, `"running"`, `"completed"`, `"failed"`, `"cancelled"`
  - `context` (object): Initial context
  - `results` (object): Execution results (if completed)
  - `logs` (array): Execution logs
  - `errors` (array): Error messages (if failed)
  - `created_at` (datetime): Execution start time
  - `updated_at` (datetime): Last update time

**FR-WORKFLOW-8: Execution Listing**
- **Requirement**: Users must be able to list executions with optional filtering
- **API**: `GET /api/executions`
- **Query Parameters**:
  - `workflow_id` (optional): Filter by workflow ID
  - `status` (optional): Filter by status
  - `limit` (optional): Pagination limit
  - `offset` (optional): Pagination offset
- **Response**: Array of execution objects

**FR-WORKFLOW-9: Execution Cancellation**
- **Requirement**: Users must be able to cancel running executions
- **API**: `POST /api/executions/{execution_id}/cancel`
- **Behavior**:
  - Updates execution status to `"cancelled"`
  - Attempts to stop workflow execution (best-effort)
- **Error Handling**:
  - `404`: Execution not found
  - `400`: Execution not in cancellable state

**FR-WORKFLOW-10: Execution Deletion**
- **Requirement**: Users must be able to delete execution records
- **API**: `DELETE /api/executions/{execution_id}`
- **Behavior**:
  - Deletes execution record from database
- **Error Handling**:
  - `404`: Execution not found

**FR-WORKFLOW-11: WebSocket Execution Streaming**
- **Requirement**: Users must be able to receive real-time execution updates via WebSocket
- **API**: `WS /ws/execution/{execution_id}`
- **Message Format**: JSON objects with:
  - `type` (string): Event type (`"status_change"`, `"log"`, `"error"`, `"result"`)
  - `execution_id` (string): Execution identifier
  - `data` (object): Event-specific data
- **Behavior**:
  - Server publishes updates as execution progresses
  - Client receives status changes, logs, errors, and final results
  - Connection closes when execution completes or fails

**FR-WORKFLOW-12: SSE Execution Streaming (Alternative)**
- **Requirement**: Users must be able to receive execution updates via Server-Sent Events
- **API**: `POST /api/executions/sse`
- **Behavior**:
  - Creates execution and streams events via SSE
  - Events emitted via API Gateway SSE channel (if configured)
- **Note**: Implementation may delegate to external API Gateway for SSE delivery

#### 4.3.2 Workflow Patterns

**Pattern 1: Sequential**
- **Description**: Execute steps one after another in order
- **Use Case**: Linear workflows where each step depends on the previous step's output
- **Execution**: Steps execute sequentially, passing context from step to step
- **Example**: Research → Analysis → Report generation

**Pattern 2: Parallel**
- **Description**: First agent selects which middle agents to execute, middle agents execute concurrently, last agent aggregates results
- **Requirements**: ≥ 3 steps (first, ≥1 middle, last)
- **Execution Contract**:
  - First agent's system prompt is enhanced to require JSON response:
    - `"agent_to_be_executed": ["agent_id1", ...]` (list of middle agent IDs to execute)
    - `"output": "..."` (optional output from first agent)
    - `"{agent_id}_content": "..."` (optional content per selected middle agent)
  - Middle agents execute concurrently (asyncio tasks)
  - Last agent receives aggregated results from middle agents
- **Failure Mode**: If first agent does not return valid JSON matching expected structure, parsing/routing fails
- **Use Case**: Dynamic agent selection with parallel execution and aggregation

**Pattern 3: Hierarchical**
- **Description**: Manager agent selects worker agents to execute based on task
- **Requirements**: First step is the manager
- **Execution Contract**:
  - Manager agent's system prompt is enhanced to require JSON response:
    - `"agent_to_be_executed": ["agent_id1", ...]` (list of worker agent IDs to execute)
    - `"context": "..."` (context passed to workers)
    - `"output": "..."` (optional output from manager)
  - Selected worker steps execute (parallelized via asyncio tasks)
  - Manager receives aggregated worker results
- **Use Case**: Manager-worker pattern for task delegation

**Pattern 4: Graph**
- **Description**: Steps execute based on dependency graph (DAG)
- **Requirements**: 
  - Steps define `depends_on` array with step indices
  - Must form a valid DAG (no circular dependencies)
- **Execution**: 
  - Steps with satisfied dependencies execute (may execute in parallel)
  - Dependency validation ensures no cycles
- **Error Handling**: Raises `"Circular dependency detected in workflow graph"` if cycles found
- **Use Case**: Complex workflows with conditional dependencies

**Pattern 5: Conditional**
- **Description**: Steps execute based on conditional expressions
- **Requirements**: Steps define `condition` expressions
- **Execution**: 
  - Conditions evaluated at runtime
  - Steps execute only if conditions evaluate to true
- **Use Case**: Branching workflows based on runtime conditions

#### 4.3.3 Error Handling in Workflows

**Error Handling Modes:**
- **`stop`** (default): Workflow stops on first error, execution status set to `"failed"`
- **`continue`**: Workflow continues to next step, errors logged but execution continues
- **`retry`**: Failed step retried up to `max_retries` times before continuing or stopping

**Error Propagation:**
- Errors captured in execution `errors` array
- Errors logged to execution `logs` array
- WebSocket/SSE streams error events to clients
- Execution status updated to `"failed"` if workflow cannot continue

#### 4.3.4 End-to-End Workflow: Sequential Research Workflow

**Workflow Steps:**

1. **Create Agents**
   ```
   POST /api/agents (create researcher agent)
   POST /api/agents (create analyzer agent)
   POST /api/agents (create writer agent)
   ```

2. **Create Sequential Workflow**
   ```
   POST /api/workflows
   {
     "name": "research-analysis-write",
     "description": "Research topic, analyze findings, write report",
     "pattern": "sequential",
     "agents": ["researcher_id", "analyzer_id", "writer_id"],
     "steps": [
       {
         "agent": "researcher_id",
         "task": "Research the topic: {{topic}}",
         "depends_on": []
       },
       {
         "agent": "analyzer_id",
         "task": "Analyze the research findings and identify key insights",
         "depends_on": [0]
       },
       {
         "agent": "writer_id",
         "task": "Write a comprehensive report based on the analysis",
         "depends_on": [1]
       }
     ],
     "error_handling": "stop",
     "timeout": 300
   }
   ```
   **Response**: `201 Created` with `workflow_id`

3. **Execute Workflow**
   ```
   POST /api/executions
   {
     "workflow_id": "{workflow_id}",
     "context": {
       "topic": "Quantum computing applications in cryptography"
     },
     "stream": true
   }
   ```
   **Response**: 
   ```json
   {
     "execution_id": "{execution_id}",
     "status": "pending"
   }
   ```

4. **Monitor Execution via WebSocket**
   ```
   WS /ws/execution/{execution_id}
   ```
   **Messages Received**:
   - `{"type": "status_change", "data": {"status": "running"}}`
   - `{"type": "log", "data": {"step": 0, "message": "Researching topic..."}}`
   - `{"type": "log", "data": {"step": 1, "message": "Analyzing findings..."}}`
   - `{"type": "log", "data": {"step": 2, "message": "Writing report..."}}`
   - `{"type": "status_change", "data": {"status": "completed"}}`
   - `{"type": "result", "data": {"output": "Comprehensive report..."}}`

5. **Retrieve Final Results**
   ```
   GET /api/executions/{execution_id}
   ```
   **Response**:
   ```json
   {
     "execution_id": "{execution_id}",
     "workflow_id": "{workflow_id}",
     "status": "completed",
     "results": {
       "step_0": "Research findings...",
       "step_1": "Key insights...",
       "step_2": "Comprehensive report..."
     },
     "logs": [...],
     "created_at": "2024-01-15T10:00:00Z",
     "updated_at": "2024-01-15T10:05:30Z"
   }
   ```

**Workflow Duration**: ~5 minutes for complete workflow execution

**Success Criteria**:
- Workflow created and persisted successfully
- Execution starts and progresses through all steps
- WebSocket streams provide real-time updates
- Final results contain outputs from all steps
- Execution status accurately reflects completion

---

### 4.4 Prompt Lifecycle Management (Part 3)

#### 4.4.1 Functional Requirements

**FR-PROMPT-1: Prompt Creation**
- **Requirement**: Users must be able to create prompt templates with versioning support
- **API**: `POST /api/v1/prompts`
- **Input Schema**:
  - `name` (required, string): Unique prompt identifier (case-sensitive, must be unique)
  - `promptType` (required, enum): `"text"` (single string template) or `"chat"` (array of `{role, content}` messages)
  - `prompt` (required, string or array): 
    - For `text`: Single string template with Jinja2 variables (e.g., `"Write an email about {{topic}}"`
    - For `chat`: Array of message objects `[{role: "user", content: "..."}, ...]`
  - `labels` (optional, array of strings): Labels for version (default: `["latest", "production"]` if not provided)
  - `config` (optional, object): Model configuration `{modelName, temperature, maxTokens}` or `null`
  - `tags` (optional, array of strings): Tags for categorization
  - `project_id` (optional, array of strings): Project identifiers (default: `["default"]` if not provided)
  - `description` (optional, string): Human-readable description
  - `variables_metadata` (optional, array): Variable metadata `[{variable: "topic", metadescription: "..."}]`

- **Behavior**:
  - Stores prompt in MongoDB collection (`aistudio_prompt_management` by default)
  - Creates version 1 with `versionNumber: 1`
  - Generates unique `versionId` (ObjectId) for the version
  - Normalizes `project_id` to list format, ensures `"default"` is present
  - Extracts template variables from Jinja2 AST if `variables_metadata` not provided
  - Rejects unknown placeholder descriptions if `variables_metadata` provided
  - Ensures tags exist in tags collection (creates if missing)
  - Creates unique index on `name` field
  - Creates non-unique index on `updatedAt` field

- **Response**: Prompt object with:
  - `_id` (ObjectId): Prompt document ID
  - `name` (string)
  - `promptType` (enum)
  - `versions` (array): Array containing version 1
  - `createdAt` (datetime)
  - `updatedAt` (datetime)
  - `tags` (array)
  - `project_id` (array)

- **Error Handling**:
  - `400`: Invalid `promptType`, empty prompt content, unknown placeholder descriptions
  - `409`: Prompt name already exists (unique constraint violation)
  - `500`: Database errors

**FR-PROMPT-2: Prompt Retrieval by ID**
- **Requirement**: Users must be able to retrieve prompt documents by ObjectId
- **API**: `GET /api/v1/prompts/{prompt_identifier}`
- **Input**: `prompt_identifier` must be a valid 24-character MongoDB ObjectId
- **Response**: Full prompt document with all versions
- **Query Parameters**:
  - `project_id` (optional): Filter by project scope (filtering only, not security boundary)
- **Error Handling**:
  - `400`: Invalid ObjectId format
  - `404`: Prompt not found

**FR-PROMPT-3: Prompt Listing with Filtering**
- **Requirement**: Users must be able to list prompts with advanced filtering
- **API**: `GET /api/v1/prompts`
- **Query Parameters**:
  - `q` (optional, string): Case-insensitive regex search on prompt name
  - `promptType` (optional, enum): Filter by `"text"` or `"chat"`
  - `label` (optional, string): Filter by label (applied to **latest** version labels)
  - `skip` (optional, integer): Pagination offset (default: 0)
  - `limit` (optional, integer): Pagination limit (1-100, default: 10)
  - `project_id` (optional, string): Filter by project (matches if project_id is in prompt's project_id list)
  - `tag` (optional, string): Exact match in prompt-level tags
- **Response**: Object with:
  - `items` (array): Array of prompt documents
  - `skip` (integer): Pagination offset
  - `limit` (integer): Pagination limit
- **Error Handling**:
  - `400`: Invalid limit (must be 1-100)

**FR-PROMPT-4: Prompt Update (Version Creation)**
- **Requirement**: Users must be able to update prompts, which creates a new version when certain fields change
- **API**: `PATCH /api/v1/prompts/{prompt_id}`
- **Input Schema** (all fields optional):
  - `prompt` (optional): New prompt content (creates new version if changed)
  - `config` (optional): New model configuration (creates new version if changed)
  - `labels` (optional): New labels (creates new version if changed)
  - `description` (optional): New description (creates new version if changed)
  - `tags` (optional): New tags (creates new version if changed)
  - `variables_metadata` (optional): New variable metadata (creates new version if changed)
  - `project_id` (optional): New project IDs (updates prompt-level field, does not create version)

- **Behavior**:
  - **Immutability Constraint**: `promptType` **cannot be changed** once created (validation error if attempted)
  - **Version Creation Logic**: New version is created if any of these fields change: `prompt`, `config`, `labels`, `description`, `tags`, `variables_metadata`
  - **Label Management**:
    - Removes `"latest"` label from previous latest version
    - If new `labels` include `"production"`, removes `"production"` label from all other versions
    - Adds new version with incremented `versionNumber` (e.g., if latest is version 3, new version is 4)
  - **Version Metadata**:
    - New version gets unique `versionId` (ObjectId)
    - New version gets `createdAt` timestamp
    - New version inherits `labels` from request (or defaults)
  - **Prompt-Level Updates**:
    - Updates `tags` at prompt level (not version-specific)
    - Updates `project_id` at prompt level
    - Updates `updatedAt` timestamp

- **Response**: Updated prompt object with new version appended to `versions` array

- **Error Handling**:
  - `400`: Attempt to change `promptType`, invalid configuration
  - `404`: Prompt not found
  - `500`: Database errors

**FR-PROMPT-5: Version Listing**
- **Requirement**: Users must be able to list all versions of a prompt
- **API**: `GET /api/v1/prompts/{prompt_id}/versions`
- **Response**: Array of version objects, sorted by `versionNumber` descending (newest first)
- **Version Object Structure**:
  - `versionId` (ObjectId)
  - `versionNumber` (integer)
  - `prompt` (string or array)
  - `config` (object or null)
  - `labels` (array of strings)
  - `createdAt` (datetime)
  - `description` (string or null)
  - `variables_metadata` (array or null)
- **Error Handling**:
  - `404`: Prompt not found

**FR-PROMPT-6: Production Version Retrieval (Cached)**
- **Requirement**: Users must be able to retrieve the production version of a prompt with Redis caching
- **API**: `GET /api/v1/prompts/{prompt_id}/production`
- **Query Parameters**:
  - `project_id` (optional): Project scope for cache key (default: `"default"`)
- **Behavior**:
  - **Cache Lookup**: Checks Redis for key `prompt:production:{prompt_id}:project_id:{scope}`
    - `scope` = `project_id` normalized to `"default"` if missing/blank
  - **Cache Hit**: Returns cached version object immediately
  - **Cache Miss**:
    - Fetches prompt from MongoDB
    - Selects production version (highest `versionNumber` with `"production"` in `labels`)
    - Caches version object in Redis with TTL (default: 3600 seconds, configurable via `REDIS_TTL`)
    - Returns version object
- **Response**: Single version object (not full prompt document)
- **Error Handling**:
  - `404`: Prompt not found, or no production version found
  - `500`: Database or cache errors

**FR-PROMPT-7: Production Version Promotion**
- **Requirement**: Users must be able to promote a specific version to production
- **API**: `PATCH /api/v1/prompts/{prompt_id}/versions/{version_id}/production`
- **Behavior**:
  - Removes `"production"` label from all versions in the prompt
  - Adds `"production"` label to the target version
  - Invalidates Redis cache for production version (all project scopes)
  - Updates `updatedAt` timestamp
- **Response**: Updated prompt object with production label moved
- **Error Handling**:
  - `404`: Prompt or version not found
  - `500`: Database errors

**FR-PROMPT-8: Version Deletion**
- **Requirement**: Users must be able to delete specific versions (with restrictions)
- **API**: `DELETE /api/v1/prompts/{prompt_id}/versions/{version_id}`
- **Behavior**:
  - **Restriction**: Blocks deletion of versions with `"production"` label (returns error)
  - Removes version from `versions` array in MongoDB document
  - Updates `updatedAt` timestamp
- **Response**: `204 No Content`
- **Error Handling**:
  - `400`: Attempt to delete production version
  - `404`: Prompt or version not found
  - `500`: Database errors

**FR-PROMPT-9: Prompt Deletion**
- **Requirement**: Users must be able to delete entire prompt documents
- **API**: `DELETE /api/v1/prompts/{prompt_id}`
- **Behavior**:
  - Deletes prompt document from MongoDB
  - Invalidates all Redis caches for the prompt (all project scopes)
- **Response**: `204 No Content`
- **Error Handling**:
  - `404`: Prompt not found
  - `500`: Database errors

**FR-PROMPT-10: Template Rendering**
- **Requirement**: Users must be able to render prompt templates with runtime variables
- **API**: `POST /api/v1/prompts/variables/{prompt_name}`
- **Input Schema**:
  - `variables` (required, object): Key-value pairs for template variables (e.g., `{"topic": "Product Launch"}`)
- **Behavior**:
  - **Prompt Lookup**: Finds prompt by `name` (not ID) in MongoDB
  - **Version Selection**: Selects **production** version (highest `versionNumber` with `"production"` in `labels`)
  - **Variable Validation**: 
    - Extracts required variables from template (via Jinja2 AST)
    - Validates all required variables are provided
    - Uses Jinja2 `StrictUndefined` environment (undefined variables raise errors)
  - **Rendering**:
    - For `text` prompts: Renders single string template
    - For `chat` prompts: Renders each message's `content` field as template
  - **Error Handling**:
    - `404`: Prompt not found, or no production version found
    - `422`: Missing values for variables (exact message: `"Missing values for variables: <var1>, <var2>"`)
    - `400`: Template syntax error (exact message: `"Template syntax error: <error>"`)
    - `500`: Unexpected rendering errors
- **Response**:
  - For `text` prompts: `{"rendered_prompt": "..."}`
  - For `chat` prompts: `{"rendered_prompt": [{role: "user", content: "..."}, ...]}`
- **Error Handling**:
  - `404`: Prompt not found, no production version
  - `422`: Missing variables
  - `400`: Template syntax error
  - `500`: Rendering errors

**FR-PROMPT-11: Tag Management**
- **Requirement**: Users must be able to create and list tags
- **APIs**:
  - `POST /api/v1/tags`: Create tag (ensures uniqueness in tags collection)
  - `GET /api/v1/tags`: List all tags (returns sorted list of tag names)
- **Behavior**:
  - Tags stored in separate MongoDB collection (`aiprompt_tags` by default)
  - Unique index on `name` field
  - Prompt creation/update ensures tags exist (creates if missing)

#### 4.4.2 Data Model: Prompt Document Structure

**MongoDB Document Schema:**
```json
{
  "_id": ObjectId("..."),
  "name": "email-writer",
  "promptType": "text",
  "createdAt": ISODate("2024-01-15T10:00:00Z"),
  "updatedAt": ISODate("2024-01-15T10:30:00Z"),
  "tags": ["email", "communication"],
  "project_id": ["default", "project-123"],
  "versions": [
    {
      "versionId": ObjectId("..."),
      "versionNumber": 1,
      "prompt": "Write an email about {{topic}}",
      "config": {
        "modelName": "gpt-4",
        "temperature": 0.7,
        "maxTokens": 500
      },
      "labels": ["latest", "production"],
      "createdAt": ISODate("2024-01-15T10:00:00Z"),
      "description": "Initial version",
      "variables_metadata": [
        {
          "variable": "topic",
          "metadescription": "The topic of the email"
        }
      ]
    },
    {
      "versionId": ObjectId("..."),
      "versionNumber": 2,
      "prompt": "Write a professional email about {{topic}} with {{tone}}",
      "config": null,
      "labels": ["latest"],
      "createdAt": ISODate("2024-01-15T10:30:00Z"),
      "description": "Added tone variable",
      "variables_metadata": [
        {
          "variable": "topic",
          "metadescription": "The topic of the email"
        },
        {
          "variable": "tone",
          "metadescription": "The tone of the email (formal, casual, friendly)"
        }
      ]
    }
  ]
}
```

**Key Invariants:**
- `promptType` is immutable (cannot be changed after creation)
- Only one version can have `"production"` label at a time
- Only one version can have `"latest"` label at a time
- `project_id` is always a list, and always includes `"default"`
- Versions are append-only (never modified, only new versions added)

#### 4.4.3 Caching Strategy

**Redis Cache Keys:**
- Production version: `prompt:production:{prompt_id}:project_id:{scope}`
  - `scope` = `project_id` normalized to `"default"` if missing/blank

**Cache TTL:**
- Default: 3600 seconds (1 hour)
- Configurable via `REDIS_TTL` environment variable

**Cache Invalidation Triggers:**
- Production version promotion: Invalidates cache for all project scopes
- Prompt deletion: Invalidates all caches for the prompt
- Version deletion: Invalidates production cache if deleted version was production

**Cache Behavior:**
- Only production versions are cached (not latest, not full prompt documents)
- Cache is scoped by `project_id` (different cache keys per project scope)
- Cache misses trigger MongoDB fetch and cache population

#### 4.4.4 End-to-End Workflow: Prompt A/B Testing and Production Promotion

**Workflow Steps:**

1. **Create Initial Prompt (Version 1)**
   ```
   POST /api/v1/prompts
   {
     "name": "customer-support-response",
     "promptType": "text",
     "prompt": "Respond to the customer inquiry: {{inquiry}}",
     "labels": ["latest", "production"],
     "tags": ["support", "customer-service"],
     "description": "Initial customer support prompt"
   }
   ```
   **Response**: `201 Created` with prompt object containing version 1

2. **Test Initial Version**
   ```
   POST /api/v1/prompts/variables/customer-support-response
   {
     "variables": {
       "inquiry": "I need help with my order"
     }
   }
   ```
   **Response**: 
   ```json
   {
     "rendered_prompt": "Respond to the customer inquiry: I need help with my order"
   }
   ```

3. **Create Variant (Version 2)**
   ```
   PATCH /api/v1/prompts/{prompt_id}
   {
     "prompt": "Respond professionally and helpfully to the customer inquiry: {{inquiry}}. Be empathetic and solution-oriented.",
     "labels": ["latest"],
     "description": "Enhanced version with empathy and solution focus"
   }
   ```
   **Response**: Updated prompt with version 2 (version 1 loses `"latest"` label, keeps `"production"`)

4. **Test Both Versions in Production**
   - Application code uses version IDs to test both versions
   - Traces include version metadata for performance analysis

5. **Analyze Performance via Traces**
   ```
   GET /api/v1/traces?limit=100&offset=0
   ```
   **Response**: Trace data with version metadata, latency, cost metrics

6. **Promote Winning Version to Production**
   ```
   PATCH /api/v1/prompts/{prompt_id}/versions/{version_2_id}/production
   ```
   **Response**: Updated prompt with version 2 now labeled `"production"`, version 1 loses `"production"` label

7. **Verify Production Version (Cached)**
   ```
   GET /api/v1/prompts/{prompt_id}/production
   ```
   **Response**: Version 2 object (served from Redis cache if previously accessed)

8. **Render Production Version**
   ```
   POST /api/v1/prompts/variables/customer-support-response
   {
     "variables": {
       "inquiry": "My order is delayed"
     }
   }
   ```
   **Response**: Rendered prompt using version 2 (production version)

**Workflow Duration**: < 1 day for complete A/B test cycle

**Success Criteria**:
- Both versions created and testable
- Production promotion moves labels correctly
- Redis cache serves production version after first access
- Template rendering uses production version
- Version history preserved (immutable versions)

---

### 4.5 Component and Flow Execution (Part 4)

#### 4.5.1 Functional Requirements

**FR-COMPONENT-1: Component Creation**
- **Requirement**: Users must be able to create reusable components with typed configurations
- **API**: `POST /api/components`
- **Input Schema**:
  - `identifier` or `slug` (required, string): Unique component identifier (slug format)
  - `name` (required, string): Human-readable component name
  - `type` (required, enum): Component type (`"api"`, `"llm"`, `"agent"`, `"guardrail"`, or other supported types)
  - `config` (required, object): Type-specific configuration
    - For `"api"`: API endpoint, method, headers, body template
    - For `"llm"`: Model, prompt template, temperature, maxTokens
    - For `"agent"`: Agent ID reference, input mapping
    - For `"guardrail"`: Guardrail configuration, validation rules
  - `description` (optional, string): Component description
  - `tags` (optional, array): Tags for categorization
  - `metadata` (optional, object): Additional metadata

- **Validation Rules**:
  - `identifier`/`slug` must be unique (enforced by database constraint)
  - Component type must be supported by component executor
  - Configuration must be valid for component type

- **Behavior**:
  - Persists component to `aistudio_components` table in PostgreSQL
  - Validates component type and configuration
  - Returns component object with generated `component_id`

- **Error Handling**:
  - `400`: Invalid component type, invalid configuration, missing required fields
  - `409`: Component with slug already exists (exact message: `"Component with slug '{component_slug}' already exists."`)
  - `500`: Database errors

**FR-COMPONENT-2: Component Retrieval**
- **Requirement**: Users must be able to retrieve components by ID or identifier
- **API**: `GET /api/components/{identifier}`
- **Input**: `identifier` can be component ID or slug
- **Response**: Component object with all configuration fields
- **Error Handling**:
  - `404`: Component not found

**FR-COMPONENT-3: Component Listing**
- **Requirement**: Users must be able to list all components with optional filtering
- **API**: `GET /api/components`
- **Query Parameters**:
  - `type` (optional): Filter by component type
  - `tags` (optional): Filter by tags
- **Response**: Array of component objects

**FR-COMPONENT-4: Component Update**
- **Requirement**: Users must be able to update existing component configurations
- **API**: `PATCH /api/components/{component_id}`
- **Input Schema**: All fields optional (partial update)
- **Behavior**:
  - Updates component in database
  - Preserves component ID and creation timestamp
  - Updates `updated_at` timestamp
- **Error Handling**:
  - `404`: Component not found (exact message: `"Component with ID '{component_id}' not found."`)
  - `400`: Invalid configuration

**FR-COMPONENT-5: Component Deletion**
- **Requirement**: Users must be able to delete components
- **API**: `DELETE /api/components/{component_id}`
- **Behavior**:
  - Deletes component from database
  - Note: Does not validate if component is referenced by flows (orphaned references may exist)
- **Error Handling**:
  - `404`: Component not found

**FR-COMPONENT-6: Component Execution**
- **Requirement**: Users must be able to execute components individually with input data
- **API**: `POST /api/components/execute/{component_id}`
- **Input Schema**:
  - `input` (required, object): Input data for component execution
  - Additional type-specific parameters as needed
- **Response**:
  - `output` (object): Component execution output
  - `metadata` (object): Execution metadata (latency, status, etc.)
- **Behavior**:
  - Loads component configuration from database
  - Routes to appropriate executor based on component type:
    - `component_executor.executor(component)` dispatches to type-specific executor
    - Executors: API executor, LLM executor, Agent executor, Guardrail executor, etc.
  - Executes component with input data
  - Returns output and metadata
- **Error Handling**:
  - `404`: Component not found
  - `400`: Invalid input data
  - `500`: Execution errors, executor errors

**FR-COMPONENT-7: Component Metadata Retrieval**
- **Requirement**: Users must be able to retrieve component metadata and schemas
- **API**: `GET /api/components_metadata`
- **Response**: Metadata about component types, schemas, and available executors
- **Use Case**: UI rendering, schema validation, component discovery

**FR-FLOW-1: Flow Creation**
- **Requirement**: Users must be able to create graph-based flows connecting components
- **API**: `POST /api/flows`
- **Input Schema**:
  - `identifier` or `slug` (required, string): Unique flow identifier (slug format)
  - `name` (required, string): Human-readable flow name
  - `description` (optional, string): Flow description
  - `nodes` (required, array): Array of flow nodes
    - `id` (required, string): Node identifier (unique within flow)
    - `component_id` (required, string): Component ID to execute at this node
    - `config` (optional, object): Node-specific configuration overrides
    - `position` (optional, object): UI position metadata `{x, y}`
  - `edges` (required, array): Array of flow edges (connections between nodes)
    - `source` (required, string): Source node ID
    - `target` (required, string): Target node ID
    - `condition` (optional, string): Conditional routing expression
  - `entry_node` (optional, string): Entry node ID (default: first node or node with no incoming edges)
  - `tags` (optional, array): Tags for categorization
  - `metadata` (optional, object): Additional metadata

- **Validation Rules**:
  - `identifier`/`slug` must be unique
  - All `component_id` references must exist in database
  - Graph must be a valid DAG (no circular dependencies)
  - All edges must reference valid node IDs
  - Entry node must exist in nodes array

- **Behavior**:
  - Persists flow to `aistudio_flows` table in PostgreSQL
  - Validates component references exist
  - Validates graph structure (DAG validation)
  - Returns flow object with generated `flow_id`

- **Error Handling**:
  - `400`: Invalid graph structure, circular dependencies, invalid component references
  - `409`: Flow with slug already exists
  - `500`: Database errors

**FR-FLOW-2: Flow Retrieval**
- **Requirement**: Users must be able to retrieve flows by ID or identifier
- **API**: `GET /api/flows/{identifier}`
- **Input**: `identifier` can be flow ID or slug
- **Response**: Flow object with nodes, edges, and all configuration
- **Error Handling**:
  - `404`: Flow not found

**FR-FLOW-3: Flow Listing**
- **Requirement**: Users must be able to list all flows
- **API**: `GET /api/flows`
- **Response**: Array of flow objects

**FR-FLOW-4: Flow Update**
- **Requirement**: Users must be able to update existing flow configurations
- **API**: `PATCH /api/flows/{flow_id}`
- **Input Schema**: All fields optional (partial update)
- **Behavior**:
  - Updates flow in database
  - Validates graph structure if nodes/edges changed
  - Preserves flow ID and creation timestamp
  - Updates `updated_at` timestamp
- **Error Handling**:
  - `404`: Flow not found
  - `400`: Invalid graph structure, circular dependencies

**FR-FLOW-5: Flow Deletion**
- **Requirement**: Users must be able to delete flows
- **API**: `DELETE /api/flows/{flow_id}`
- **Behavior**:
  - Deletes flow from database
- **Error Handling**:
  - `404`: Flow not found

**FR-FLOW-6: Flow Execution**
- **Requirement**: Users must be able to execute flows with input data
- **API**: `POST /api/flows/execute/{flow_id}`
- **Input Schema**:
  - `input` (required, object): Initial input data for entry node
  - `context` (optional, object): Additional execution context

- **Behavior**:
  - **Graph Execution**: Executes components in dependency order
  - **Dependency Resolution**: 
    - Starts with entry node (or nodes with no dependencies)
    - Executes nodes when all dependencies are satisfied
    - Parallel execution: Nodes with satisfied dependencies may execute concurrently
  - **Threading Model**: 
    - Each component executes in its own thread
    - `FlowExecutor` manages thread pool and synchronization
    - Threads joined in dependency order
  - **Conditional Routing**: 
    - Edges with `condition` expressions evaluated at runtime
    - Downstream nodes skipped if conditions evaluate to false
  - **Error Handling**:
    - Component execution errors captured and propagated
    - Downstream nodes may be skipped based on error handling mode
  - **Timeout**: 
    - Flow execution bounded by `WORKFLOW_MAX_EXECUTION_TIME` (default: 180 seconds)
    - Timeout enforced at flow level, not per-component

- **Response**:
  - `output` (object): Final flow output (from exit node or last executed node)
  - `node_outputs` (object): Outputs from each executed node `{node_id: output}`
  - `errors` (array): Errors encountered during execution
  - `metadata` (object): Execution metadata (latency, nodes_executed, etc.)

- **Error Handling**:
  - `404`: Flow not found
  - `400`: Invalid input data, graph validation errors
  - `500`: Execution errors, timeout errors

#### 4.5.2 Component Types

**Type 1: API Component**
- **Description**: Executes HTTP API calls
- **Configuration**:
  - `endpoint` (string): API endpoint URL
  - `method` (enum): HTTP method (`GET`, `POST`, `PUT`, `DELETE`, etc.)
  - `headers` (object): HTTP headers
  - `body` (object or string): Request body (may be template)
  - `timeout` (integer): Request timeout in seconds
- **Input**: Request parameters, body data
- **Output**: API response (status, headers, body)

**Type 2: LLM Component**
- **Description**: Executes LLM calls with prompt templates
- **Configuration**:
  - `model` (string): LLM model identifier
  - `prompt_template` (string): Prompt template with variables
  - `temperature` (float): Model temperature
  - `maxTokens` (integer): Maximum tokens
  - `provider` (string): LLM provider (optional, uses defaults if not specified)
- **Input**: Template variables
- **Output**: LLM response text

**Type 3: Agent Component**
- **Description**: Executes an agent (references agent by ID)
- **Configuration**:
  - `agent_id` (string): Agent ID to execute
  - `input_mapping` (object): Maps flow input to agent input format
- **Input**: Flow context
- **Output**: Agent response

**Type 4: Guardrail Component**
- **Description**: Validates text content against guardrail rules
- **Configuration**:
  - `guardrail_id` (string): Guardrail ID to execute
  - `validation_rules` (object): Guardrail-specific rules
- **Input**: Text content to validate
- **Output**: Validation result (`pass`/`fail`, reasons, suggestions)

**Type 5: Custom Component Types**
- **Description**: Extensible component types via component executor registry
- **Configuration**: Type-specific
- **Input/Output**: Type-specific

#### 4.5.3 Flow Execution Model

**Execution Algorithm:**
```
1. Load flow definition from database
2. Validate graph structure (DAG validation)
3. Identify entry node(s) (nodes with no incoming edges)
4. Initialize execution context with input data
5. While there are nodes to execute:
   a. Find nodes with satisfied dependencies (all dependencies completed)
   b. Execute eligible nodes in parallel (thread pool)
   c. Wait for node executions to complete
   d. Update execution context with node outputs
   e. Evaluate conditional edges (skip downstream if condition false)
   f. Mark completed nodes
6. Return final output and node outputs
```

**Dependency Resolution:**
- Nodes execute only when all source nodes (via incoming edges) have completed
- Nodes with no dependencies execute immediately (entry nodes)
- Nodes with satisfied dependencies may execute in parallel

**Conditional Routing:**
- Edges with `condition` expressions evaluated after source node completes
- If condition evaluates to false, target node is skipped
- Skipped nodes do not execute, and their outputs are not included in results

**Error Handling:**
- Component execution errors captured in `errors` array
- Error handling mode determines behavior:
  - `stop`: Flow execution stops on first error
  - `continue`: Flow execution continues, errors logged
  - `retry`: Failed components retried before continuing

**Timeout Management:**
- Flow-level timeout: `WORKFLOW_MAX_EXECUTION_TIME` (default: 180 seconds)
- Timeout enforced across entire flow execution
- Partial results returned if timeout occurs

#### 4.5.4 End-to-End Workflow: Component-Based Customer Support Flow

**Workflow Steps:**

1. **Create Components**
   ```
   POST /api/components (API Component: Customer Database Lookup)
   {
     "identifier": "customer-lookup",
     "name": "Customer Database Lookup",
     "type": "api",
     "config": {
       "endpoint": "https://api.example.com/customers/{customer_id}",
       "method": "GET",
       "headers": {"Authorization": "Bearer {api_key}"}
     }
   }

   POST /api/components (LLM Component: Response Generation)
   {
     "identifier": "response-generator",
     "name": "Response Generator",
     "type": "llm",
     "config": {
       "model": "gpt-4",
       "prompt_template": "Generate a helpful response for customer {{customer_name}} regarding {{inquiry}}",
       "temperature": 0.7
     }
   }

   POST /api/components (Guardrail Component: Safety Check)
   {
     "identifier": "safety-check",
     "name": "Safety Check",
     "type": "guardrail",
     "config": {
       "guardrail_id": "content-safety",
       "validation_rules": {"check_toxicity": true}
     }
   }
   ```
   **Response**: Component objects with `component_id` for each

2. **Create Flow**
   ```
   POST /api/flows
   {
     "identifier": "customer-support-flow",
     "name": "Customer Support Flow",
     "description": "Lookup customer, generate response, validate safety",
     "nodes": [
       {
         "id": "node-1",
         "component_id": "{customer-lookup-component-id}",
         "config": {}
       },
       {
         "id": "node-2",
         "component_id": "{response-generator-component-id}",
         "config": {}
       },
       {
         "id": "node-3",
         "component_id": "{safety-check-component-id}",
         "config": {}
       }
     ],
     "edges": [
       {
         "source": "node-1",
         "target": "node-2"
       },
       {
         "source": "node-2",
         "target": "node-3"
       }
     ],
     "entry_node": "node-1"
   }
   ```
   **Response**: `201 Created` with `flow_id`

3. **Execute Flow**
   ```
   POST /api/flows/execute/{flow_id}
   {
     "input": {
       "customer_id": "12345",
       "inquiry": "I need help with my order"
     }
   }
   ```
   **Response**:
   ```json
   {
     "output": {
       "status": "pass",
       "response": "Thank you for contacting us..."
     },
     "node_outputs": {
       "node-1": {
         "customer_name": "John Doe",
         "customer_email": "john@example.com"
       },
       "node-2": {
         "response": "Thank you for contacting us..."
       },
       "node-3": {
         "status": "pass",
         "checks": ["toxicity: pass"]
       }
     },
     "errors": [],
     "metadata": {
       "latency_ms": 2500,
       "nodes_executed": 3,
       "execution_time": "2024-01-15T10:05:30Z"
     }
   }
   ```

4. **Update Flow (Add Conditional Branching)**
   ```
   PATCH /api/flows/{flow_id}
   {
     "nodes": [
       ...existing nodes...,
       {
         "id": "node-4",
         "component_id": "{escalation-component-id}",
         "config": {}
       }
     ],
     "edges": [
       ...existing edges...,
       {
         "source": "node-3",
         "target": "node-4",
         "condition": "output.status == 'fail'"
       }
     ]
   }
   ```
   **Response**: Updated flow object

5. **Re-execute Flow with Conditional Routing**
   - If guardrail fails, escalation node executes
   - If guardrail passes, escalation node is skipped

**Workflow Duration**: ~3 seconds for complete flow execution

**Success Criteria**:
- Components created and reusable across multiple flows
- Flow executes components in dependency order
- Conditional routing works correctly
- Node outputs captured and returned
- Errors handled gracefully

---

### 4.6 LLM Operations (Part 5)

#### 4.6.1 Functional Requirements

**FR-LLM-1: LLM Response API (Chat or Text)**
- **Requirement**: Users must be able to call LLMs via standardized HTTP API
- **API**: `POST /api/v1/llm/response` (LLM Engine service)
- **Input Schema**:
  - `prompt` (optional, string): Text prompt (if `messages` not provided)
  - `messages` (optional, array): Chat messages `[{role: "user", content: "..."}, ...]`
  - `model` (optional, string): Model identifier (uses default if not provided)
  - `temperature` (optional, float): Model temperature
  - `max_tokens` (optional, integer): Maximum tokens
  - `trace_context` (optional, object): Trace metadata (see FR-TRACE-1)
- **Validation Rules**:
  - Either `prompt` or `messages` must be provided (not both)
  - `prompt` cannot be empty/whitespace-only (exact error: `"Prompt cannot be empty"`)
  - `messages` must have at least one message with non-empty content
- **Behavior**:
  - If `prompt` provided: Wraps into single message `{"role": "user", "content": prompt}`
  - If `messages` provided: Converts each message via `message.model_dump()` and forwards to upstream
  - Calls upstream LiteLLM via OpenAI-compatible client:
    - `base_url = LITELLM_BASE_URL`
    - `api_key = LITELLM_API_KEY`
    - Uses `run_in_executor` (threadpool) to avoid blocking event loop
  - Computes cost estimate (best-effort, limited model support)
  - Emits trace payload to `TRACE_API_URL + "/traces"` if configured (best-effort)
- **Response**:
  - `response` (string): LLM response text
  - `metadata` (object): Usage metadata (tokens, cost, latency) - usage fields are stringified
- **Error Handling**:
  - `400`: Empty prompt, invalid input
  - `500`: Upstream errors, execution errors
  - `502`: Upstream service unavailable

**FR-LLM-2: Embedding API**
- **Requirement**: Users must be able to generate embeddings via standardized HTTP API
- **API**: `POST /api/v1/llm/embedding` (LLM Engine service)
- **Input Schema**:
  - `text` (required, string): Text to embed
  - `model` (optional, string): Embedding model identifier
  - `trace_context` (optional, object): Trace metadata
- **Validation Rules**:
  - `text` cannot be empty/whitespace-only (exact error: `"Text cannot be empty"`)
- **Behavior**:
  - Calls upstream embeddings API via OpenAI-compatible client
  - **Note**: Embedding call is currently event-loop blocking (not wrapped in executor)
  - Emits trace payload to `TRACE_API_URL + "/traces"` if configured (best-effort)
- **Response**:
  - `embedding` (array of floats): Embedding vector
  - `vector_dim` (integer): Vector dimension (equals `len(embedding)`)
- **Error Handling**:
  - `400`: Empty text, invalid input
  - `500`: Upstream errors, execution errors
  - `502`: Upstream service unavailable

**FR-LLM-3: LLM Judge API**
- **Requirement**: Users must be able to score/compare candidate responses using an LLM judge
- **API**: `POST /api/v1/playground/llm-judge` (LLM Engine service)
- **Input Schema**:
  - `user_prompt` (required, string or array): User prompt (string or message array)
  - `responses` (required, array): 1-4 candidate responses
    - Each response: `{response: string, id?: string}` (response must be non-empty)
  - `model_name` (optional, string): Judge model (default: `LLMRegistryModelEnums.SANGRIA_REASONING.value`)
- **Validation Rules**:
  - `responses` must have 1-4 items
  - Each response must have non-empty `response` field
  - If all responses are empty/whitespace: returns `400` with exact message `"At least one non-empty response is required."`
- **Behavior**:
  - Fetches judge prompt from MongoDB:
    - Database: `MONGO_DB_NAME`
    - Collection: **hardcoded** `"aistudio_system_prompts"`
    - Query: `name == JUDGE_PROMPT_NAME` (default: `"LLMJudgePrompt"`)
    - Selects **production** version: `labels` contains `"production"`
  - Renders judge prompt template with Jinja2 **StrictUndefined**:
    - Variables: `user_prompt`, `responses` (formatted sections)
    - Missing variables produce HTTP 422 error
  - Calls judge model via LLM response service (internal call)
  - Parses judge output as JSON:
    - Cleans JSON (removes markdown code blocks, whitespace)
    - Validates JSON structure
  - Validates judge JSON has required score fields
- **Response**:
  - `scores` (array): Array of score objects per response
  - `reasoning` (string): Judge reasoning
  - `winner` (string or null): Winning response ID (if applicable)
- **Error Handling**:
  - `400`: Invalid input, all responses empty
  - `404`: Judge prompt not found in MongoDB
  - `422`: Missing template variables
  - `502`: Judge returned invalid JSON (exact: `"Judge returned invalid JSON"`)
  - `502`: Judge returned empty response (exact: `"Judge returned empty response"`)
  - `502`: Judge JSON missing score fields (exact: `"Judge JSON missing score fields"`)

**FR-LLM-4: Chat Completions Passthrough (LangChain)**
- **Requirement**: Users must be able to call LLMs with tool/function calling support via LangChain
- **API**: `POST /api/v1/chat/completions` (LLM Engine service)
- **Input Schema**:
  - `messages` (required, array): Chat messages
  - `tools` (optional, array): Tool definitions for function calling
  - `model` (optional, string): Model identifier
  - Additional OpenAI-compatible parameters
- **Behavior**:
  - Uses LangChain `ChatOpenAI` client configured with:
    - `base_url = LITELLM_BASE_URL`
    - `api_key = LITELLM_API_KEY`
  - Binds tools if present
  - Executes chat completion with tool calling support
- **Response**:
  - `content` (string): Response content
  - `tool_calls` (array): Tool calls if model requested tools
- **Error Handling**:
  - `400`: Invalid input
  - `500`: Execution errors

**FR-LLM-5: LiteLLM Registry Proxy API**
- **Requirement**: Users must be able to manage models via LiteLLM registry proxy
- **APIs** (LLM Engine service):
  - `GET /api/v1/llm-registry/health`: Health check for LiteLLM service
  - `GET /api/v1/llm-registry/health/{model_name}`: Health check for specific model
  - `GET /api/v1/llm-registry/models`: List available models
  - `GET /api/v1/llm-registry/models/info?litellm_model_id=...`: Get model info
  - `POST /api/v1/llm-registry/model/new`: Create new model
  - `POST /api/v1/llm-registry/model/update`: Update model
  - `PATCH /api/v1/llm-registry/model/{model_id}/update`: Patch model
  - `POST /api/v1/llm-registry/model/delete`: Delete model
  - `GET /api/v1/llm-registry/provider/{provider}/schema`: Get provider schema from MongoDB
  - `GET /api/v1/llm-registry/providers`: List providers
- **Behavior**:
  - Proxies requests to upstream LiteLLM service:
    - Base URL: `LITELLM_BASE_URL`
    - Authorization: `Bearer {LITELLM_API_KEY}`
    - Uses `httpx.AsyncClient` for async HTTP calls
  - Provider schemas stored in MongoDB (`MODEL_CONFIG_COLLECTION`)
  - Provider-change prevention:
    - Update rejects mismatched provider/type with `400` (exact: `"Changing provider during update is not allowed."`)
    - Patch rejects `provider` field and `litellm_params.type` with `400` (exact: `"Changing provider during patch is not allowed."`)
- **Response**: Varies by endpoint (LiteLLM response models)
- **Error Handling**:
  - `400`: Invalid configuration, provider-change attempts
  - `404`: Model not found
  - `504`: Upstream timeout (exact detail: `{"error": "Request timeout", "message": "The LiteLLM service did not respond within the timeout period"}`)
  - `502`: Upstream errors (surfaces upstream JSON/body)

**FR-LLM-6: LLM Proxy Endpoints (Prompt Engine)**
- **Requirement**: Prompt Engine provides proxy endpoints to upstream LLM service
- **APIs** (Prompt Engine service):
  - `GET /api/v1/llms/models`: Proxy to `GET {LLM_PROXY_BASE_URL}/litellm/models`
  - `POST /api/v1/llms/response`: Proxy to `POST {LLM_PROXY_BASE_URL}/llm/response`
  - `POST /api/v1/llms/embedding`: Proxy to `POST {LLM_PROXY_BASE_URL}/llm/embedding`
  - `POST /api/v1/llms/playground/llm-judge`: Proxy to `POST {LLM_PROXY_BASE_URL}/playground/llm-judge`
- **Behavior**:
  - Proxies requests to upstream LLM service
  - **Note**: Does not forward `Authorization` headers (no auth forwarding)
  - Timeout: `LLM_PROXY_TIMEOUT` (default: 30 seconds)
- **Error Handling**:
  - `502`: Upstream service unavailable (exact: `"Unable to reach upstream LLM service"`)
  - `502`: Invalid judge response from upstream

#### 4.6.2 Trace Context and Emission

**Trace Context Structure:**
- `name` (required, string): Trace name/identifier
- `environment` (optional, string): Environment identifier (from `ENV` env var if not provided)
- `tags` (optional, array): Tags for categorization
- `metadata` (optional, object): Additional metadata (string:string key-value pairs)
- `session_id` (optional, string): Session identifier
- `project_id` (optional, string): Project identifier

**Trace Emission Behavior:**
- **Service-Overwritten Fields**:
  - `trace_context["input_data"]` is always overwritten by service with actual prompt/messages
  - `trace_context["output_data"]` is always set by service with LLM response
- **Trace Payload Sent**:
  - Built in trace decorator (`app/decorators/tracesDecorator.py`)
  - Posted to `TRACE_API_URL + "/traces"` (if `TRACE_API_URL` configured)
  - **Best-Effort**: Failures are logged and swallowed (not surfaced to caller)
  - Timeout: 10 seconds for trace POST

**Configuration Mismatch Warning:**
- Trace decorator posts to `TRACE_API_URL + "/traces"`
- Examples/documentation may treat `TRACE_API_URL` as full endpoint (ending in `/traces`)
- **Requirement**: Operators must set `TRACE_API_URL` as base URL (without trailing `/traces`)

#### 4.6.3 End-to-End Workflow: LLM Call with Tracing

**Workflow Steps:**

1. **Call LLM Response API**
   ```
   POST /api/v1/llm/response
   {
     "prompt": "Explain quantum computing in simple terms",
     "model": "gpt-4",
     "temperature": 0.7,
     "trace_context": {
       "name": "quantum-explanation",
       "environment": "production",
       "tags": ["education", "quantum"],
       "session_id": "session-123",
       "project_id": "project-456"
     }
   }
   ```
   **Response**:
   ```json
   {
     "response": "Quantum computing is a type of computation...",
     "metadata": {
       "usage": {
         "prompt_tokens": "150",
         "completion_tokens": "300",
         "total_tokens": "450"
       },
       "cost_usd": "0.015",
       "latency_ms": 1250
     }
   }
   ```

2. **Trace Emission (Background)**
   - Service overwrites `input_data` with actual prompt
   - Service sets `output_data` with LLM response
   - Trace decorator measures latency
   - POST to `TRACE_API_URL + "/traces"` (best-effort, async)

3. **Verify Trace (Optional)**
   ```
   GET /api/v1/traces?name=quantum-explanation&session_id=session-123
   ```
   **Response**: Trace object with full context

**Workflow Duration**: ~1.5 seconds (LLM call + trace emission)

**Success Criteria**:
- LLM response returned successfully
- Trace emitted to trace backend (if configured)
- Metadata includes usage and cost information
- Trace queryable by name, session, project

---

### 4.7 Observability and Tracing (Part 6)

#### 4.7.1 Functional Requirements

**FR-TRACE-1: Trace Ingestion**
- **Requirement**: Services must be able to ingest structured traces for observability
- **API**: `POST /api/v1/traces` (Prompt Engine service)
- **Input Schema**:
  - `name` (required, string): Trace name/identifier
  - `latency_ms` (optional, integer): Latency in milliseconds
  - `input_data` (optional, string or object): Input data (JSON string or raw string)
  - `output_data` (optional, string or object): Output data (JSON string or raw string)
  - `environment` (optional, string): Environment identifier
  - `tags` (optional, array): Tags for categorization
  - `metadata` (required, object): Metadata object (stored as JSON string in ClickHouse)
    - Common fields: `cost_usd`, `completion_tokens`, `prompt_tokens`, `total_tokens`
  - `session_id` (optional, string): Session identifier
  - `project_id` (optional, string): Project identifier (defaults to `"default"` if empty)
- **Behavior**:
  - Generates unique `trace_id` (UUID v4)
  - Returns `202 Accepted` immediately with `trace_id`
  - Inserts trace into ClickHouse **asynchronously** via FastAPI `BackgroundTasks`
  - Background task:
    - Generates server timestamp (UTC)
    - Coerces `project_id` to `"default"` if empty
    - Serializes `input_data`, `output_data`, `metadata` to JSON strings
    - Wraps dicts in `output_data` to list before storage (ClickHouse array handling)
    - INSERT into ClickHouse table (`aiprompt_traces` by default)
    - Errors are logged and NOT returned to caller
- **Response**:
  - `202 Accepted` with body:
    ```json
    {
      "accepted": true,
      "trace_id": "uuid-here"
    }
    ```
- **Error Handling**:
  - `422`: Invalid payload (Pydantic validation)
  - **Note**: Background insert failures are logged but not surfaced (caller receives 202)

**FR-TRACE-2: Trace Querying**
- **Requirement**: Users must be able to query traces with filtering and pagination
- **API**: `GET /api/v1/traces` (Prompt Engine service)
- **Query Parameters**:
  - `from` (optional, datetime ISO string): Start date (required if `to` provided)
  - `to` (optional, datetime ISO string): End date (default: now, max 1 day in future)
  - `name` (optional, string): Filter by trace name
  - `search` (optional, string): Search in input/output data
  - `environment` (optional, string): Filter by environment
  - `tag` (optional, string): Filter by tag
  - `session_id` (optional, string): Filter by session
  - `project_id` (optional, string): Filter by project
  - `limit` (optional, integer): Pagination limit (1-500, default: 10)
  - `offset` (optional, integer): Pagination offset (default: 0)
- **Validation Rules**:
  - Date range: `from <= to` (exact error: `"Invalid date range: 'from' date must be less than or equal to 'to' date"`)
  - Max range: 365 days (exact error: `"Date range too large: maximum allowed range is 365 days"`)
  - `to` max: 1 day in future (exact error: `"'to' date cannot be more than 1 day in the future"`)
  - `limit` must be 1-500
- **Behavior**:
  - Builds ClickHouse query dynamically:
    - Uses `PREWHERE` for time window (if provided) for performance
    - Optional filters: name, search, environment, tag, session_id, project_id
    - Cost aggregation reads `metadata` JSON with `JSONHas`/`JSONExtractFloat`
  - Parses `metadata` JSON string back to object (best-effort; invalid JSON becomes `{}`)
  - Normalizes timestamps to UTC ISO strings
- **Response**: Object (not bare array):
  ```json
  {
    "items": [...trace objects...],
    "total": 1000,
    "total_cost": "125.50",
    "limit": 10,
    "offset": 0
  }
  ```
- **Error Handling**:
  - `400`: Invalid date range, limit out of bounds
  - `500`: ClickHouse query errors

**FR-TRACE-3: Trace Retrieval by ID**
- **Requirement**: Users must be able to retrieve specific traces by trace_id
- **API**: `GET /api/v1/traces/{trace_id}` (Prompt Engine service)
- **Input**: `trace_id` must be valid UUID (36 characters)
- **Response**: Single trace object with:
  - `trace_id` (string)
  - `timestamp` (datetime ISO string, UTC)
  - `name` (string)
  - `latency_ms` (integer)
  - `input_data` (string or object, parsed from JSON)
  - `output_data` (string or object, parsed from JSON)
  - `environment` (string)
  - `tags` (array)
  - `metadata` (object, parsed from JSON)
  - `session_id` (string)
  - `project_id` (string)
- **Error Handling**:
  - `400`: Invalid UUID format
  - `404`: Trace not found

**FR-TRACE-4: Session Aggregation**
- **Requirement**: Users must be able to aggregate traces by session for cost/token analysis
- **API**: `GET /api/v1/traces/session-ids` (Prompt Engine service)
- **Query Parameters**:
  - `from` (optional, datetime): Start date
  - `to` (optional, datetime): End date
  - `project_id` (optional, string): Filter by project
  - `limit` (optional, integer): Pagination limit
  - `offset` (optional, integer): Pagination offset
- **Behavior**:
  - Aggregates traces by `session_id`
  - Extracts cost/tokens from `metadata` JSON:
    - `cost_usd`: Sums `JSONExtractFloat(metadata, 'cost_usd')`
    - `tokens`: Sums `JSONExtractFloat(metadata, 'total_tokens')`
  - Groups by `session_id` and aggregates metrics
- **Response**: Object:
  ```json
  {
    "items": [
      {
        "session_id": "session-123",
        "total_cost": "25.50",
        "total_tokens": 50000,
        "trace_count": 150
      }
    ],
    "total": 10,
    "limit": 10,
    "offset": 0
  }
  ```

**FR-TRACE-5: Trace Table Creation**
- **Requirement**: Users must be able to create ClickHouse trace table schema
- **API**: `POST /api/v1/traces/create-table` (Prompt Engine service)
- **Behavior**:
  - Creates ClickHouse table if not exists:
    - Table name: `CLICKHOUSE_TABLE` (default: `aiprompt_traces`)
    - Database: `CLICKHOUSE_DB`
    - Schema:
      - `trace_id` (String)
      - `timestamp` (DateTime64(6))
      - `name` (String)
      - `latency_ms` (UInt64)
      - `input_data` (String) - JSON string or raw string
      - `output_data` (String) - JSON string or raw string
      - `environment` (String)
      - `tags` (Array(String))
      - `metadata` (String) - JSON string
      - `session_id` (String)
      - `project_id` (String) DEFAULT 'default'
  - Returns `201 Created` if table created, `200 OK` if already exists
- **Response**:
  - `201 Created`: Table created
  - `200 OK`: Table already exists
- **Error Handling**:
  - `500`: ClickHouse connection/creation errors

#### 4.7.2 ClickHouse Schema

**Trace Table Schema:**
```sql
CREATE TABLE IF NOT EXISTS aiprompt_traces (
    trace_id String,
    timestamp DateTime64(6),
    name String,
    latency_ms UInt64,
    input_data String,           -- JSON string or raw string
    output_data String,          -- JSON string or raw string
    environment String,
    tags Array(String),
    metadata String,             -- JSON string
    session_id String,
    project_id String DEFAULT 'default'
) ENGINE = MergeTree()
ORDER BY (timestamp, trace_id);
```

**Migration Tracking Table:**
```sql
CREATE TABLE IF NOT EXISTS <db>.migrations (
    migration_name String,
    migration_version String,
    status String,               -- 'pending'|'running'|'success'|'failed'
    added_at DateTime DEFAULT now(),
    executed_at Nullable(DateTime),
    execution_duration_ms Nullable(UInt64),
    error_message Nullable(String),
    checksum String,
    description Nullable(String)
) ENGINE = MergeTree()
ORDER BY (added_at, migration_name);
```

#### 4.7.3 End-to-End Workflow: Trace Ingestion and Analysis

**Workflow Steps:**

1. **Ingest Trace**
   ```
   POST /api/v1/traces
   {
     "name": "llm-call",
     "latency_ms": 150,
     "input_data": "Explain quantum computing",
     "output_data": "Quantum computing is...",
     "environment": "production",
     "tags": ["llm", "gpt-4"],
     "metadata": {
       "cost_usd": "0.01",
       "completion_tokens": "42",
       "prompt_tokens": "10",
       "total_tokens": "52"
     },
     "session_id": "session-123",
     "project_id": "project-456"
   }
   ```
   **Response**: `202 Accepted` with `trace_id`

2. **Query Traces**
   ```
   GET /api/v1/traces?from=2024-01-15T00:00:00Z&to=2024-01-15T23:59:59Z&project_id=project-456&limit=100
   ```
   **Response**:
   ```json
   {
     "items": [...trace objects...],
     "total": 500,
     "total_cost": "125.50",
     "limit": 100,
     "offset": 0
   }
   ```

3. **Aggregate by Session**
   ```
   GET /api/v1/traces/session-ids?from=2024-01-15T00:00:00Z&to=2024-01-15T23:59:59Z
   ```
   **Response**:
   ```json
   {
     "items": [
       {
         "session_id": "session-123",
         "total_cost": "25.50",
         "total_tokens": 50000,
         "trace_count": 150
       }
     ],
     "total": 1,
     "limit": 10,
     "offset": 0
   }
   ```

4. **Retrieve Specific Trace**
   ```
   GET /api/v1/traces/{trace_id}
   ```
   **Response**: Full trace object with parsed metadata

**Workflow Duration**: < 1 second for trace ingestion, < 2 seconds for queries

**Success Criteria**:
- Traces ingested asynchronously (202 Accepted)
- Traces queryable with filters and date ranges
- Cost aggregation works correctly
- Session aggregation provides accurate metrics
- Trace retention supports long-term analysis

---

## 5. Non-Functional Requirements

### 5.1 Performance Requirements

#### 5.1.1 Latency Requirements

**NFR-PERF-1: API Response Latency**
- **Requirement**: API endpoints must respond within acceptable latency thresholds
- **Targets**:
  - **Health Endpoints**: p95 < 100ms (including dependency checks)
  - **CRUD Operations** (agents, workflows, prompts, components, flows):
    - Create: p95 < 500ms
    - Read: p95 < 200ms
    - Update: p95 < 500ms
    - Delete: p95 < 300ms
  - **Prompt Production Retrieval** (cached): p95 < 50ms
  - **Prompt Production Retrieval** (cache miss): p95 < 200ms
  - **Template Rendering**: p95 < 100ms
  - **LLM Response API**: p95 within acceptable SLA for upstream LiteLLM service (typically 2-5 seconds)
  - **Embedding API**: p95 within acceptable SLA for upstream service (typically 1-3 seconds)
  - **Trace Ingestion**: p95 < 50ms (202 Accepted response, async insert)
  - **Trace Queries**: p95 < 2 seconds for queries over 30-day windows
  - **Session Aggregation**: p95 < 3 seconds for aggregation across millions of traces

**NFR-PERF-2: Workflow Execution Latency**
- **Requirement**: Workflow executions must complete within reasonable timeframes
- **Targets**:
  - **Simple Sequential Workflow** (2-3 steps): p95 < 30 seconds
  - **Complex Multi-Agent Workflow** (5+ steps): p95 < 5 minutes
  - **Flow Execution** (graph-based): p95 < 3 minutes (bounded by `WORKFLOW_MAX_EXECUTION_TIME`: 180 seconds default)
- **Timeout Configuration**:
  - Agent execution timeout: `AGENT_EXECUTION_TIMEOUT` (default: 300 seconds)
  - Workflow execution timeout: Configurable per workflow (default: no limit, but bounded by thread pool)
  - Flow execution timeout: `WORKFLOW_MAX_EXECUTION_TIME` (default: 180 seconds)

**NFR-PERF-3: Throughput Requirements**
- **Requirement**: System must handle concurrent requests efficiently
- **Targets**:
  - **Concurrent Workflow Executions**: Support 10+ concurrent executions (ThreadPoolExecutor with 10 workers)
  - **Concurrent Agent Executions**: Support 20+ concurrent executions (AGENT_EXECUTOR_WORKERS: 20)
  - **Concurrent LLM Calls**: Limited by upstream LiteLLM service capacity
  - **Trace Ingestion Rate**: Support 1000+ traces/second (async background tasks)
  - **Prompt Retrieval Rate**: Support 100+ requests/second (Redis caching)

#### 5.1.2 Resource Utilization

**NFR-PERF-4: Memory Usage**
- **Requirement**: Services must operate within reasonable memory constraints
- **Targets**:
  - **Workflow Engine**: < 2GB per instance (excluding database connections)
  - **Prompt Engine**: < 1GB per instance (excluding database connections)
  - **LLM Engine**: < 1GB per instance
- **Optimization Strategies**:
  - Lazy client initialization (MongoDB, Redis, ClickHouse)
  - Connection pooling for database connections
  - Streaming responses for large payloads (WebSocket, SSE)

**NFR-PERF-5: CPU Usage**
- **Requirement**: Services must efficiently utilize CPU resources
- **Targets**:
  - **Idle CPU Usage**: < 5% per service instance
  - **Peak CPU Usage**: < 80% per service instance under normal load
- **Optimization Strategies**:
  - Async/await for I/O-bound operations
  - Thread pools for CPU-bound operations (agent execution, flow execution)
  - ClickHouse operations wrapped in `asyncio.to_thread()` to avoid blocking event loop

#### 5.1.3 Caching Performance

**NFR-PERF-6: Cache Hit Rates**
- **Requirement**: Caching must significantly improve performance for frequently accessed data
- **Targets**:
  - **Production Prompt Cache Hit Rate**: > 80% (Redis caching)
  - **Model Config Cache Hit Rate**: > 70% (in-process cache, 300-second TTL)
- **Cache Configuration**:
  - Redis TTL: `REDIS_TTL` (default: 3600 seconds)
  - Model config cache TTL: 300 seconds (hardcoded)
  - Cache invalidation: On production promotion, prompt deletion

