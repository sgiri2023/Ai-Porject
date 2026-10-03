# Python Agent Service Architecture

## Overview

The proposed architecture uses a single Python Agent Service containing multiple agent modules. The first module is **Document Classification**, with additional agents added later without creating a separate microservice for each agent.

The service exposes:

- **FastAPI** for synchronous HTTP requests
- **Queue Worker** for asynchronous/background processing
- **Agent Router** for dispatching requests to the correct agent
- Shared integrations for Azure Document Intelligence, OpenAI, Langfuse, Blob Storage, and the Main Node.js Backend

The Main Node.js Backend remains the system of record for application and business data.

---

# Final Architecture

```text
                         ┌──────────────────┐
                         │      React       │
                         └────────┬─────────┘
                                  │
                                  ▼
                         ┌──────────────────┐
                         │  Main Backend    │
                         │     Node.js      │
                         │                  │
                         │ Auth             │
                         │ Documents        │
                         │ Business Rules   │
                         │ PostgreSQL       │
                         └───────┬─────┬────┘
                                 │     │
                         HTTP    │     │ Queue
                                 │     ▼
                                 │  Azure Service Bus
                                 │     │
                                 ▼     ▼
                       ┌─────────────────────────┐
                       │     Python Agent        │
                       │        Service          │
                       │                         │
                       │ ┌─────────────────────┐ │
                       │ │    FastAPI API      │ │
                       │ └──────────┬──────────┘ │
                       │            │            │
                       │ ┌──────────▼──────────┐ │
                       │ │    Agent Router     │ │
                       │ └──────────┬──────────┘ │
                       │            │            │
                       │     ┌──────┴──────┐     │
                       │     ▼             ▼     │
                       │ Document       Future  │
                       │ Classification Agents │
                       │     │                   │
                       │     └─────────┐         │
                       │               ▼         │
                       │       Shared Integrations│
                       │       ┌──────────────┐  │
                       │       │ Azure DI     │  │
                       │       │ OpenAI       │  │
                       │       │ Langfuse     │  │
                       │       │ Blob         │  │
                       │       │ Main Backend │  │
                       │       └──────────────┘  │
                       └─────────────────────────┘
```

---

# Project Structure

```text
agent-service/
│
├── app/
│   │
│   ├── main.py
│   │
│   ├── api/
│   │   ├── __init__.py
│   │   ├── dependencies.py
│   │   │
│   │   └── routes/
│   │       ├── __init__.py
│   │       ├── health.py
│   │       └── agents.py
│   │
│   ├── agents/
│   │   ├── __init__.py
│   │   │
│   │   ├── base/
│   │   │   ├── __init__.py
│   │   │   ├── agent.py
│   │   │   └── result.py
│   │   │
│   │   ├── router/
│   │   │   ├── __init__.py
│   │   │   └── agent_router.py
│   │   │
│   │   └── document_classification/
│   │       ├── __init__.py
│   │       ├── agent.py
│   │       ├── classifier.py
│   │       ├── rules.py
│   │       ├── prompts.py
│   │       └── models.py
│   │
│   ├── integrations/
│   │   ├── __init__.py
│   │   │
│   │   ├── main_backend/
│   │   │   ├── __init__.py
│   │   │   ├── client.py
│   │   │   ├── documents.py
│   │   │   ├── teams.py
│   │   │   └── configuration.py
│   │   │
│   │   ├── azure/
│   │   │   ├── __init__.py
│   │   │   └── document_intelligence.py
│   │   │
│   │   ├── openai/
│   │   │   ├── __init__.py
│   │   │   └── client.py
│   │   │
│   │   ├── storage/
│   │   │   ├── __init__.py
│   │   │   └── blob_storage.py
│   │   │
│   │   └── langfuse/
│   │       ├── __init__.py
│   │       └── client.py
│   │
│   ├── queue/
│   │   ├── __init__.py
│   │   ├── consumer.py
│   │   ├── messages.py
│   │   └── publisher.py
│   │
│   ├── config/
│   │   ├── __init__.py
│   │   └── settings.py
│   │
│   └── utils/
│       ├── __init__.py
│       ├── logging.py
│       └── exceptions.py
│
├── tests/
│   ├── agents/
│   │   └── document_classification/
│   │       ├── test_agent.py
│   │       ├── test_rules.py
│   │       └── test_classifier.py
│   │
│   ├── integrations/
│   │   ├── test_main_backend.py
│   │   └── test_document_intelligence.py
│   │
│   └── api/
│       └── test_agents.py
│
├── .env
├── .env.example
├── .gitignore
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
└── README.md
```

