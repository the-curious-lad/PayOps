# PayGuard AI

### AI-Powered Payment Reliability, Investigation, and Recovery

PayGuard AI is a payment reliability layer built around PayPal
integration. A merchant routes supported payment operations through
PayGuard, which uses PayPal APIs to process payments and maintains its
own records of payment state, attempts, events, and recovery work.

When a payment becomes stuck or an internal processing task repeatedly
fails, an AI investigation agent gathers evidence from PayGuard's
database, available PayPal information, webhook/event history, and
connected application logs. It explains what the evidence establishes,
what remains uncertain, and what safe next step is recommended. A
merchant dashboard provides payment history, investigation reports,
reliability insights, dead-letter queue (DLQ) inspection, and controlled
recovery actions.

> **Core idea:** PayPal processes the payment. PayGuard adds
> merchant-facing tracking, investigation, operational reliability, and
> controlled recovery around that integration.

## The Problem

A payment request can time out even if the provider processed it. A
webhook handler can fail after the payment succeeds. A local database
record can remain pending while the provider reports a completed
payment. Repeating a payment blindly can risk charging a customer twice,
while a failed background task does not necessarily mean the payment
itself failed.

Merchants need to understand what happened, what the available evidence
proves, what remains unknown, and what action is safe.

## The Solution

PayGuard brings together four capabilities:

1.  **PayPal payment integration:** PayGuard's backend uses supported
    PayPal APIs for the payment flow.
2.  **Separate PayGuard records:** A dedicated database tracks payment
    operations, provider references, attempts, events, processing jobs,
    investigations, and audit history.
3.  **Reliability engine:** Asynchronous workers, durable processing,
    bounded retries, reconciliation, and dead-letter handling for
    eligible internal jobs.
4.  **AI investigation and dashboard:** An agent gathers evidence using
    controlled tools, explains individual failures, identifies recurring
    patterns, and recommends safe next steps.

The AI is not the source of truth for payment state. Provider
information and PayGuard's durable records are evidence; deterministic
backend rules validate and execute recovery actions.

## Product Principles

-   PayPal remains responsible for processing the actual payment.
-   PayGuard stores only the data required to track and investigate
    payment operations it handles; it does not need to copy the
    merchant's entire database.
-   Evidence is gathered before diagnosis.
-   Reports distinguish confirmed facts, hypotheses, and unknowns.
-   Payment state and processing-job state are separate.
-   An uncertain payment outcome must be reconciled before another
    charge is considered.
-   The AI recommends; backend rules enforce permissions, idempotency,
    payment-state rules, and recovery policies.
-   Recovery actions are validated, audited, and verified.

## End-to-End Flow

1.  **Customer initiates payment.** The merchant application sends a
    payment request to PayGuard.
2.  **Validate and deduplicate.** PayGuard validates the request against
    trusted server-side order and amount data and applies idempotency
    rules.
3.  **Persist the operation.** PayGuard creates its payment record and
    an outbox event in one database transaction.
4.  **Publish work.** An outbox publisher sends pending events to
    RabbitMQ. This prevents a committed database record from being
    stranded if message publication fails.
5.  **Process payment.** A worker consumes the job and uses the
    appropriate PayPal API flow. It records provider identifiers and
    returned evidence.
6.  **Record the outcome.**
    -   Confirmed success: record the verified result.
    -   Confirmed decline/failure: record the outcome and available
        reason.
    -   Timeout or ambiguous result: mark it unresolved and reconcile
        with PayPal before considering another payment operation.
    -   Internal processing error: apply the relevant retry policy only
        if the operation is safe to retry.
7.  **Process events and webhooks.** Where supported and configured,
    PayPal webhooks provide event notifications. PayGuard verifies,
    records, and processes them idempotently.
8.  **Handle eligible failures.** Internal tasks may be retried with
    bounded attempts and backoff. Jobs that exhaust the policy go to the
    DLQ. Ambiguous payment outcomes follow a reconciliation path rather
    than blindly repeating a charge.
9.  **Investigate.** The merchant can investigate an individual payment
    or review patterns across failed jobs.
10. **Recover safely.** The backend validates an authorized action,
    executes it, records an audit trail, and verifies the resulting
    state.

