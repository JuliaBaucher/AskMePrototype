# AskMe Prototype

AskMe is a prototype AI chatbot designed to help operational teams investigate issues related to machine learning forecasting models faster and more consistently.

The chatbot acts as a **decision-support assistant**. It helps users understand why a forecasting issue occurred by combining:

- the relevant **SOP** for the selected ML model (retrieved deterministically from S3)
- the relevant **ML signals** for the selected warehouse and date (from DynamoDB)
- the most relevant **FAQ knowledge chunks** (retrieved via semantic similarity)
- the **user question**
- the **session context**

The system is designed to reduce investigation time, improve response quality, accelerate decision making, and increase trust in machine learning model outputs.

---

# 1. Purpose and Business Objective

In current operations, when forecasting anomalies occur, users often need to manually investigate multiple data sources and consult experts to understand the cause. This process is slow, expert availability is limited, and the delay reduces trust in machine learning models.

AskMe addresses this problem by providing an AI assistant that can:

- interpret model signals
- classify forecasting issues according to predefined SOPs
- explain the result in a way adapted to the user's role

The target outcomes are:

- faster understanding of why a forecasting issue occurred
- identification of the root cause according to the official SOP
- retrieval of relevant signals used in the forecast
- explanations adapted to the user's role and technical level
- lower time spent investigating anomalies
- increased trust in ML models

---

# 2. Scope and Supported Models

AskMe only supports questions related to these ML models:

- **Absence forecast**
- **Volume forecast**

Each model has one corresponding SOP.

If the question is outside this scope, AskMe must respond that the request is outside the supported scope.

---

# 3. Target Architecture

## 3.1 High-Level Architecture

    [User Browser]
             |
         v
    [API Gateway HTTP API]
         |
         v
    [Lambda: askme-backend]
         |------------------------------|
         |                              |
         v                              v
    [DynamoDB: ML Signals]         [S3 Knowledge Base]
    [DynamoDB: Sessions]           [SOP + FAQ JSON + embeddings]
    [DynamoDB: Feedback]
    [DynamoDB: Escalations]
         |
         v
    [Amazon Bedrock Titan Embeddings]
         |
         v
    [Amazon Bedrock Converse API]
         |
         v
    [Response to UI]

### Other supporting services

- CloudWatch Logs
- CloudWatch custom metrics via EMF
- S3 bucket for SOP and FAQ knowledge storage

## 3.2 What Each Component Does

### S3 Static Web UI

Hosts a simple HTML/JS chatbot page.

It collects the following user inputs:

- user role
- model type
- warehouse
- date
- natural language question

### API Gateway HTTP API

Exposes the backend endpoints:

- `POST /ask`
- `POST /feedback`
- `POST /escalate`

### Lambda Backend

The Lambda function is the orchestration layer of the application. It:

- validates scope
- retrieves session memory (last 3–5 turns)
- retrieves the correct SOP deterministically from S3 using model_type
- retrieves relevant FAQ chunks using embedding similarity
- retrieves ML signals from DynamoDB
- builds the LLM prompt
- calls Amazon Bedrock Converse
- stores the session turn
- emits logs and metrics

### S3 Knowledge Base

Used as the only knowledge store.

It:

- stores SOP documents (one per model) as JSON with embeddings
- stores FAQ chunks as JSON with embeddings
- enables deterministic retrieval for SOP
- enables semantic retrieval for FAQ using cosine similarity

### DynamoDB

Used to store operational and conversational data.

Tables:

- `AskMeSignals`: ML signals by model, site, and date
- `AskMeSessions`: session-based conversation memory
- `AskMeFeedback`: like/dislike records
- `AskMeEscalations`: escalation records for human follow-up

### Amazon Bedrock

Used for both embeddings and reasoning.

- **Titan embeddings model**: embeds FAQ chunks and user queries
- **Converse API**: performs classification and generates grounded responses

---

# 4. Core Design Decision: Who Performs Classification

This is the most important behavior rule in the system.

## Correct Behavior

The **LLM performs the classification**, but **only after retrieval**.

The LLM must classify using:

- the retrieved SOP for the selected model
- the retrieved ML signals for the selected warehouse and date
- the retrieved FAQ knowledge (for grounding and context)
- the current user question
- the session context (last 3–5 turns only)

The LLM must **not** classify the issue from the user question alone.

It must:

- follow `classification_policy.rule_priority_order`
- evaluate rules sequentially
- select the root cause mapped to the **first matching rule**
- fall back to `default_root_cause` if signals are insufficient or contradictory

