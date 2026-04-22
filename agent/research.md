# AgentScope - Phân tích toàn diện codebase

> Nguồn: `/Users/vanductai/Repo/research/oss/agentscope`
> Tác giả: Alibaba Tongyi Lab
> Ngày phân tích: 2026-04-22

## 1. Tổng quan kiến trúc

AgentScope là một multi-agent framework **async-first**, xây dựng trên Python 3.10+, thiết kế cho production. Kiến trúc gồm các layer chính:

```
┌─────────────────────────────────────────────────┐
│  Application Layer (Workflows, Examples, Games)  │
├─────────────────────────────────────────────────┤
│  Orchestration: MsgHub, Pipeline, ChatRoom       │
├─────────────────────────────────────────────────┤
│  Agent Layer: ReActAgent, A2AAgent, Realtime...  │
├───────────┬──────────┬──────────┬───────────────┤
│  Model    │  Memory  │  Tool/   │  Formatter    │
│  Adapters │  System  │  Toolkit │               │
├───────────┴──────────┴──────────┴───────────────┤
│  Message System (Msg + ContentBlocks)            │
├─────────────────────────────────────────────────┤
│  Infrastructure: Tracing, Session, Embedding     │
└─────────────────────────────────────────────────┘
```

---

## 2. Logic hoạt động chi tiết từng module

### 2.1 Message System (Lõi giao tiếp)

- **Msg**: Object trung tâm chứa `name`, `role` (user/assistant/system), `content` (text hoặc list ContentBlock), `metadata` (structured output), `timestamp`
- **7 loại ContentBlock**: `TextBlock`, `ThinkingBlock`, `ImageBlock`, `AudioBlock`, `VideoBlock`, `ToolUseBlock`, `ToolResultBlock`
- Mọi giao tiếp giữa agent đều thông qua `Msg` - tạo ra một **lingua franca** thống nhất cho multimodal content

### 2.2 Agent Layer

**AgentBase** → abstract base:
- `async reply(msg)` → xử lý input, trả về `Msg`
- `async observe(msg)` → nhận message từ agent khác (dùng trong MsgHub)
- **Hook system 6 vị trí**: `pre_reply`, `post_reply`, `pre_print`, `post_print`, `pre_observe`, `post_observe`
- **Subscriber pattern**: agent tự động broadcast reply tới subscribers

**ReActAgent** (agent chính, ~1138 dòng) - logic vòng lặp:

```
reply(msg) được gọi
  ├── 1. Lưu msg vào memory
  ├── 2. Retrieve từ long-term memory (nếu có)
  ├── 3. Retrieve từ knowledge base/RAG (nếu có)
  ├── 4. Setup structured output tool (nếu cần)
  ├── 5. LOOP (max_iters lần):
  │     ├── Compress memory nếu vượt threshold
  │     ├── _reasoning(): Format memory → gọi LLM → nhận response
  │     ├── _acting(): Thực thi tool calls (parallel hoặc sequential)
  │     ├── Kiểm tra exit condition:
  │     │    ├── Có structured output? → tạo reply + break
  │     │    └── Không có tool_use? → text reply → break
  │     └── Continue loop
  ├── 6. Post-process: record vào long-term memory
  └── 7. Return reply Msg
```

Tính năng nâng cao của ReActAgent:
- **Memory compression**: Khi context vượt `trigger_threshold` token → tự động nén lịch sử cũ thành structured summary (Pydantic `SummarySchema`)
- **Plan notebook**: Phân rã task phức tạp thành subtasks, agent tự quản lý tiến độ
- **Long-term memory 3 chế độ**: `agent_control`, `static_control`, `both`
- **RAG**: Rewrite query → search knowledge base → inject context
- **Realtime steering**: Hỗ trợ interrupt giữa chừng, graceful cancellation

### 2.3 Model Adapters

6 adapters: **OpenAI, Anthropic, DashScope (Qwen), Gemini, Ollama, Trinity**
- Unified interface `ChatModelBase` với `async __call__()` → `ChatResponse` hoặc `AsyncGenerator[ChatResponse]`
- Unified tool calling, streaming, structured output
- `ChatResponse` chứa list blocks: `TextBlock`, `ToolUseBlock`, `ThinkingBlock`, `AudioBlock`

### 2.4 Formatter

Mỗi model provider có formatter riêng chuyển `Msg` → format API cụ thể:
- `OpenAIChatFormatter`, `AnthropicChatFormatter`, `DashScopeChatFormatter`, `GeminiChatFormatter`, `OllamaChatFormatter`
- MultiAgent variants cho group chat scenarios
- Hỗ trợ truncation khi vượt context window

