# PayOps

**An AI-assisted payment operations and reliability layer for small
merchants, using PayPal Sandbox.**

PayOps demonstrates a full payment-operations story: checkout → provider
processing → verified outcome → AI failure analysis → safe
reconciliation/recovery → customer communication and audit history.

> **Critical safety rule:** Never initiate another payment operation
> merely because a request timed out or appeared to fail. First
> establish from authoritative provider evidence and documented provider
> semantics that the previous operation did not complete and that
> another attempt is safe. If the outcome is unknown, reconcile or
> escalate---never blindly retry.

## Product overview

PayOps has two separate experiences:

1.  **Customer/demo storefront and checkout:** a customer chooses an
    order and starts a PayPal Sandbox payment.
2.  **Merchant operations dashboard:** an operator inspects payments,
    attempts, background jobs, failures, AI investigations,
    notifications, and audited recovery actions.

The backend---not a browser redirect---must verify the provider outcome.
Customer-supplied amounts/order data must not be trusted without
server-side validation.

## Architecture

``` text
Demo store / merchant backend
          |
          v
      PayOps API
          |
          | PostgreSQL transaction:
          | payment record + event + outbox event
          v
      PostgreSQL <-------------------------------+
          |                                      |
          v                                      |
    Outbox publisher                             |
          |                                      |
          v                                      |
       RabbitMQ                                  |
          |                                      |
          v                                      |
 Payment/processing worker                       |
          |                                      |
          v                                      |
     PayPal Sandbox                              |
          |                                      |
          v                                      |
 Verify and persist provider outcome ------------+
          |
          +-- confirmed success --> confirmation notification
          |
          +-- confirmed non-completion --> AI analysis
          |                                  |
          |                           backend safety policy
          |                                  |
          |                       retry only if authorized
          |
          +-- unknown outcome --> reconciliation/investigation
                                  no blind payment retry

Dashboard --> PayOps APIs --> payment history, attempts, jobs/DLQ,
                              AI analyses, investigations, notifications,
                              and audit records
```

This is a conceptual flow. Verify current official PayPal documentation
for checkout/capture lifecycle, idempotency, webhook verification, and
reconciliation before implementing provider-specific behavior.

## Preferred MVP stack

-   Frontend: Next.js + TypeScript.
-   Backend: Java + Spring Boot, as a **modular monolith**.
-   Database: PostgreSQL.
-   Messaging: RabbitMQ.
-   Local dependencies: Docker Compose.
-   Payment provider: PayPal Sandbox.
-   AI: model-backed per-attempt failure analyzer plus main
    investigation agent.
-   Notifications: email provider selected during implementation, plus
    dashboard/API status.

Use current supported compatible versions at project initialization.
Exact Java/Spring Boot versions, AI provider/model, email provider, and
final PayPal flow were not fixed in advance; verify and document
choices. Do not introduce Kafka alongside RabbitMQ without a clear
requirement.

## Core safety and reliability principles

1.  **Unknown is not failed.** Timeouts, lost responses, missing
    callbacks, and incomplete logs do not prove that payment did not
    complete.
2.  **AI is advisory.** It explains evidence and recommends actions.
    Backend policy plus authoritative provider evidence determine
    whether a retry or recovery is permitted.
3.  **AI confidence is not authorization.** If the model is unavailable,
    malformed, or inconclusive, preserve facts and reconcile/escalate.
4.  **Separate payment and job states.** A payment can be successful
    while its email/webhook-processing job is dead-lettered.
5.  **Use idempotency** for merchant requests, broker consumers, webhook
    processing, and notifications.
6.  **Use a transactional outbox** for DB-to-RabbitMQ publication.
    Expect duplicate publication and delivery.
7.  Never hold a database transaction open across a PayPal, AI, or email
    network call.
8.  Store original facts separately from AI interpretation.
9.  Retry notifications are sent only after the backend has authorized
    and scheduled a safe retry.
10. If the outcome is unknown, use a "we're checking your payment"
    update if appropriate---not "payment failed" or "we're retrying."
11. Recovery actions require backend authorization, state/policy checks,
    and an audit record.
