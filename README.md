Table of Contents

What Is This?
How It Works
Project Structure
Technology Stack — Deep Dives

Groq API
LangChain
LangGraph
LangSmith
FastAPI
Pydantic


Setup & Installation
Running the Project
API Reference
Architecture Decisions
How This Project Grows You as an Engineer
Contributing


What Is This?
The Autonomous Coding Squad is a production-grade multi-agent AI system built as an educational showcase of modern AI engineering patterns.
You submit a coding problem in plain English via REST API. Three specialist agents take over:
AgentRoleDeveloperWrites Python code from the problem statement and prior feedbackExecutorRuns the code in a real subprocess sandbox; captures stdout + stderrReviewerReads the code AND its execution results; approves or requests a fix
These three stages loop until the Reviewer approves the solution or the maximum iteration count is reached. The system is fully async, stateful, observable, and exposed via a clean REST API.

How It Works
POST /api/v1/solve  ─────────────────────────────────────────────────────────┐
                                                                               │
  ┌────────────────────────────────────────────────────────────────────────┐  │
  │                      LangGraph State Machine                           │  │
  │                                                                        │  │
  │   START                                                                │  │
  │     │                                                                  │  │
  │     ▼                                                                  │  │
  │  ┌──────────────────┐                                                  │  │
  │  │  developer_node  │  ← Calls Groq LLM (Llama 3 70B) to write code   │  │
  │  └────────┬─────────┘    Gets feedback from prior Reviewer iteration   │  │
  │           │                                                            │  │
  │           ▼                                                            │  │
  │  ┌──────────────────┐                                                  │  │
  │  │  executor_node   │  ← Runs code in subprocess sandbox               │  │
  │  └────────┬─────────┘    Captures stdout (output) + stderr (errors)   │  │
  │           │                                                            │  │
  │      [conditional]─── error + iterations < max ──► developer_node     │  │
  │           │                                                            │  │
  │           ▼                                                            │  │
  │  ┌──────────────────┐                                                  │  │
  │  │  reviewer_node   │  ← Calls Groq LLM to audit code + execution      │  │
  │  └────────┬─────────┘                                                  │  │
  │           │                                                            │  │
  │      [approved] ────────────────────────────────────────────► END      │  │
  │      [rejected + iterations < max] ─────────────► developer_node       │  │
  │      [rejected + iterations >= max] ───────────────────────► END       │  │
  │                                                                        │  │
  └────────────────────────────────────────────────────────────────────────┘  │
                                                                               │
  JSON Response ◄────────────────────────────────────────────────────────────┘
  { thread_id, status, final_code, execution_output, iterations_used, ... }
Every node reads from and writes to AgentState — a shared TypedDict that acts as the graph's memory. LangGraph manages all state transitions safely between concurrent requests using MemorySaver checkpointing, keyed by a per-request UUID.

Project Structure
autonomous-coding-squad/
│
├── .env                        # Environment variables (API keys, config)
├── .env.example                # Template — copy this to .env
├── .gitignore                  # Files excluded from version control
├── README.md                   # This file
├── requirements.txt            # Python dependencies
│
└── src/
    ├── config.py               # Pydantic Settings — loads + validates .env at startup
    │
    ├── agents/
    │   ├── __init__.py
    │   ├── developer.py        # Developer LangChain chain (writes code)
    │   └── reviewer.py         # Reviewer LangChain chain (audits code + execution)
    │
    ├── api/
    │   ├── __init__.py
    │   ├── main.py             # FastAPI app factory + lifespan startup/shutdown
    │   ├── routes.py           # POST /solve and GET /health route handlers
    │   └── schemas.py          # Pydantic I/O models for the API contract
    │
    ├── graph/
    │   ├── __init__.py
    │   ├── builder.py          # Compiles the LangGraph state machine (module-level singleton)
    │   ├── edges.py            # Pure routing functions (conditional edge logic)
    │   ├── nodes.py            # Async node functions (developer, executor, reviewer)
    │   └── state.py            # AgentState TypedDict — the graph's shared memory
    │
    └── tests/
        ├── __init__.py
        └── test_endpoint.py    # API integration tests
