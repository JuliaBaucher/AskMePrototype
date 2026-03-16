# AskMe Prototype

AskMe is a prototype AI chatbot designed to help operational teams investigate issues related to machine learning forecasting models faster and more consistently.

The chatbot acts as a **decision-support assistant**. It helps users understand why a forecasting issue occurred by combining:

- the relevant **SOP** for the selected ML model
- the relevant **ML signals** for the selected warehouse and date
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
    [DynamoDB: ML Signals]         [OpenSearch Serverless]
    [DynamoDB: Sessions]           [SOP vector index]
    [DynamoDB: Feedback]
    [DynamoDB: Escalations]
         |                              |
         |                              |
         v                              v
    [Amazon Bedrock Titan Embeddings] [retrieve SOP chunks]
         |
         v
    [Amazon Bedrock Converse API]
         |
         v
    [Response to UI]

### Other supporting services

- CloudWatch Logs
- CloudWatch custom metrics via EMF
- S3 bucket for raw SOP JSON documents

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
- retrieves session memory
- retrieves the right SOP from the vector database
- retrieves ML signals from DynamoDB
- builds the LLM prompt
- calls Amazon Bedrock Converse
- stores the session turn
- emits logs and metrics

### OpenSearch Serverless

Used as the vector database for SOP retrieval.

It:

- stores SOP embeddings
- returns the relevant SOP content for the selected model

### DynamoDB

Used to store operational and conversational data.

Tables:

- `AskMeSignals`: ML signals by model, site, and date
- `AskMeSessions`: session-based conversation memory
- `AskMeFeedback`: like/dislike records
- `AskMeEscalations`: escalation records for human follow-up

### Amazon Bedrock

Used for both embeddings and chat generation.

- **Titan embeddings model**: embeds SOP chunks and retrieval queries
- **Converse API**: classifies the issue using retrieved SOP + signals + user question + session context

---

# 4. Core Design Decision: Who Performs Classification

This is the most important behavior rule in the system.

## Correct Behavior

The **LLM performs the classification**, but **only after retrieval**.

The LLM must classify using:

- the retrieved SOP for the selected model
- the retrieved ML signals for the selected warehouse and date
- the current user question
- the session context

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
- retrieving the SOP
- retrieving the ML signals
- retrieving session history
- building the prompt
- storing logs, feedback, escalation, and memory

### LLM Responsibilities

The LLM is responsible for:

- interpreting the retrieved SOP
- applying SOP rule order
- determining the root cause
- explaining reasoning in role-adapted language

This separation keeps the architecture aligned with the business requirement: the response must be grounded in retrieved SOP and signals, not in user question alone.

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
3. Retrieve the appropriate SOP document for the selected ML model
4. Retrieve the relevant ML signal records for the selected warehouse and date
5. Combine retrieved SOP and ML signal data with the user question and session context
6. Apply SOP classification rules
7. Determine the root cause according to the rule priority order
8. Generate a response adapted to the user role
9. Return the response to the UI

Before generating an answer, the chatbot retrieves two categories of information:

- SOP documents defining classification rules
- ML signal records describing forecast context

If the question is outside supported scope, the chatbot responds that the request is outside its scope.

## 5.2 End-to-End Runtime Request Flow

1. User opens the web UI
2. User enters role, model type, warehouse, date, and question
3. Browser calls `POST /ask`
4. Lambda validates:
   - required fields are present
   - model is in the supported list
5. Lambda loads recent session turns from `AskMeSessions`
6. Lambda retrieves the SOP from OpenSearch using model-filtered vector search
7. Lambda queries `AskMeSignals` for all signals for the model, site, and date
8. Lambda builds a strict system prompt
9. Lambda calls Amazon Bedrock Converse
10. Lambda stores the session turn in `AskMeSessions`
11. Lambda emits structured logs and metrics
12. UI shows the answer with feedback and escalation options

---

# 6. Role-Adapted Response Design

AskMe must tailor explanations to the selected user role.

## Operations Manager

Response style:

- simple explanation
- focus on operational impact
- highlight root cause
- concise wording

## Forecasting Analyst

Response style:

- detailed reasoning
- include classification logic
- reference retrieved signals
- reference SOP rule used
- emphasize traceability

---

# 7. Conversational Memory

AskMe supports follow-up questions within the same session.

Examples include:

- comparisons between warehouses
- comparisons between dates
- clarifications about previous answers

Conversation memory is:

- session-based
- stored in the backend
- retrieved before each new answer generation

The `AskMeSessions` DynamoDB table stores prior turns so that the assistant can use recent context when answering follow-up questions.

---

# 8. Escalation to Human

The UI allows users to escalate a question to a human expert.

For the prototype:

- escalation does **not** trigger external systems
- escalation records are stored
- escalation data is available for later analytics

Each escalation record contains:

- session ID
- user question
- chatbot response
- user role
- model type
- warehouse
- date
- timestamp

---

# 9. Feedback Collection

Users can provide feedback on each response:

- `like`
- `dislike`

Feedback must be linked to the specific response so it can be used in analytics and future system improvement.

---

# 10. Traceability and Logging

The system logs the following for each request:

- user query
- session ID
- selected role
- selected model
- warehouse
- date
- retrieved SOP
- retrieved signals
- prompt inputs
- generated response
- timestamp

This ensures the system is auditable and explainable.

---

# 11. Metrics Tracked

The system tracks the following operational metrics:

- total responses
- responses per role
- escalation rate
- response latency
- positive feedback count
- negative feedback count

These metrics are emitted through CloudWatch custom metrics via EMF.

---

# 12. Explainability Requirement

All responses must explicitly reference:

- the SOP rule used
- the ML signals used for classification

This is a core product requirement. The assistant should never give an ungrounded answer.

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

# 14. SOP Storage and Retrieval Design

## 14.1 Why Vector Search Is Used

OpenSearch Serverless vector collections are well suited for this use case because they are designed for modern similarity-search scenarios used in ML and GenAI applications.

They allow the system to retrieve relevant SOP content without managing search cluster infrastructure manually.

## 14.2 Simplified SOP Indexing Strategy for This Prototype

This prototype only has two SOPs:

- Absence forecast SOP
- Volume forecast SOP

For this implementation:

- each SOP is stored as raw JSON in S3
- each SOP is flattened into a text block for retrieval
- the text block is embedded
- one vector document is indexed per SOP in OpenSearch Serverless

This is sufficient for the prototype because retrieval only needs to return the correct SOP for the selected model.

Each indexed vector document contains:

    {
      "document_id": "sop_absence_forecast_v3",
      "model_type": "Absence forecast",
      "version": "v3",
      "title": "Absence Forecast Issue Classification SOP",
      "content": "Full flattened SOP text...",
      "embedding": [ ... ]
    }

---

# 15. Prompt Design

Prompt design is central to this implementation because it enforces the intended behavior.

The prompt ensures that:

- the LLM uses retrieved evidence
- the LLM follows SOP rule priority
- the LLM returns a structured response
- the explanation is adapted to the role
- the response remains explainable and auditable

## 15.1 System Prompt Template

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

    Role adaptation:
    - Operations Manager: simple language, operational impact, concise.
    - Forecasting Analyst: detailed reasoning, rule logic, signals, traceability.

    Return JSON only in this exact format:
    {
      "in_scope": true,
      "root_cause_code": "",
      "root_cause_name": "",
      "rule_id": "",
      "rule_reason": "",
      "signals_used": [],
      "role_adapted_explanation": "",
      "confidence_note": ""
    }

## 15.2 User Message Template

    {
      "user_role": "Operations Manager",
      "model_type": "Absence forecast",
      "warehouse": "BCN8",
      "date": "2026-04-10",
      "user_question": "Why is the absence forecast off for BCN8?",
      "session_context": [
        {
          "user_question": "Was it due to a local event?",
          "assistant_response": "No local event signal was found."
        }
      ],
      "retrieved_sop": { "...": "full SOP json..." },
      "retrieved_signals": [ "...signal records..." ]
    }

---

# 16. Example Operating Principle

The implementation follows a strict grounded-generation pattern:

1. The user selects the model and submits a question
2. The backend retrieves the correct SOP
3. The backend retrieves the relevant ML signals
4. The LLM receives both retrieved sources plus the session context
5. The LLM applies the SOP rules in order
6. The LLM selects the first matching root cause
7. The LLM explains the result in language adapted to the user role
8. The system stores the turn for memory, analytics, and auditability

This is the key principle that prevents the model from answering based only on the question.

---

# 17. Example Response Expectations

A valid AskMe response should include:

- whether the question is in scope
- root cause code
- root cause name
- rule ID used
- reason the rule matched
- signals used
- role-adapted explanation
- confidence note

The response must stay grounded in retrieved evidence and reference both:

- the SOP rule
- the ML signals used for classification

---

# 18. Assumptions for This Prototype

To keep the prototype practical and easy to deploy, the implementation assumes:

- AWS Region is `eu-west-1`
- frontend is a static HTML/JS app in S3
- backend is a single Lambda behind API Gateway
- no authentication is included in the prototype
- only two ML models are supported
- only one SOP exists per supported model
- SOP retrieval is based on vector search over flattened SOP documents
- ML signals are stored in DynamoDB and queried by model/site/date
- session memory is short-term and stored in DynamoDB
- escalation is stored only for demo analytics and does not trigger external workflows

---

# 19. Repository Contents

This repository should contain implementation assets such as:

- frontend files for the web UI
- backend deployment assets
- SOP JSON examples
- ML signal example data
- deployment scripts
- configuration files
- this README

This README intentionally documents the architecture, runtime behavior, data model, retrieval design, and prompt strategy without embedding full Lambda source code.

---

# 20. Summary

AskMe is a serverless, retrieval-grounded AI assistant for investigating forecasting anomalies.

Its main design principles are:

- **retrieval before classification**
- **LLM classification grounded in SOP + signals**
- **strict SOP rule order**
- **role-adapted explanation**
- **session-based memory**
- **traceability and explainability**
- **feedback and escalation support**

This design makes the prototype suitable for demonstrating how an AI assistant can support operational investigation workflows while remaining auditable, controlled, and grounded in official SOP logic.