### 2.5 Memory System

**Working Memory (short-term)**:
- `InMemoryMemory`: List-based, in-process
- `AsyncSQLAlchemyMemory`: Persistent DB (SQLite, PostgreSQL...)
- `RedisMemory`: Distributed
- `TablestoreMemory`: Alibaba Cloud
- Mark system để tag messages: `HINT`, `COMPRESSED`

**Long-term Memory**:
- `Mem0`: Third-party service integration
- `ReMe`: Relationship-based memory (personal, task, tool)
- Interface: `retrieve(query)` + `record(messages)`

### 2.6 Tool / Toolkit System

- **Auto schema generation**: Parse Python docstrings → JSON schema cho LLM
- **Tool groups**: Nhóm tools, enable/disable động - agent có thể tự quản lý qua meta-tool `reset_equipped_tools`
- **Middleware pipeline**: Intercept tool execution (logging, validation, rate limiting...)
- **MCP integration**: Register tools từ MCP servers trực tiếp
- **Agent Skills**: Load skill directories với `SKILL.md` + scripts
- Built-in tools: `execute_python_code`, `execute_shell_command`, `view/write_text_file`, multimodal tools

### 2.7 Pipeline / Orchestration

- **MsgHub**: Broadcast messages giữa group agents (pub/sub pattern). Agents auto-subscribe khi enter hub, auto-unsubscribe khi exit
- **SequentialPipeline**: Chain agents tuần tự A → B → C
- **FanoutPipeline**: Fan-out cùng 1 input cho nhiều agents (parallel/sequential)
- **ChatRoom**: Real-time multi-agent coordination (WebSocket-based)

### 2.8 RAG Module

- **Readers**: Text, PDF, Word, Excel, PowerPoint, Image
- **Vector Stores**: Qdrant, Milvus, MongoDB, OceanBase, AliMySQL
- **SimpleKnowledge**: Wrapper kết hợp reader + embedding + store
- Tích hợp trực tiếp vào ReActAgent qua `retrieve_knowledge()` tool
- Query rewriting tự động để improve retrieval quality

### 2.9 A2A (Agent-to-Agent Protocol)

- Hỗ trợ chuẩn A2A protocol cho inter-agent communication
- Agent card resolver: File, Well-known URL, Nacos service discovery
- Cho phép agents từ different systems giao tiếp

### 2.10 Evaluation & Tuner

- **Evaluation**: Metric system, Benchmark framework, Ray-based distributed evaluation
- **Tuner**: RL via Trinity-RFT, prompt tuning via DSPy, model selection

### 2.11 Observability

- **OpenTelemetry** integration: `@trace_reply`, `@trace_llm`, `@trace_toolkit`
- Backend: Arize-Phoenix, Langfuse, AgentScope Studio
- Token usage tracking trong mỗi ChatResponse

---

## 3. Điểm mạnh cho tự động hoá doanh nghiệp

### S1. Kiến trúc multi-agent linh hoạt
- `MsgHub` + `Pipeline` cho phép xây dựng bất kỳ workflow nào: sequential (approval chain), parallel (multi-department analysis), debate (decision making)
- Ví dụ: Task từ sales → marketing agent review → manager agent approve → execution agent thực thi

### S2. Tool integration rất mạnh
- **MCP support** = kết nối hàng ngàn external tools (Google, Slack, database, APIs...)
- **Agent Skills** = đóng gói domain knowledge thành reusable packages
- **Middleware** = thêm logging, audit trail, permission check vào mọi tool call

### S3. Memory management tinh vi
- **Short-term**: Giữ context conversation, auto-compression khi dài
- **Long-term**: Nhớ thông tin qua sessions (customer preferences, project history)
- **Mark system**: Tag messages theo mục đích (internal hint vs. user-facing)

### S4. RAG tích hợp sẵn
- Đọc được PDF, Word, Excel, PowerPoint → vector search → inject context
- Phù hợp cho: knowledge management, policy compliance, internal wiki agents

### S5. Production-ready infrastructure
- **OpenTelemetry tracing**: Monitor agent behavior, debug production issues
- **Session persistence**: JSONSession, Redis, SQLAlchemy → restart-safe
- **State serialization**: `state_dict()`/`load_state_dict()` pattern cho checkpointing
- **Deployment**: Quart/FastAPI + SSE streaming cho web integration