Responsibility Map
FileSingle Responsibilityconfig.pyLoad and validate all env vars at startup. Fail fast if anything is missing.agents/developer.pyDefine the Developer's prompt, output schema, and LCEL chain. Nothing else.agents/reviewer.pyDefine the Reviewer's prompt, output schema, and LCEL chain. Nothing else.graph/state.pyDeclare AgentState shape and reducers. The single source of truth for graph memory.graph/nodes.pyAsync node functions — call agents, run subprocesses, return state updates.graph/edges.pyPure routing functions — read state, return next node name. No business logic.graph/builder.pyAssemble and compile the graph once at import time.api/schemas.pyPydantic models for HTTP I/O. Completely independent of internal graph state.api/routes.pyThin HTTP adapter — validate input, invoke graph, handle errors, shape response.api/main.pyApp factory + lifespan. Import order is intentional (dotenv before LangChain).

Technology Stack — Deep Dives
Groq API — Lightning-Fast LPU Inference
What it is: Groq is a cloud inference provider that runs LLMs (Llama 3, Mixtral, Gemma) on custom LPU (Language Processing Unit) hardware. LPUs are purpose-built for the sequential token-generation workload of LLMs, producing dramatically lower latency than GPU inference.
Why it matters architecturally:

GPU inference is optimized for parallel matrix operations (great for training). Token generation is inherently sequential — GPUs have to fight their own architecture to do it.
Groq's LPU matches the workload: sequential, high-throughput token generation at ~10x the speed of comparable GPU inference.
The API is OpenAI-compatible, meaning LangChain's ChatGroq wrapper can be swapped for ChatOpenAI in one line.

How it's used:
python# Developer: temperature=0.1 — slight creativity for code generation
_dev_llm = ChatGroq(model=settings.developer_model, api_key=settings.groq_api_key, temperature=0.1)

# Reviewer: temperature=0.0 — completely deterministic approval decisions
_reviewer_llm = ChatGroq(model=settings.reviewer_model, api_key=settings.groq_api_key, temperature=0.0)
What this teaches you: Cloud provider patterns, API key security, hardware-aware design thinking, provider abstraction.

LangChain — The LLM Application Framework
What it is: LangChain is a Python framework that provides standardized building blocks for LLM applications: prompt templates, model wrappers, output parsers, chains, and tools. It abstracts away provider differences so your application logic is never tied to one LLM vendor.
Key concepts used:
ConceptWhat it doesChatPromptTemplateReusable, parameterized prompt structure. Separates prompt text from runtime variables.ChatGroqLangChain's wrapper implementing BaseChatModel. Swap with ChatOpenAI trivially.with_structured_output()Forces the LLM to return a Pydantic schema as JSON. No string parsing needed.LCEL (| operator)LangChain Expression Language — pipe composition for chains. Lazy, streaming-ready.HumanMessageTyped message object for seeding conversation history in AgentState.
The LCEL pipeline:
python# This single line creates a complete, streaming-ready LLM pipeline:
developer_chain = developer_prompt | _structured_dev_llm

# Invoking it returns a typed Pydantic object — not raw text:
result: CodeGenerationOutput = await developer_chain.ainvoke({
    "problem_statement": "...",
    "feedback": "Fix the off-by-one error on line 7"
})
print(result.code)  # Already parsed, already validated
What this teaches you: Pipeline composition, structured outputs, provider-agnostic design, contract-driven LLM integration.

LangGraph — Stateful Multi-Agent Orchestration
What it is: LangGraph is built on top of LangChain and lets you define AI workflows as a directed graph where nodes are functions and edges define the flow between them. Unlike simple chains, LangGraph supports cycles (feedback loops), conditional routing, shared state, and checkpointing — essential for any real agentic system.
The graph is based on Google's Pregel model: nodes communicate by reading and writing to a shared state object. After each node completes, LangGraph applies reducers to merge the node's state updates into the global AgentState.
AgentState — the graph's memory:
pythonclass AgentState(TypedDict):
    problem_statement: str
    messages: Annotated[list, add_messages]  # reducer: append, not replace
    generated_code: str
    execution_output: str
    execution_error: Optional[str]
    review_feedback: str
    reviewer_approved: bool
    iteration_count: int
    max_iterations: int
    thread_id: str