It must also explain:

- which SOP rule matched
- which ML signals were used
- why the explanation fits the user role

## 4.1 Backend Responsibilities vs LLM Responsibilities

### Backend Responsibilities

The backend is responsible for:

- validating input
- validating whether the model type is in scope
- retrieving the SOP (deterministic S3 retrieval)
- retrieving FAQ chunks (embedding similarity)
- retrieving ML signals
- retrieving session history (bounded memory)
- building the prompt
- storing logs, feedback, escalation, and memory

### LLM Responsibilities

The LLM is responsible for:

- interpreting the retrieved SOP
- applying SOP rule order
- determining the root cause
- explaining reasoning in role-adapted language

---

# 5. Functional Flow

## 5.1 User Interaction Flow

The user interacts with the chatbot through a web interface.

The user provides:

- user role
- ML model type
- warehouse name
- date
- natural language question

The chatbot executes the following process:

1. Receive the user input from the UI
2. Validate whether the question is relevant to supported ML model issues
3. Retrieve the SOP deterministically based on model_type
4. Retrieve relevant FAQ chunks using semantic similarity
5. Retrieve ML signals for the selected warehouse and date
6. Retrieve recent session context (last 3–5 turns)
7. Combine SOP + FAQ + signals + question + session context
8. Apply SOP classification rules
9. Determine the root cause
10. Generate a role-adapted response
11. Return the response to the UI

## 5.2 End-to-End Runtime Request Flow

1. User opens the web UI
2. User enters role, model type, warehouse, date, and question
3. Browser calls `POST /ask`
4. Lambda validates input
5. Lambda loads recent session turns
6. Lambda retrieves SOP from S3 (deterministic)
7. Lambda retrieves FAQ chunks using embeddings + cosine similarity
8. Lambda queries `AskMeSignals`
9. Lambda builds the prompt
10. Lambda calls Bedrock Converse
11. Lambda stores session turn
12. Lambda emits logs and metrics
13. UI displays response

---

# 6. Role-Adapted Response Design

## Operations Manager

- simple explanation
- focus on operational impact
- highlight root cause
- concise wording

## Forecasting Analyst

- detailed reasoning
- include classification logic
- reference retrieved signals
- reference SOP rule used
- emphasize traceability

---

# 7. Conversational Memory

AskMe supports follow-up questions within the same session.

- Only the **last 3–5 turns** are included in the prompt
- Older context is not used
- If missing context is detected, the assistant asks for clarification

Memory is stored in `AskMeSessions` DynamoDB table.

---

# 8. Escalation to Human

- Users can escalate questions
- Stored in `AskMeEscalations`
- No external integration (demo mode)
- Used for analytics and improvement

---

# 9. Feedback Collection

Users can provide:

- like
- dislike

Stored in `AskMeFeedback` and used for evaluation and improvement.

---

# 10. Traceability and Logging

The system logs:

- user query
- session ID
- selected role/model
- retrieved SOP
- retrieved FAQ chunks
- retrieved signals
- prompt inputs
- generated response
- timestamp

---

# 11. Metrics Tracked

- total responses
- responses per role
- escalation rate
- response latency
- positive feedback count
- negative feedback count

---

# 12. Explainability Requirement

All responses must reference:

- the SOP rule used
- the ML signals used

---

# 13. Data Model

## 13.1 DynamoDB Tables

### A. `AskMeSignals`

**Purpose:** store ML signals for a model, site, and date

**Primary key**

- `pk = MODEL#<MODEL_KEY>#SITE#<WAREHOUSE>`
- `sk = DATE#<YYYY-MM-DD>#SIGNAL#<SIGNAL_KEY>`

**Example**

    {
      "pk": "MODEL#ABSENCE_FORECAST#SITE#BCN8",
      "sk": "DATE#2026-04-10#SIGNAL#ABSENCE_RATE",
      "model_type": "Absence forecast",
      "warehouse": "BCN8",
      "date": "2026-04-10",
      "signal_name": "absence_rate",
      "signal_value": 0.142,
      "expected_value": 0.089,
      "deviation": 0.053,
      "signal_summary": "Observed absence rate significantly exceeds expected baseline."
    }

### B. `AskMeSessions`

**Purpose:** session memory

**Primary key**

- `session_id`
- `turn_id`