## Architecture

``` text
Customer
   |
Merchant Checkout
   |
PayGuard API ---- validates request + idempotency
   |
   v
PayGuard Database (payment record + transactional outbox)
   |
Outbox Publisher
   |
RabbitMQ
   |
Payment Worker
   |
PayPal APIs
   |
   +---- confirmed success / confirmed failure / uncertain outcome
                         |
                         v
                 Update PayGuard records
                         |
                         v
                  Merchant Dashboard


Internal processing failures
   |
Bounded retry + backoff
   |
   +---- retry succeeds ---> job completed
   |
   +---- retry limit reached ---> DLQ
                                    |
                                    v
                           AI Investigation Agent
                                    |
                    +---------------+---------------+
                    |               |               |
                PayGuard DB     PayPal APIs    Connected logs
                    |               |               |
                    +---------------+---------------+
                                    |
                          Evidence-backed findings
                                    |
                          Validated recovery action
                                    |
                             Audit + verification
```

This is a logical architecture. Webhooks, reconciliation jobs, and
dashboard investigations may enter through different paths.

### Why RabbitMQ for the MVP?

RabbitMQ is a practical starting point for a work-queue pattern with
acknowledgements, retries, and dead-letter handling. Kafka is also
viable if durable event streams, replay, and multiple independent
consumers are central requirements. The first version should choose one
broker, not both.

### Why the transactional outbox?

Writing to a database and publishing a message are two separate
operations. If the database write succeeds but publishing fails, the
payment can be recorded without a queued job. Persisting an outbox event
in the same database transaction and publishing it asynchronously allows
failed publications to be retried.

Queue durability alone does not guarantee exactly-once external payment
execution. Workers must expect duplicate delivery and use idempotency
and reconciliation.

## Core Components

### Payment API

-   Accepts merchant payment requests.
-   Validates required fields and trusted order/amount data.
-   Enforces idempotency for repeated requests.
-   Creates the payment record and outbox event transactionally.
-   Returns an operation reference and current processing state.

### PayGuard Database

Stores PayGuard's operational record of payments handled by the layer.
It is separate from the merchant's database and should not unnecessarily
duplicate customer or merchant business data.

### Outbox Publisher

Publishes pending events to the broker and retries publication failures.

### Payment Worker

Consumes queued work, checks current state, invokes supported PayPal
operations, records outcomes, and acknowledges messages after the
relevant durable processing step. Since a worker may crash after an
external operation succeeds but before acknowledgement, duplicate
delivery must be expected.

### PayPal Integration

Uses the supported PayPal APIs and webhook capabilities relevant to the
selected payment flow. Exact operations, permissions, statuses,
idempotency mechanisms, and webhook event types must be verified against
current PayPal documentation. PayGuard must not claim access to every
internal PayPal processing stage.

### Reliability Engine

-   Tracks processing jobs and attempts.
-   Classifies failures by operation and retry eligibility.
-   Schedules bounded retries with backoff.
-   Routes eligible exhausted jobs to a DLQ.
-   Triggers reconciliation for uncertain payment outcomes.
-   Keeps job status separate from payment status.

### AI Investigation Agent

Uses narrowly scoped backend tools to retrieve evidence and produce
structured investigation results. It can recommend actions but cannot
bypass authorization or payment-safety rules.

### Merchant Dashboard

Provides payment search, timelines, individual investigations,
reliability insights, DLQ inspection, controlled recovery controls, and
audit history.

## Payment State vs. Processing-Job State

A payment can be successful while a webhook-processing job fails. A job
can fail before the payment outcome is known. These concepts must not
share one status field.

### Example internal payment states

The final state machine must be defined during implementation. Possible
states include:

-   `RECEIVED`
-   `QUEUED`
-   `PROCESSING`
-   `PENDING`
-   `SUCCEEDED`
-   `FAILED`
-   `UNRESOLVED`
-   `RECONCILIATION_REQUIRED`

### Example processing-job states

-   `PENDING`
-   `PROCESSING`
-   `RETRY_SCHEDULED`
-   `COMPLETED`
-   `DEAD_LETTERED`
-   `CANCELLED`

### Example incident states