Conditional routing:
python# Routing functions are pure — they only read state and return a string
def route_after_reviewer(state: AgentState) -> str:
    if state["reviewer_approved"]:
        return END                              # success
    if state["iteration_count"] >= state["max_iterations"]:
        return END                              # give up
    return "developer"                          # try again
MemorySaver checkpointing: After EVERY node completes, MemorySaver snapshots the full AgentState keyed by thread_id. This gives you:

Crash recovery — resume from last checkpoint
State inspection — query any past state by thread_id
Multi-request isolation — concurrent requests never share state

What this teaches you: State machine design, event sourcing, cyclic graph orchestration, agentic system architecture, workflow engine patterns (Airflow, Temporal, Prefect use the same model).

LangSmith — Observability & Tracing
What it is: LangSmith is an observability platform for LLM applications. It automatically captures every LLM call, prompt, response, latency, token count, and error — giving you a full trace of exactly what happened during every request.
What it captures (zero code changes needed):
DataValueFull promptsExact text sent to Groq, including all injected variablesResponsesRaw LLM output before parsingToken countsInput + output tokens per call (essential for cost modeling)LatencyPer-node and per-call timingGraph visualizationInteractive view of the state machine with node timingErrorsFull stack traces linked to the exact prompt that triggered them
Setup (three env vars, zero code):
bashLANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=ls__your_key_here
LANGCHAIN_PROJECT=autonomous-coding-squad
LangChain instruments everything automatically via its callback system when these are set.
What this teaches you: Observability as a first-class engineering concern, distributed tracing patterns (same model as OpenTelemetry/Jaeger/Datadog APM), LLM cost modeling, debugging non-deterministic systems.

FastAPI — Async REST API
What it is: FastAPI is a modern Python web framework built on Starlette and Pydantic. It generates OpenAPI documentation automatically, validates all I/O against Pydantic models, and is built async-first using Python's asyncio — the correct foundation for LLM-backed APIs.
Why async is non-negotiable here:
A single LLM solve request can take 60–240 seconds. A synchronous handler would block its thread for the entire duration, making your server single-user. Async means the event loop handles hundreds of concurrent requests, each yielding control at await points (LLM calls, subprocess waits).
Key patterns:
python# asyncio.wait_for: the last line of defense against a hung LLM
final_state = await asyncio.wait_for(
    coding_squad_graph.ainvoke(initial_state, config=graph_config),
    timeout=float(settings.api_timeout_seconds),  # → 504 if exceeded
)
python# Lifespan: modern replacement for @app.on_event
@asynccontextmanager
async def lifespan(app: FastAPI):
    # startup: compile graph, log config
    yield
    # shutdown: cleanup
What this teaches you: Async I/O programming, contract-first API design, timeout engineering, HTTP error modeling, OpenAPI documentation as a product.

Pydantic — Validation and Settings
Pydantic v2 plays three distinct roles:
1. Agent output schemas — enforce structured LLM responses:
pythonclass CodeGenerationOutput(BaseModel):
    code: str = Field(description="The Python code. NO markdown backticks.")
2. API I/O models — validate HTTP boundaries:
pythonclass SolveRequest(BaseModel):
    problem_statement: str = Field(..., min_length=20, max_length=4000)
    max_iterations: Optional[int] = Field(default=None, ge=1, le=10)
3. Application settings — fail-fast at startup:
pythonclass Settings(BaseSettings):
    groq_api_key: str              # Required — app refuses to start if missing
    developer_model: str = "llama3-70b-8192"
    max_iterations: int  = 3
What this teaches you: Fail-fast validation, schema-first thinking, type safety, settings management, the principle of validating at the boundary.

Setup & Installation
Prerequisites

Python 3.11+
A Groq API key (free)
Optional: A LangSmith API key (free)

Install
bash# 1. Clone the repo
git clone https://github.com/yourusername/autonomous-coding-squad.git
cd autonomous-coding-squad

# 2. Create virtual environment
python -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Configure environment
cp .env.example .env
# Edit .env and add your GROQ_API_KEY
.env Configuration
bash# ── Required ───────────────────────────────────────────
GROQ_API_KEY=gsk_your_groq_key_here

# ── Model Selection (optional) ─────────────────────────
DEVELOPER_MODEL=llama3-70b-8192
REVIEWER_MODEL=llama3-70b-8192