**Example**

    {
      "session_id": "sess_123",
      "turn_id": "2026-03-14T16:35:01.999Z",
      "role": "Operations Manager",
      "model_type": "Absence forecast",
      "warehouse": "BCN8",
      "date": "2026-04-10",
      "user_question": "Why is the absence forecast off?",
      "assistant_response": "The likely root cause is School holiday impact...",
      "root_cause_code": "ABS_RC2",
      "rule_id": "R1",
      "created_at": "2026-03-14T16:35:01.999Z",
      "ttl": 1773515701
    }

`DynamoDB TTL` is a good fit for expiring old session memory because expired items can be deleted automatically after their expiration timestamp without consuming write throughput.

### C. `AskMeFeedback`

    {
      "response_id": "resp_abc123",
      "session_id": "sess_123",
      "feedback_type": "like",
      "timestamp": "2026-03-14T16:40:00Z"
    }

### D. `AskMeEscalations`

    {
      "escalation_id": "esc_001",
      "session_id": "sess_123",
      "user_question": "Why did BCN8 absence spike?",
      "chatbot_response": "The likely root cause is School holiday impact...",
      "user_role": "Operations Manager",
      "model_type": "Absence forecast",
      "warehouse": "BCN8",
      "date": "2026-04-10",
      "timestamp": "2026-03-14T16:42:00Z"
    }

---


# 14. Knowledge Storage and Retrieval Design

## 14.1 Design Overview

The AskMe prototype uses a **lightweight, fully serverless retrieval architecture based on S3** instead of a traditional vector database.

The system combines three sources of information:

- **SOP (deterministic retrieval)** → retrieved using model_type
- **FAQ (semantic retrieval)** → retrieved using embeddings + cosine similarity
- **ML signals (structured retrieval)** → retrieved from DynamoDB

This hybrid approach ensures:

- strong **determinism for critical business logic (SOP)**
- flexible **semantic search for contextual knowledge (FAQ)**
- full **traceability and auditability**

---

## 14.2 Why S3 + Embeddings Instead of Vector Database

In this prototype, S3 is used as the **only knowledge store**, with embeddings stored directly inside JSON files.

### Why this works well for AskMe

- The knowledge base is **small and controlled**:
  - 2 SOP documents
  - small FAQ (≤ 10 pages)
- Retrieval does not require:
  - large-scale indexing
  - ANN (Approximate Nearest Neighbor)
  - distributed search

### Advantages of S3-based retrieval

- **Simplicity**
  - no cluster to manage
  - no indexing pipeline complexity
- **Cost efficiency**
  - S3 storage is extremely cheap
  - no always-on infrastructure (unlike OpenSearch)
- **Full control**
  - deterministic SOP retrieval
  - explicit metadata filtering
- **Auditability**
  - documents are versioned and directly readable
- **Flexibility**
  - easy to update SOP or FAQ without reindexing pipelines

### Trade-offs vs Vector Database

| Aspect | S3 + Embeddings | Vector DB (OpenSearch, Pinecone) |
|------|----------------|----------------------------------|
| Cost | Very low | Higher (compute + storage) |
| Complexity | Very low | Medium to high |
| Scalability | Limited (brute force) | High (ANN indexing) |
| Latency | Acceptable for small data | Optimized for large scale |
| Use case fit | Small, controlled KB | Large, dynamic KB |

### Design decision

S3 + embeddings is the **best fit for this prototype** because:

- knowledge size is small
- retrieval must be **transparent and controllable**
- cost and simplicity are prioritized over scalability

---

## 14.3 SOP Retrieval Strategy (Deterministic)

Unlike FAQ, SOP retrieval is **not semantic**.

### Why deterministic retrieval is used

- The user **explicitly selects the model_type**
- There is **exactly one SOP per model**
- The SOP defines **formal classification rules**

### Implementation

- SOP is stored in S3 as a single JSON document per model
- Retrieval is done using **metadata (model_type)**

Example:

    absence_forecast_sop.json
    volume_forecast_sop.json

### Benefits

- **No ambiguity**
- **No risk of retrieving wrong document**
- **Full alignment with business rules**

---

## 14.4 FAQ Retrieval Strategy (Semantic)

FAQ is retrieved using:

- Titan embeddings
- cosine similarity
- brute-force search

### Why semantic retrieval is used for FAQ

- FAQ contains:
  - product explanations
  - ML concepts
  - system behavior explanations
- User questions are **free-form**
- Exact keyword match is not sufficient

### Retrieval steps

1. Embed user question
2. Load FAQ chunks from S3
3. Compute cosine similarity
4. Select top-K chunks