### S6. Human-in-the-loop design
- `UserAgent` cho interactive workflows
- **Realtime steering**: Interrupt agent mid-execution, resume seamlessly
- Hook system cho approval gates (pre_acting hook = require human approval before tool execution)

### S7. Structured output via Pydantic
- Agent trả về typed data (not just text) → dễ integrate vào business systems
- Ví dụ: Agent phân tích email → trả `OrderRequest(product, quantity, price)` → feed vào ERP

### S8. Model-agnostic
- 6 model adapters → không bị lock-in vào 1 provider
- Có thể dùng Ollama (local) cho sensitive data, OpenAI/Claude cho complex tasks

### S9. Agentic RL / Fine-tuning
- Tune agent behavior bằng RL (Trinity-RFT) hoặc prompt optimization (DSPy)
- Train agent trên historical data của công ty → improve performance over time

---

## 4. Điểm yếu / Rủi ro cho tự động hoá doanh nghiệp

### W1. Complexity cao - learning curve dốc
- ReActAgent đơn lẻ ~1138 dòng với rất nhiều config options
- Toolkit class có TODO comment chính team thấy cần split
- **Rủi ro**: Team nhỏ mất nhiều thời gian để hiểu đủ sâu để customize

### W2. Async-only design
- Toàn bộ codebase là `async/await` - không có sync API fallback
- **Rủi ro**: Nếu hệ thống hiện tại là sync (Django, Flask cũ...) → migration effort lớn

### W3. Alibaba ecosystem bias
- DashScope là first-class citizen trong examples
- Một số tính năng chỉ hoạt động với Alibaba cloud (Tablestore, Nacos)
- **Rủi ro**: Nếu công ty không dùng Alibaba Cloud → một số features bị hạn chế

### W4. Thiếu built-in security/permission model
- Không có RBAC cho agents
- `execute_python_code`, `execute_shell_command` không có sandbox rõ ràng → **lỗ hổng bảo mật nghiêm trọng** trong production
- **Rủi ro**: Agent có thể execute arbitrary code nếu không cẩn thận

### W5. Thiếu built-in workflow engine
- Không có persistent workflow state machine (như Temporal, Airflow)
- Pipeline/MsgHub là in-memory → mất state khi crash
- **Rủi ro**: Long-running business processes (multi-day approval flows) cần external orchestrator

### W6. Error handling / Retry logic hạn chế
- Không có built-in retry với exponential backoff cho model API calls
- Không có circuit breaker pattern cho external tool calls
- **Rủi ro**: Production reliability phụ thuộc vào developer tự implement

### W7. Testing infrastructure chưa mature cho enterprise
- Tests chủ yếu là unit tests với mock objects
- Không có integration test framework cho multi-agent workflows
- **Rủi ro**: Khó đảm bảo reliability ở scale enterprise

### W8. Thiếu cost tracking
- OpenTelemetry tốt nhưng không có cost aggregation
- Không có budget/quota management cho agents
- **Rủi ro**: Chi phí LLM API có thể vượt kiểm soát khi scale

### W9. API instability
- "AgentScope 2.0 is on the way" (April 2026) → breaking changes sắp tới
- **Rủi ro**: Investment vào current API có thể cần rework

---

## 5. Đánh giá tổng thể

| Tiêu chí | Đánh giá | Ghi chú |
|---|---|---|
| Multi-agent orchestration | Tốt | MsgHub + Pipeline đủ linh hoạt |
| Tool/API integration | Rất tốt | MCP + Toolkit + Skills |
| Memory/Context | Tốt | Short + Long-term, compression |
| RAG/Knowledge | Tốt | Multi-format readers, multi VDB |
| Production readiness | Trung bình | Có tracing, thiếu security + workflow persistence |
| Enterprise security | Yếu | Thiếu RBAC, sandbox, audit |
| Scalability | Trung bình | Redis/DB support, thiếu distributed workflow |
| Cost management | Yếu | Không có budget/quota control |
| Ease of adoption | Trung bình | Async-only, complex API surface |

**Kết luận**: AgentScope mạnh về technical capabilities (tool integration, memory, RAG, multi-agent) nhưng chưa sẵn sàng cho enterprise production ở khía cạnh security, workflow persistence, và cost management.

Phù hợp nhất cho:
- **Prototyping** các use case tự động hoá
- **Internal tools** nơi security requirements thấp hơn
- **Research/experimentation** trước khi commit vào production

Nếu triển khai production, cần bổ sung:
1. RBAC layer cho agents
2. Code execution sandbox
3. Persistent workflow engine (Temporal/Celery)
4. Cost tracking/budgeting
5. Comprehensive audit logging