# ── Behavior Tuning (optional) ─────────────────────────
MAX_ITERATIONS=3
API_TIMEOUT_SECONDS=300

# ── LangSmith Tracing (optional but highly recommended) ─
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=ls__your_langsmith_key_here
LANGCHAIN_PROJECT=autonomous-coding-squad

Running the Project
bash# Development (with auto-reload)
uvicorn src.api.main:app --host 0.0.0.0 --port 8000 --reload

# Production
uvicorn src.api.main:app --host 0.0.0.0 --port 8000 --workers 4
Navigate to http://localhost:8000/docs for the interactive Swagger UI.

API Reference
POST /api/v1/solve
Submit a coding problem to the agent squad.
Request:
json{
  "problem_statement": "Write a Python function that finds the longest palindromic substring in a given string. Include test cases in main().",
  "max_iterations": 3
}
Response:
json{
  "thread_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "approved",
  "final_code": "def longest_palindrome(s: str) -> str:\n    ...",
  "execution_output": "Test 1 passed: 'bab'\nTest 2 passed: 'bb'\n",
  "execution_error": null,
  "iterations_used": 2,
  "review_feedback": "The code executes correctly and handles all edge cases.",
  "approved": true
}
Status values:
ValueMeaningapprovedReviewer signed off. Code works.exhaustedMax iterations reached without approval.errorUnexpected graph execution failure.
GET /api/v1/health
Liveness probe for Docker/Kubernetes.
json{
  "status": "ok",
  "ollama_reachable": true,
  "message": "All systems operational."
}
Example with curl:
bashcurl -X POST http://localhost:8000/api/v1/solve \
  -H "Content-Type: application/json" \
  -d '{
    "problem_statement": "Write a Python function that computes the nth Fibonacci number iteratively. Test it for n=0,1,10,20 in main().",
    "max_iterations": 3
  }'

Architecture Decisions
Why compile the graph at module level?
Graph compilation validates the topology and sets up the Pregel runtime — it's expensive. Done at import time, not per-request, so the cost is paid once at startup.
Why separate schemas from internal state?
SolveRequest/SolveResponse (API contract) is independent of AgentState (internal state). This lets you evolve the graph internals without breaking API consumers, and vice versa.
Why asyncio.wait_for in addition to per-node timeouts?
Defense in depth. Per-node timeouts catch slow individual operations. wait_for catches pathological cases where the graph itself hangs (e.g., MemorySaver deadlock, graph topology bug). Two layers of protection are better than one.
Why is execution_error: Optional[str] instead of a boolean?
The actual stderr content is the most useful debugging signal — both for the Reviewer agent (it reads it to write specific feedback) and for the API consumer (they can see exactly what went wrong). A boolean would discard that information.
Why temperature=0.0 for the Reviewer?
The approval decision is binary and consequential. Any non-zero temperature introduces randomness into a gate that controls the feedback loop. The Reviewer should be a deterministic judge, not a creative one.

How This Project Grows You as an Engineer
Skill AreaWhat You LearnAPI securityEnvironment variable management, API key handling, never hardcoding secretsAsync programmingasyncio, await, event loop architecture, concurrent request handlingState machinesDirected graphs, conditional routing, cyclic execution, Pregel modelContract-driven designPydantic schemas as contracts between layers, API-first thinkingObservabilityDistributed tracing, token cost modeling, debugging LLM non-determinismAgentic patternsAgent roles, tool use, feedback loops, multi-agent coordinationFail-fast principlesValidate at the boundary, startup validation, defensive timeoutsProvider abstractionSwap LLM providers without touching business logicSeparation of concernsAgents / Nodes / Edges / State / API — each layer has exactly one jobProcess sandboxingSubprocess isolation, stdout/stderr capture, security boundaries
The architecture patterns here — state machines, provider abstraction, contract-first design, observability as a first-class concern — appear in production systems at every major tech company. This project is a compact, runnable example of all of them working together.

Contributing
Contributions are welcome. Please open an issue before submitting a PR describing what you'd like to change and why. All code should follow the existing separation of concerns: agents stay in agents/, routing logic stays in edges.py, and routes stay thin.

License
MIT — use it, fork it, learn from it, build on it.