### Example FAQ chunk

    {
      "chunk_id": "faq_001",
      "doc_type": "FAQ",
      "scope_type": "general",
      "model_type": null,
      "title": "What does AskMe do?",
      "content": "AskMe is an AI assistant that helps explain forecasting issues using SOP rules and ML signals.",
      "embedding": [ ... ]
    }

---

## 14.5 Combined Retrieval Pattern

For every request, the system ALWAYS retrieves:

- SOP (deterministic)
- FAQ (semantic)
- signals (structured)

There is:

- no routing
- no conditional retrieval
- no LLM-based retrieval decisions

This ensures:

- consistent behavior
- explainability
- no hidden logic

---

# 15. Classification Design: LLM vs Deterministic Logic

## 15.1 Why Not Use Deterministic Code for Classification

An alternative would be to implement SOP rules in Python code.

Example:

    if school_holiday_flag == True:
        return ABS_RC2

### Limitations of deterministic logic

- Hard to maintain as rules evolve
- Difficult to:
  - interpret complex conditions
  - handle missing or noisy signals
- No natural language explanation
- No flexibility for edge cases

---

## 15.2 Why LLM-Based Classification Is Used

The LLM performs classification **after retrieval**, using SOP + signals.

### Key advantages

#### 1. Flexibility

- Handles:
  - partial signals
  - noisy data
  - ambiguous cases

#### 2. Natural reasoning

- Interprets:
  - conditions
  - context
  - relationships between signals

#### 3. Explainability

- Generates:
  - human-readable explanation
  - references to SOP and signals

#### 4. Maintainability

- No need to rewrite code when SOP changes
- Just update SOP JSON

---

## 15.3 Trade-off: LLM vs Deterministic Logic

| Aspect | Deterministic Rules | LLM Classification |
|------|--------------------|-------------------|
| Control | Very high | High (prompt-controlled) |
| Flexibility | Low | High |
| Explainability | Low | High |
| Maintenance | Hard-coded | Data-driven (SOP) |
| Robustness to missing data | Low | High |

### Design decision

The system uses:

- **LLM for classification**
- **SOP as the source of truth**

This provides:

- flexibility
- explainability
- alignment with business rules

---

# 16. Prompt Design

Prompt design enforces strict behavior:

- use retrieved evidence only
- follow SOP rule order
- return structured JSON
- adapt to user role
- avoid hallucinations

## 16.1 System Prompt Template

    You are AskMe, an AI decision-support assistant for forecasting issue investigation.

    You only support these ML models:
    - Absence forecast
    - Volume forecast

    Your task is to classify the forecasting issue based on:
    1. the retrieved SOP document
    2. the retrieved ML signals
    3. the user question
    4. the session context

    You must NOT classify using the user question alone.

    Classification rules:
    - Read the SOP classification_policy.
    - Follow rule_priority_order exactly in sequence.
    - Evaluate each rule against the retrieved ML signals.
    - Select the root cause associated with the first matching rule.
    - If signals are missing, contradictory, or insufficient, use default_root_cause.

    Output requirements:
    - State whether the question is in scope.
    - If in scope, provide:
      - root cause code
      - root cause name
      - SOP rule used
      - ML signals used
      - explanation adapted to the user's role
    - Always reference the SOP rule and the ML signals used.
    - If out of scope, say the request is outside AskMe scope.
    - Do not invent signals that were not retrieved.
    - Do not invent SOP rules that were not retrieved.

---

# 17. Example Operating Principle

1. Retrieve SOP (deterministic)
2. Retrieve FAQ (semantic)
3. Retrieve signals
4. Build prompt
5. LLM applies SOP rules
6. Select first matching root cause
7. Generate explanation
8. Store and return response

---

# 18. Assumptions for This Prototype

- AWS Region is `eu-west-1`
- frontend is static (GitHub Pages or S3)
- backend is a single Lambda
- no authentication
- only 2 models
- 1 SOP per model
- small FAQ
- brute-force retrieval
- no vector DB
- session memory is short-term
- escalation is demo-only

---

# 19. Repository Contents

- frontend UI
- Lambda backend
- SOP JSON
- FAQ JSON
- deployment scripts
- configuration
- this README

---

# 20. Summary

AskMe is a serverless, retrieval-grounded AI assistant.

Key design principles:

- **deterministic SOP retrieval**
- **semantic FAQ retrieval**
- **LLM-based classification**
- **no vector database**
- **bounded memory**
- **traceability and auditability**
- **low-cost, simple architecture**

This design demonstrates how to build a **controlled, explainable AI system** that combines structured rules with LLM reasoning.