12. Sanitize logs and AI context. Never expose secrets, raw provider
    payloads, stack traces, or unnecessary personal/payment data.

## Main workflows

### Payment initiation

1.  Customer selects Pay Now.
2.  A trusted merchant backend/integration calls PayOps.
3.  PayOps validates merchant/order/amount and applies an idempotency
    key.
4.  A single PostgreSQL transaction stores the logical payment operation
    and outbox event.
5.  The outbox publisher sends the event to RabbitMQ.
6.  Worker calls PayPal Sandbox using the documented provider flow.
7.  Backend verifies and stores the outcome; a redirect alone is not
    proof.
8.  Confirmed success produces a single confirmation notification.

### Failure analysis and retry gate

1.  Persist factual result, error category, timestamps,
    correlation/provider IDs, and sanitized logs.
2.  Build context from payment record, prior attempts, provider
    status/evidence, webhook history, idempotency state, and relevant
    logs.
3.  The small AI analyzer returns an evidence-based classification,
    explanation, missing evidence, and recommendation such as `RETRY`,
    `RECONCILE`, `STOP`, or `INVESTIGATE`.
4.  Persist AI output separately.
5.  Backend validates the recommendation against provider evidence,
    documented semantics, idempotency guarantees, payment state, policy,
    and retry limits.
6.  Authorize retry only when non-completion is established and another
    attempt is safe.
7.  Send a retry notification only after authorization and scheduling.
8.  If unknown, reconcile. If still unresolved after bounded attempts,
    escalate.

Three retries after the initial execution means up to four **job
executions**. That is not blanket permission to create four payment
attempts. Permanent errors should not be retried blindly.

### Reconciliation

Use a bounded, separate workflow for unknown outcomes. Reconcile using
supported provider APIs/webhooks and relevant provider identifiers.
Confirmed success prevents another charge. Definitive non-completion
permits a policy evaluation. Persistent uncertainty goes to
investigation/manual review. Reconciliation jobs may have their own DLQ
path.

### DLQ and investigations

A DLQ represents exhausted background processing work---not all failed
payments. Keep job status separate from payment status. Investigate one
job/payment or group related jobs using structured signatures (error
code, operation, service, time window). AI may summarize likely
patterns, but correlation is not proof of root cause. Investigation
tools should be narrow and primarily read-only; actual recovery goes
through an authorized backend endpoint.

### Notifications

-   Confirmed success: receipt/confirmation.
-   Retry authorized and scheduled: explain the retry and advise not to
    submit another payment.
-   Unknown outcome: explain that the payment is being checked and
    status is not yet confirmed.
-   Delayed resolution: calm update without internal DLQ/stack-trace
    details.
-   Confirmed failure: accurate next steps. Use notification records,
    deduplication keys, and bounded delivery retries to prevent
    duplicate messages.

## Initial logical data model

  -----------------------------------------------------------------------
  Entity/table                        Responsibility
  ----------------------------------- -----------------------------------
  `merchants`                         Merchant identity/configuration.

  `payment_operations`                One logical payment, trusted
                                      order/amount/currency, state,
                                      provider identifiers, idempotency
                                      data.

  `payment_attempts`                  Concrete provider operations and
                                      outcomes.

  `payment_events`                    Significant payment state
                                      transitions.

  `processing_jobs`                   Background job type/state, attempt
                                      limit, next run, completion/DLQ
                                      data.

  `processing_attempts`               Execution history for jobs.

  `outbox_events`                     Durable events to publish to
                                      RabbitMQ.

  `failure_analyses`                  AI explanation, recommendation,
                                      evidence references, unknowns,
                                      model/schema metadata.

  `dead_letter_records`               Queryable exhausted-job history.

  `investigation_reports`             Individual and batch
                                      investigations.

  `notification_records`              Notification type, dedupe key,
                                      delivery state/history.

  `audit_records`                     Operator actions,
                                      policy/authorization decisions, and
                                      outcomes.
  -----------------------------------------------------------------------

These are starting concepts, not a mandate to create every table before
the workflow requires it. Use migrations. Avoid a generic `dlq=true`
flag on a payment row.

## Proposed repository layout