-   `OPEN`
-   `INVESTIGATING`
-   `ACTION_REQUIRED`
-   `RESOLVED`
-   `CLOSED`

These are proposed PayGuard states, not PayPal API status names.

## AI Investigation

PayGuard supports two primary investigation workflows.

### Investigate one payment

The merchant selects a payment and clicks **Investigate**. The agent can
retrieve:

-   Internal payment record and state transitions.
-   Payment attempts and timestamps.
-   PayPal order/capture references and available current status.
-   PayGuard-recorded payment and webhook events.
-   Webhook receipt and processing history.
-   Related jobs and retry history.
-   Sanitized application logs, if the merchant has connected them.

The report should include:

1.  Summary of what is known.
2.  Timeline of relevant events.
3.  Evidence references supporting the findings.
4.  Likely cause, clearly labelled as a hypothesis when unconfirmed.
5.  Unknowns and missing evidence.
6.  Recommended next step.
7.  Safety considerations.

**Example**

> **Finding:** PayPal's available information indicates that the capture
> completed, but PayGuard's internal record remains pending. The
> original request timed out and the expected local update is not
> recorded.
>
> **Recommended action:** Reconcile the internal record against the
> confirmed provider outcome and inspect webhook-processing history. Do
> not initiate another charge.

### Analyse recurring DLQ failures

The system can examine dead-lettered jobs, error metadata, retry
histories, job types, timestamps, and related payment references. The AI
can group similar error signatures and explain recurring patterns, such
as database connectivity failures, missing internal references,
webhook-processing errors, or configuration problems.

The AI should recommend investigations rather than claim an unproven
root cause. It must not automatically replay every job.

### Proposed internal tool contracts

These are conceptual tools to implement in PayGuard, not names of
PayPal-provided AI tools:

-   `get_payment_record(payment_id)`
-   `get_payment_attempts(payment_id)`
-   `get_payment_events(payment_id)`
-   `get_paypal_payment_status(provider_reference)`
-   `get_webhook_history(payment_id)`
-   `get_processing_job(job_id)`
-   `get_retry_history(job_id)`
-   `get_related_jobs(payment_id, error_signature)`
-   `get_sanitized_diagnostics(payment_id, time_range)`
-   `request_reconciliation(payment_id)`
-   `request_job_replay(job_id)`
-   `create_investigation_report(payment_id)`

Read-only evidence tools should be separated from state-changing tools.
Action tools must enforce authentication, authorization, validation,
idempotency where applicable, and audit logging. Risky actions may
require human approval.

## Retry and Dead-Letter Queue

Retries apply to **eligible internal processing jobs**, not
indiscriminately to every payment operation.

``` text
Internal job received
        |
     Attempt 1 fails
        |
  Retry with backoff
        |
     Attempt 2 fails
        |
  Retry with backoff
        |
     Attempt 3 fails
        |
   Retry limit reached
        |
        v
       DLQ
        |
        v
Investigate + validate recovery
```

Three attempts are an example MVP configuration, not a universal rule.
Use bounded retries, backoff, and error classification so permanent
failures are not repeatedly retried.

Important safeguards:

-   Retrying a webhook handler must not charge the customer again.
-   A timeout during a payment operation is ambiguous, not proof of
    failure.
-   Reconcile with PayPal before considering another payment operation
    after an uncertain result.
-   Do not acknowledge and discard a message before required durable
    processing is complete.
-   Expect duplicate message delivery and make consumers idempotent.
-   Keep authoritative attempt counts and job states in durable records.
-   DLQ replay must be explicit, audited, and guarded by safety checks.

## Data Model

A conceptual starting model:

### `PaymentOperation`

-   `payment_id`
-   `merchant_id`
-   `merchant_order_id`
-   `idempotency_key`
-   `amount`
-   `currency`
-   `payment_status`
-   `provider`
-   `provider_order_id`
-   `provider_capture_id`
-   `created_at`
-   `updated_at`

### `PaymentAttempt`

-   `attempt_id`
-   `payment_id`
-   `attempt_type`
-   `started_at`
-   `finished_at`
-   `outcome`
-   `error_code`
-   `provider_debug_reference`, where available
-   `retry_eligible`