---

# Main Components

## 1. FastAPI Application

`app/main.py` is the HTTP entry point.

```text
main.py
   │
   └── FastAPI
          │
          └── /api/v1/agents/...
```

FastAPI should only handle HTTP concerns such as:

- Request validation
- Authentication/dependencies
- Calling the Agent Router
- Returning responses

It should **not contain agent/business logic**.

---

# 2. Agent Router

Location:

```text
app/agents/router/agent_router.py
```

The Agent Router is responsible for selecting the correct agent.

Example:

```python
class AgentRouter:

    def __init__(self, agents):
        self.agents = agents

    async def execute(self, agent_name, request):

        agent = self.agents.get(agent_name)

        if not agent:
            raise ValueError(
                f"Unknown agent: {agent_name}"
            )

        return await agent.run(request)
```

Initially:

```text
AgentRouter
    │
    └── document_classification
```

Later:

```text
AgentRouter
    │
    ├── document_classification
    ├── tax_extraction
    ├── document_summary
    ├── document_comparison
    └── tax_analysis
```

This makes the service extensible without creating a new service for every agent.

---

# 3. Base Agent

A common base interface keeps all agents consistent.

Example:

```python
from abc import ABC, abstractmethod


class BaseAgent(ABC):

    @abstractmethod
    async def run(self, request):
        pass
```

Each agent implements the same basic contract:

```text
request
   ↓
agent.run()
   ↓
result
```

---

# 4. Document Classification Agent

Location:

```text
app/agents/document_classification/
```

Files:

```text
agent.py
classifier.py
rules.py
prompts.py
models.py
```

The first agent identifies what type of document has been uploaded.

For the current tax workflow, the main document classifications include:

- Tax Return
- Tax Computation

The agent can use deterministic checks first and AI/document-intelligence capabilities when required.

Conceptually:

```text
Receive document
      ↓
Get required configuration
      ↓
Extract document information
      ↓
Run deterministic classification
      ↓
Confidence sufficient?
      │
    YES ──────→ Result
      │
     NO
      ↓
AI / Document Intelligence fallback
      ↓
Result
```

The classification should happen **when the document is uploaded**, before the later comparison/process execution, so the UI can show the detected document type.

---

# 5. Shared Integrations

Integrations are kept outside individual agents so they can be reused by future agents.

## Azure Document Intelligence

```text
app/integrations/azure/document_intelligence.py
```

Used for document analysis and extraction.

Future agents can reuse the same integration.

---

## OpenAI

```text
app/integrations/openai/client.py
```

Provides a common interface for LLM calls.

It can be used for:

- Classification fallback
- Extraction
- Reasoning
- Comparison
- Summarization
- Future agent workflows

---

## Langfuse

```text
app/integrations/langfuse/client.py
```

Used for AI observability, tracing, prompt/LLM monitoring, and debugging.

The integration should be shared rather than implemented separately inside every agent.

---

## Blob Storage

```text
app/integrations/storage/blob_storage.py
```

Responsible for retrieving documents from Azure Blob Storage when the agent needs the actual document content.

---

# 6. Main Backend Integration

Location:

```text
app/integrations/main_backend/
```

The Python Agent Service needs to communicate with the existing Node.js Main Backend.

Possible client methods:

```python
backend.get_document(...)
backend.get_document_types(...)
backend.get_rules(...)
backend.get_configuration(...)
backend.save_classification(...)
backend.save_result(...)
```

The exact methods depend on the Main Backend API.

The important architectural principle is:

```text
Python Agent Service
        │
        ▼
MainBackendClient
        │
        ▼
Node.js Main Backend
        │
        ▼
PostgreSQL
```

The Python service should not directly own or modify the Main Backend's database.

The Node.js Backend remains the system of record for application and business data.

---

# 7. Queue Processing

For long-running or background operations, use Azure Service Bus.

```text
Node
 │
 ▼
Azure Service Bus
 │
 ▼
Python Queue Worker
 │
 ▼
Agent Router
 │
 ▼
Correct Agent
```

The worker is located under:

```text
app/queue/
```

with:

```text
consumer.py
messages.py
publisher.py
```

The important design principle is that the queue worker should use the **same Agent Router and agents** as the HTTP API.

There should not be separate implementations such as:

