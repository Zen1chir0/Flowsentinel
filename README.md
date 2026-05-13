# FlowSentinel

AI-assisted reliability and incident classification for n8n workflows.

<p align="center">
  <img src="./screenshots/workflow-architecture.png" alt="FlowSentinel workflow architecture" width="100%">
</p>

FlowSentinel is an external observability layer for n8n automations. It captures failed workflow executions, stores the raw incident payload in Supabase, normalizes the failure into an operational incident, classifies it with deterministic rules first, and uses AI only when the rule layer cannot confidently identify the failure.

The guiding principle is simple:

> Reliability-critical decisions should be deterministic. AI should assist with ambiguous diagnosis, not silently control the incident pipeline.

## Table of Contents

- [Why FlowSentinel Exists](#why-flowsentinel-exists)
- [Key Features](#key-features)
- [Architecture](#architecture)
- [Workflow Pipeline](#workflow-pipeline)
- [Deterministic Classification](#deterministic-classification)
- [AI Fallback](#ai-fallback)
- [Incident Lifecycle](#incident-lifecycle)
- [Data Model](#data-model)
- [Sample Payload](#sample-payload)
- [Sample Notification](#sample-notification)
- [Tech Stack](#tech-stack)
- [Project Files](#project-files)
- [Roadmap](#roadmap)
- [Screenshots](#screenshots)

## Why FlowSentinel Exists

Automation workflows often fail for ordinary but costly reasons:

- invalid API requests
- expired credentials or OAuth tokens
- malformed payloads
- schema mismatches
- rate limits
- webhook delivery issues
- unstable upstream services
- orchestration logic errors

n8n provides execution logs and error triggers, but production teams often still need a separate layer for:

- centralized incident records
- structured failure classification
- retryability analysis
- ownership-aware escalation
- actionable notifications
- audit-friendly operational history

FlowSentinel turns raw workflow failures into structured incidents that can be classified, stored, routed, and reviewed.

## Key Features

- Captures failed n8n executions through an Error Trigger
- Persists raw incident payloads in Supabase for audit and replay
- Normalizes inconsistent workflow errors into a common incident shape
- Generates incident fingerprints for grouping and deduplication
- Classifies known failures with deterministic operational rules
- Uses OpenAI as a fallback for ambiguous or unknown failures
- Determines severity, retryability, escalation need, and owner routing
- Sends structured incident notifications to workflow owners or main developers

## Architecture

```text
Failing Workflow Environment
        |
        v
n8n Error Trigger
        |
        v
POST Request to FlowSentinel
        |
        v
FlowSentinel Webhook Intake
        |
        v
Raw Payload Persistence (Supabase)
        |
        v
Incident Spec Generation
        |
        v
Deterministic Classification Engine
        |
        +-- Match found --> Save Classified Incident
        |                  Notify Responsible Owner
        |
        +-- No match -----> AI Incident Classification
                           Save Classified Incident
                           Notify Responsible Owner
```

![FlowSentinel architecture](./screenshots/flowsentinel-architecture.png)

## Workflow Pipeline

### 1. Failed Workflow Execution

A monitored n8n workflow fails in a separate workflow environment. Example failures include invalid endpoints, expired OAuth tokens, malformed JSON, failed HTTP requests, and missing required fields.

### 2. Error Trigger Capture

The n8n Error Trigger captures the failed execution metadata and forwards it to FlowSentinel through an HTTP POST request.

The payload can include:

- workflow metadata
- execution metadata
- failed node information
- error details
- environment
- ownership metadata

### 3. Raw Payload Persistence

FlowSentinel stores the original incoming payload in Supabase before classification.

This preserves:

- the original failure event
- execution context
- ownership metadata
- incident audit history
- future replay and debugging context

### 4. Incident Spec Generation

The incident spec layer converts raw workflow errors into normalized operational incident objects.

It is responsible for:

- standardizing inconsistent payload shapes
- converting status codes into consistent values
- normalizing workflow, node, and environment metadata
- deriving a stable incident fingerprint
- preparing deterministic classification inputs
- preparing compact AI context for fallback reasoning

![Incident spec generation](./screenshots/incident-spec-generation.png)

### 5. Deterministic Classification

The deterministic engine is the primary reliability layer. It handles known operational patterns such as HTTP errors, authentication failures, rate limits, validation issues, network instability, and schema mismatches.

![Deterministic engine](./screenshots/deterministic-engine.png)

### 6. AI Classification Fallback

When deterministic rules cannot confidently classify an incident, FlowSentinel routes the normalized incident to the AI layer for contextual analysis.

![AI classification](./screenshots/ai-classification.png)

### 7. Classified Incident Persistence

The classified incident is stored in a structured Supabase table with the classification source, severity, retryability, routing decision, and recommended action.

### 8. Notification Routing

If escalation is required, FlowSentinel determines the correct recipient and notification path.

Targets can include:

- workflow owners
- main developers

![Incident notification](./screenshots/incident-notification.png)

## Deterministic Classification

The deterministic layer handles the cases that should be predictable, repeatable, and auditable.

It determines:

- failure category
- severity
- retryability
- escalation requirement
- escalation target
- recommended action

Example rules:

```text
Status: 404
Category: not_found
Retryable: false
Escalate: true
Target: workflow_owner
```

```text
Status: 429
Category: rate_limit
Retryable: true
Escalate: false
```

```text
Pattern: credential or OAuth failure
Category: credential_or_oauth_error
Severity: high
Retryable: false
Escalate: true
Target: main_devs
```

The deterministic layer can also apply escalation guardrails such as environment checks, critical keyword detection, ownership-aware routing, and notification path generation.

## AI Fallback

AI is used only after deterministic classification fails to produce a confident result.

The AI layer helps with:

- ambiguous error messages
- unstructured failure details
- inferred operational causes
- severity estimation
- remediation suggestions
- unknown failure patterns

This keeps AI useful without making it the authority for reliability-sensitive decisions.

## Incident Lifecycle

```text
Workflow Failure
    |
    v
Error Trigger Capture
    |
    v
Payload Ingestion
    |
    v
Raw Incident Storage
    |
    v
Incident Spec Generation
    |
    v
Deterministic Classification
    |
    v
AI Fallback, if needed
    |
    v
Classified Incident Storage
    |
    v
Escalation Routing
    |
    v
Email Notification
```

## Data Model

FlowSentinel separates raw ingestion from classified incidents.

### Raw Incidents

The raw incident table stores:

- original payload
- workflow metadata
- execution metadata
- ownership metadata
- incident fingerprint
- ingestion timestamp

This table supports auditing, debugging, and future replay.

### Classified Incidents

The classified incident table stores:

- classification source
- failure category
- severity
- retryability
- escalation decision
- escalation target
- recommended action
- AI reasoning, when applicable
- notification metadata

This table supports operational tracking, incident review, and future analytics.

## Sample Payload

```json
{
  "workflow_name": "HubSpot Lead Sync",
  "workflow_id": "abc123",
  "execution_id": "91",
  "node_name": "HTTP Request",
  "error_type": "NodeApiError",
  "status_code": 404,
  "error_message": "The resource you are requesting could not be found",
  "method": "GET",
  "environment": "production",
  "workflow_owner_email": "owner@company.com",
  "main_dev_email": "devs@company.com"
}
```

## Sample Notification

```text
Subject: [FlowSentinel][HIGH] authentication_error in HubSpot Lead Sync

FlowSentinel detected a workflow incident requiring attention.

Workflow: HubSpot Lead Sync
Environment: production
Severity: high
Category: authentication_error

Recommended Action:
Check expired credentials, API keys, OAuth scopes, and token refresh handling.
```

## Tech Stack

- n8n for workflow orchestration, error capture, routing, and notifications
- Supabase for raw and classified incident persistence
- PostgreSQL for structured incident storage
- OpenAI API for fallback incident analysis
- JavaScript for workflow-level transformation and classification logic

## Project Files

```text
.
+-- Flowsentinel.json
+-- README.md
+-- screenshots/
```

`Flowsentinel.json` contains the exported n8n workflow. To inspect or run the workflow, import it into an n8n instance and configure the required Supabase, OpenAI, and notification credentials.

## Roadmap

Planned improvements:

- retry orchestration with retry-state awareness
- incident deduplication and grouping
- workflow ownership registry
- Slack and Discord alerting
- incident analytics dashboard
- replay-safe recovery workflows
- state tracking across failures and retries
- adaptive retry policies based on incident history

## Screenshots

### Workflow Architecture

![Workflow architecture](./screenshots/workflow-architecture.png)

### Flowchart

![Flowchart](./screenshots/flowchart.png)

### Deterministic Checks

![Deterministic checks](./screenshots/deterministic-checks.png)

### AI Classification

![AI classification](./screenshots/ai-classification.png)

### Incident Email

![Incident email](./screenshots/incident-email.png)

![Incident email 2](./screenshots/incident-email2.png)

## Summary

FlowSentinel demonstrates a reliability pattern for AI-assisted automation systems:

- deterministic logic remains authoritative
- AI is constrained to fallback reasoning
- workflow failures become structured operational incidents
- ownership-aware escalation improves response time
- raw and classified incident records support observability and auditability