Distinguish customer-initiated attempts from internal retries and
reconciliation calls.

### `PaymentEvent`

-   `event_id`
-   `payment_id`
-   `event_type`
-   `source`
-   `occurred_at`
-   `received_at`
-   `correlation_id`
-   `sanitized_payload_reference`

### `ProcessingJob`

-   `job_id`
-   `payment_id`
-   `job_type`
-   `job_status`
-   `attempt_count`
-   `max_attempts`
-   `last_error`
-   `next_attempt_at`
-   `created_at`
-   `updated_at`

### `Incident`

-   `incident_id`
-   `payment_id`, where applicable
-   `incident_type`
-   `severity`
-   `status`
-   `summary`
-   `evidence_references`
-   `created_at`
-   `resolved_at`

### `RecoveryAction` and `AuditRecord`

Track action type and target, requesting actor, approval where required,
validation result, execution result, before/after state references,
timestamps, and correlation IDs.

### `OutboxEvent`

Track the event ID, aggregate ID, event type, payload reference, publish
status, creation time, and publication time.

The final schema should be adapted to the selected database and
implementation.

## Dashboard Experience

### 1. Payment Overview

Recent payments, amounts, states, unresolved/inconsistent records,
investigations needed, and jobs awaiting retry or in the DLQ.

### 2. Payment Investigation

A timeline of events, internal and provider-side status, attempts,
errors, webhook history, connected diagnostics, and an AI report with
evidence and uncertainty clearly shown.

### 3. Reliability Insights

Retry outcomes, DLQ size and age, repeated error signatures, recurring
patterns, and recommended investigations.

### 4. Recovery Operations

Inspect eligible jobs, request controlled replay, request payment
reconciliation, and review approvals, action results, and audit history.

The dashboard must not imply that a merchant can change PayPal's
internal state. It shows available evidence and controls only the
actions exposed and authorized by PayGuard.

## Security and Reliability

-   Keep PayPal credentials server-side; never expose secrets to the
    browser or AI context.
-   Verify webhook signatures and process duplicate webhook deliveries
    idempotently.
-   Enforce authentication and merchant-level authorization on all
    operations.
-   Validate amounts and order ownership using trusted backend data.
-   Treat provider timeouts as ambiguous until reconciled.
-   Combine durable database records, outbox publishing, persistent
    broker messages, acknowledgements, and retry policies.
-   Redact tokens, secrets, and unnecessary personal data from logs and
    AI context.
-   Limit the AI to the minimum data and tools needed.
-   Keep state-changing operations behind deterministic backend
    validation.
-   Audit recovery requests and outcomes.
-   Investigate only information exposed through supported APIs,
    webhooks, and authorized integrations.

## MVP Scope

### Build

1.  A reference merchant checkout.
2.  A working PayPal Sandbox payment flow through PayGuard.
3.  A separate PayGuard database.
4.  Idempotent payment request handling.
5.  RabbitMQ asynchronous processing.
6.  Transactional outbox publishing.
7.  Payment state and event history.
8.  PayPal status lookup/reconciliation for the selected flow.
9.  Bounded retries and a DLQ for eligible internal jobs.
10. An AI agent with controlled investigation tools.
11. A merchant dashboard for payments, investigations, DLQ analysis, and
    safe recovery.
12. Deterministic failure simulation and tests.

### Defer

-   Plug-and-play access to arbitrary merchant databases.
-   Support for every payment method or PayPal integration flow.
-   Integrations with every logging provider.
-   Multiple message brokers.
-   Unrestricted autonomous recovery.
-   Automatic replay of ambiguous payment operations.

## Demo Scenarios

### 1. Provider succeeded, local state is stale

Simulate a confirmed provider outcome while the local state update or
event handler fails. Show PayGuard detecting the inconsistency,
gathering evidence, and reconciling safely without another charge.

### 2. Webhook-processing job enters the DLQ

Simulate an internal webhook-handler failure and exhaust the configured
retries. Show the job in the DLQ, the AI analysing the attempt history
and related payment, and a validated recovery option.

### 3. Payment outcome is uncertain

Simulate a timeout at the application boundary. Do not assume PayPal
failed. Show the agent checking available provider information and
either reporting the verified outcome or leaving it unresolved with a
recommendation for further verification.