```text
HTTP classification logic
Queue classification logic
```

Instead:

```text
DocumentClassificationAgent
```

is the actual business/agent implementation.

Both HTTP and queue processing invoke it.

---

# 8. Immediate Document Classification Flow

Document classification needs to happen at upload time.

Example:

```text
React
  │
  │ Upload PDF
  ▼
Node Main Backend
  │
  │ Request classification
  ▼
FastAPI
  │
  ▼
Agent Router
  │
  ▼
Document Classification Agent
  │
  ├── Main Backend
  ├── Blob Storage
  ├── Azure Document Intelligence
  ├── Deterministic Rules
  └── OpenAI fallback
  │
  ▼
Classification Result
  │
  ▼
Node Main Backend
  │
  ▼
React
```

The UI can then show something such as:

```text
Document Type: Tax Return
Confidence: 96%
```

or:

```text
Document Type: Tax Computation
Confidence: 94%
```

The exact confidence representation should be defined according to the classification implementation.

---

# 9. Background Agent Flow

Long-running tasks should be handled asynchronously.

```text
React
  │
  ▼
Node Main Backend
  │
  ▼
Azure Service Bus
  │
  ▼
Python Agent Worker
  │
  ▼
Agent Router
  │
  ▼
Selected Agent
  │
  ├── Main Backend
  ├── Blob Storage
  ├── Azure Document Intelligence
  ├── OpenAI
  └── Langfuse
  │
  ▼
Result
  │
  ▼
Main Backend
```

This avoids keeping an HTTP request open for expensive processing.

---

# 10. Deployment

Initially, deploy the same Python codebase as two containers:

```text
                    agent-service
                         │
             ┌───────────┴───────────┐
             │                       │
             ▼                       ▼
      agent-api                  agent-worker
      FastAPI                    Queue Consumer
```

## API Container

Example:

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

## Worker Container

Example:

```bash
python -m app.queue.consumer
```

Both containers use the same application code, agents, and integrations.

They can later be scaled independently.

Example:

```text
agent-api
  × 3

agent-worker
  × 5
```

This allows HTTP traffic and background workloads to scale independently.

---

# 11. Why We Do Not Need Azure Functions

With this architecture, separate Azure Functions such as:

```text
Dispatcher Function
Performer Function
Function per Agent
```

are not required.

Instead:

```text
FastAPI
   = HTTP entry point

Queue Worker
   = Async entry point

Agent Router
   = Dispatcher

Agents
   = Performers
```

This keeps the architecture simpler and keeps agent orchestration inside the Python service.

Azure Service Bus still provides the messaging layer for asynchronous work.

---

# 12. Responsibility Matrix

| Component | Responsibility |
|---|---|
| React | UI and document upload |
| Node.js | Main application and business logic |
| PostgreSQL | Application/business data |
| Azure Blob Storage | Original documents |
| Azure Service Bus | Async messaging |
| FastAPI | Agent HTTP API |
| Queue Worker | Async agent execution |
| Agent Router | Selects the correct agent |
| Classification Agent | Classifies documents |
| Azure Document Intelligence | Document analysis/extraction |
| OpenAI | AI reasoning/fallback |
| Langfuse | AI observability |
| Main Backend Client | Python → Node communication |

---

# 13. Tax AI Product Context

The Agent Service supports the broader tax-document comparison workflow.

A typical process can involve:

```text
Previous Year PDF
        +
Current Year PDF
        ↓
Document Classification
        ↓
Tax Return / Tax Computation
        ↓
Relevant Data Extraction
        ↓
Datapoint Mapping
        ↓
Previous vs Current Comparison
        ↓
Deterministic Rules
        ↓
AI Fallback Where Required
        ↓
Final Result
```

The uploaded document should be classified before the main process is executed.

This means the system can validate that the uploaded document is the expected type for the selected workflow.

---

# 14. Future Agents

The service should be designed to grow.

Potential future modules:

```text
app/agents/
│
├── document_classification/
├── tax_extraction/
├── document_summary/
├── document_comparison/
├── tax_analysis/
└── anomaly_detection/
```

All of these can use shared integrations:

```text
Azure Document Intelligence
OpenAI
Langfuse
Blob Storage
Main Backend
```

The Agent Router determines which agent executes a request.

---

# 15. Recommended Request Flow

## Synchronous

Use HTTP when the result is required immediately.