``` text
PayOps/
├── README.md
├── information.md
├── .gitignore
├── .env.example
├── frontend/                         # Next.js + TypeScript
│   └── src/
│       ├── app/
│       │   ├── page.tsx
│       │   ├── shop/page.tsx
│       │   ├── checkout/page.tsx
│       │   ├── payment-result/page.tsx
│       │   └── dashboard/
│       │       ├── page.tsx
│       │       ├── payments/page.tsx
│       │       ├── investigations/page.tsx
│       │       ├── investigations/[id]/page.tsx
│       │       └── dlq/page.tsx
│       ├── components/{ui,checkout,payments,dashboard,investigations,dlq}/
│       ├── features/{payments,investigations,reliability,dlq}/
│       ├── lib/{api.ts,types.ts}
│       └── hooks/
├── payment-service/                   # one Spring Boot backend
│   ├── pom.xml
│   └── src/
│       ├── main/java/com/payops/
│       │   ├── PayOpsApplication.java
│       │   ├── config/  common/  merchant/
│       │   ├── payment/ paypal/ processing/
│       │   ├── messaging/ outbox/ retry/ dlq/
│       │   ├── diagnostics/ investigation/
│       │   ├── notification/ audit/ observability/
│       ├── main/resources/
│       │   ├── application.yml
│       │   └── db/migration/
│       └── test/java/com/payops/
├── infrastructure/
│   └── docker-compose.yml              # PostgreSQL + RabbitMQ
├── docs/
│   ├── architecture/
│   ├── api/
│   └── demo-scenarios/
└── scripts/
```

This is a guide, not evidence that files already exist. Inspect the
repository first, preserve existing work, and add packages incrementally
rather than creating empty folders in advance.

## Implementation milestones

1.  Inspect repository, existing README/configuration, Git status,
    tests, and official provider documentation.
2.  Present the mandatory pre-implementation briefing described in
    `information.md`; wait for explicit approval before changing files.
3.  Establish frontend/backend foundations and Docker Compose
    dependencies.
4.  Add migrations, domain models, API validation, health checks, and
    error handling.
5.  Implement a minimal PayPal Sandbox flow with server-side outcome
    verification.
6.  Add transactional outbox, RabbitMQ consumers, job/attempt history,
    and idempotency.
7.  Implement explicit unknown outcomes and reconciliation before
    enabling payment retries.
8.  Add the AI analyzer, stored evidence-grounded analyses, and safe
    fallback behavior.
9.  Add notifications and deduplication.
10. Add dashboard, investigation agent, DLQ views, controlled recovery,
    and audit.
11. Test duplicate delivery, timeout ambiguity, success-after-timeout,
    AI outage, notification duplication, and recovery authorization.
12. Document a repeatable demo and honest limitations.

## Minimum acceptance criteria

-   Verified PayPal Sandbox outcome is recorded.
-   Browser redirect alone never marks payment successful.
-   Repeated merchant request with the same idempotency key does not
    create a duplicate logical payment.
-   Duplicate broker deliveries do not repeat business side effects.
-   Unknown timeout outcome enters reconciliation, not a new payment
    attempt.
-   Confirmed success discovered during reconciliation prevents another
    charge.
-   A retry requires authoritative evidence plus backend policy
    authorization.
-   AI failure/uncertainty cannot authorize retry.
-   Retry notification is sent only after retry authorization and
    scheduling.
-   Payment state remains independent from processing-job/DLQ state.
-   Notification redelivery does not send duplicate logical messages.
-   Recovery is permission-checked and audited.
-   Sensitive data is redacted from logs and AI context.
-   Automated tests cover safety-critical failure paths.

## Configuration and security

Use environment variables and provide `.env.example` with placeholder
names only. Keep `.env` and all credentials out of version control. Use
synthetic data and PayPal Sandbox. Select AI and email providers based
on current documentation and constraints; document data sent to those
providers, required credentials, and any material cost. Do not claim a
test, build, or provider flow succeeded unless it was actually run and
verified.

For the full product context, agent autonomy rules, detailed AI
behavior, test scenarios, and the mandatory approval checkpoint, read
[`information.md`](./information.md).