### 4. Many DLQ jobs share a failure pattern

Create multiple controlled jobs with the same internal failure
signature. Show the dashboard grouping the failures and the AI
explaining the common pattern and next diagnostic step.

Clearly label simulated application failures as simulations; do not
present them as real PayPal outages.

## Suggested Implementation Plan

### Phase 1 --- Payment foundation

Set up PayPal Sandbox, the reference checkout, PayGuard API, idempotency
behaviour, payment schema, and state machine.

### Phase 2 --- Durable asynchronous processing

Add RabbitMQ, the transactional outbox, publisher, payment worker,
acknowledgements, and duplicate-delivery handling.

### Phase 3 --- Provider evidence and reconciliation

Store PayPal identifiers and responses, implement the relevant status
lookups and webhooks, verify webhook authenticity, and build the
uncertain-outcome reconciliation path.

### Phase 4 --- Reliability operations

Add processing-job records, error classification, bounded retries,
backoff, DLQ routing, controlled replay, and audit logging.

### Phase 5 --- AI investigation

Implement read-only evidence tools, a structured evidence bundle and
report, explicit distinction between facts/hypotheses/unknowns, and
guarded action tools after backend safety checks exist.

### Phase 6 --- Dashboard and demo

Build payment overview, transaction timelines, individual investigation,
DLQ insights, recovery controls, and tests for the demo scenarios.

## Technology Choices

  -----------------------------------------------------------------------
  Layer                   Suggested choice        Reason
  ----------------------- ----------------------- -----------------------
  Backend API             Java + Spring Boot      API, transaction, and
                                                  integration logic

  Database                PostgreSQL              Transactions,
                                                  constraints, durable
                                                  records, and outbox
                                                  pattern

  Message broker          RabbitMQ                Work queues,
                                                  acknowledgements,
                                                  retry/DLQ patterns

  Payment provider        PayPal Sandbox          Test the supported
                                                  PayPal payment flow

  AI                      Tool-calling LLM        Evidence gathering and
                                                  investigation

  Dashboard               Next.js + TypeScript    Merchant-facing
                                                  interface

  Observability           Structured logs +       Link requests, jobs,
                          correlation IDs         events, and
                                                  investigations
  -----------------------------------------------------------------------

These are proposed choices, not hard requirements. Choose one broker and
keep the MVP small enough to finish and demonstrate.

## Limitations and Non-Goals

-   PayGuard does not replace PayPal or independently settle funds.
-   It cannot inspect PayPal's private internal infrastructure unless
    relevant information is explicitly exposed through supported
    capabilities.
-   Merchant application logs are available only when collected or
    connected with authorization.
-   Merchant databases are not automatically accessible; integration
    requires a defined API, adapter, or explicit permission.
-   AI diagnosis can be uncertain and must not invent certainty.
-   A DLQ is for work that could not be processed under its policy; it
    is not a bucket for every failed payment.
-   Queue durability does not alone guarantee exactly-once external
    payment execution.
-   Recovery actions require validation, authorization, idempotency, and
    audit controls.

## Success Criteria

The MVP should demonstrate that:

-   A customer can complete a PayPal Sandbox payment through PayGuard.
-   Duplicate submissions do not create duplicate logical payment
    operations.
-   Payment records and queue publication survive designed failure
    cases.
-   Uncertain outcomes are not blindly retried as new charges.
-   Eligible internal jobs follow bounded retries and reach the DLQ when
    appropriate.
-   The AI uses internal evidence and available PayPal information to
    produce traceable investigations.
-   The dashboard explains individual payment issues and recurring DLQ
    patterns clearly.
-   Recovery actions are validated, audited, and verified.

## Positioning

**PayGuard AI is an AI-powered payment reliability and investigation
layer built around PayPal integration.**

It combines payment processing, durable asynchronous work, internal
state tracking, provider-state reconciliation, controlled retries,
dead-letter analysis, and evidence-backed AI investigations in one
merchant-facing workflow.

The differentiator is not simply displaying payment statuses. It is
helping a merchant understand **what happened, what the evidence
establishes, what remains uncertain, and what safe action should happen
next**.