```text
Client
  ↓
Node
  ↓
FastAPI
  ↓
Agent Router
  ↓
Agent
  ↓
Result
  ↓
Node
  ↓
Client
```

Example:

```text
Upload document
       ↓
Classify document
       ↓
Return Tax Return / Tax Computation
```

---

## Asynchronous

Use the queue when processing can happen in the background.

```text
Node
  ↓
Service Bus
  ↓
Worker
  ↓
Agent Router
  ↓
Agent
  ↓
Save Result
```

Example:

```text
Start tax comparison process
       ↓
Queue
       ↓
Worker
       ↓
Extract datapoints
       ↓
Run rules
       ↓
AI fallback
       ↓
Save process result
```

---

# 16. Key Architectural Principles

## Principle 1 — One Agent Service

Do not create a microservice for every agent initially.

Use:

```text
Python Agent Service
    ├── Agent Router
    ├── Classification Agent
    ├── Future Agent
    └── Shared Integrations
```

---

## Principle 2 — Separate Entry Points from Agent Logic

FastAPI and Queue Worker are entry points.

The agent contains the actual processing logic.

```text
HTTP ───────┐
            ├──> Agent Router ──> Agent
Queue ──────┘
```

---

## Principle 3 — Shared Integrations

Do not duplicate clients.

Use:

```text
integrations/
    ├── azure/
    ├── openai/
    ├── langfuse/
    ├── storage/
    └── main_backend/
```

All agents can reuse them.

---

## Principle 4 — Main Backend Owns Business Data

The Node.js Main Backend remains responsible for:

- Users
- Teams
- Documents
- Document types
- Rules
- Configuration
- Process data
- Results

The Agent Service performs AI/agent workloads and communicates with the Main Backend through APIs.

---

## Principle 5 — Classification at Upload Time

Document classification is not postponed until the full process execution.

The upload flow should determine the document type immediately so the UI and subsequent workflow know whether the uploaded document is:

```text
Tax Return
```

or

```text
Tax Computation
```

---

# 17. Final Target Architecture

```text
                         ┌──────────────┐
                         │    React     │
                         └──────┬───────┘
                                │
                                ▼
                    ┌─────────────────────┐
                    │   Node.js Backend   │
                    │                     │
                    │ Business Logic      │
                    │ Auth                │
                    │ PostgreSQL          │
                    │ Document Metadata   │
                    └───────┬───────┬─────┘
                            │       │
                     HTTP   │       │ Queue
                            │       ▼
                            │   Azure Service Bus
                            │       │
                            ▼       ▼
                    ┌────────────────────────┐
                    │   Python Agent Service  │
                    │                        │
                    │ ┌────────────────────┐ │
                    │ │      FastAPI       │ │
                    │ └─────────┬──────────┘ │
                    │           │            │
                    │ ┌─────────▼──────────┐ │
                    │ │    Agent Router    │ │
                    │ └─────────┬──────────┘ │
                    │           │            │
                    │ ┌─────────▼──────────┐ │
                    │ │ Document           │ │
                    │ │ Classification     │ │
                    │ │ Agent              │ │
                    │ └─────────┬──────────┘ │
                    │           │            │
                    │ ┌─────────▼──────────┐ │
                    │ │ Shared Integrations│ │
                    │ │                    │ │
                    │ │ Azure DI            │ │
                    │ │ OpenAI              │ │
                    │ │ Langfuse            │ │
                    │ │ Blob Storage        │ │
                    │ │ Main Backend Client │ │
                    │ └────────────────────┘ │
                    │                        │
                    └────────────────────────┘
```

---

# Conclusion

The recommended baseline is:

```text
                 Python Agent Service
                         │
          ┌──────────────┼──────────────┐
          │              │              │
       FastAPI        Queue Worker   Shared Integrations
          │              │              │
          └───────┬──────┘              │
                  ▼                     │
             Agent Router ◄─────────────┘
                  │
                  ▼
       Document Classification
                  │
                  ▼
           Future Agents
```

The first implementation should focus on:

1. FastAPI setup
2. Agent base interface
3. Agent Router
4. Document Classification Agent
5. Main Backend client
6. Azure Document Intelligence integration
7. OpenAI integration
8. Blob Storage integration
9. Langfuse integration
10. Azure Service Bus consumer
11. Unit/integration tests
12. Docker deployment

This provides a clean foundation for the tax-document classification and later tax comparison/extraction agents without introducing unnecessary Azure Functions or separate microservices.
