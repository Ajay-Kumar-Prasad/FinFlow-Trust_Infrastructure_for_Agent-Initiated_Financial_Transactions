# FinFlow Architecture Specification v0.1

## 1. Problem Statement

FinFlow is a trust infrastructure platform for agent-initiated financial transactions.

The system allows software agents to initiate payments on behalf of users while preventing agents from having unrestricted authority over financial execution.

The core problem is that software agents are probabilistic systems, while financial transactions require deterministic enforcement of authorization, financial correctness, and security.

FinFlow introduces a deterministic trust and control layer between the agent and the financial settlement system.

The system must ensure that every agent-initiated transaction is:

* Authenticated to a known agent identity.
* Authorized under an explicit delegation policy.
* Within the user's configured spending limits.
* Evaluated against deterministic risk controls.
* Subject to human approval when required.
* Processed idempotently.
* Recorded correctly in the financial ledger.
* Fully auditable.
* Recoverable when distributed components fail.

The fundamental architectural boundary is:

```text
Probabilistic Agent
        |
        v
+---------------------------+
|     FinFlow Trust Layer   |
|                           |
| Identity                  |
| Delegated Authorization   |
| Policy                    |
| Risk                      |
| Human Approval            |
| Audit                     |
+---------------------------+
        |
        v
Deterministic Payment Engine
        |
        v
Financial Ledger
        |
        v
Payment Settlement
```

Agents may propose payment intent, but agents must never directly control financial settlement.

---

## 2. Goals

### G1. Secure Agent Authorization

Allow users to delegate financial authority to software agents using explicit, scoped policies.

Authorization must support constraints such as:

* Maximum transaction amount.
* Daily spending limit.
* Monthly spending limit.
* Allowed merchants or merchant categories.
* Allowed beneficiaries.
* Policy expiration.
* Human approval thresholds.
* Agent activation and revocation.

An agent must not be able to exceed the authority granted by the user.

### G2. Financial Correctness

Ensure that financial state remains correct even when multiple payment requests are processed concurrently.

The system must prevent:

* Double spending.
* Duplicate payments.
* Incorrect balance updates.
* Invalid ledger entries.
* Partial financial state changes.

Financial state must be protected using transactional database guarantees rather than relying on application-level assumptions.

### G3. Idempotent Payment Processing

The same payment request must not result in multiple financial transactions when clients, agents, networks, or downstream services retry requests.

The system must support idempotency keys and detect conflicting reuse of an idempotency key.

### G4. Reliable Distributed Processing

Use event-driven architecture to decouple payment processing from downstream activities such as:

* Risk processing.
* Ledger projections.
* Notifications.
* Analytics.
* Audit processing.

Events must be recoverable when failures occur.

### G5. Auditability

Every important authorization and payment decision must produce an auditable record.

The audit trail should make it possible to determine:

* Who initiated the action.
* Which agent initiated it.
* Which policy authorized or rejected it.
* Which risk decision was produced.
* Whether human approval was required.
* What payment state transitions occurred.
* Which services processed the transaction.

### G6. Risk Control

Evaluate agent-initiated payments using deterministic risk rules.

The initial risk engine should consider signals such as:

* Transaction amount.
* Transaction velocity.
* Merchant.
* Merchant category.
* Agent history.
* Beneficiary.
* Time of transaction.
* Previous failures.
* Policy violations.

Risk decisions should result in:

```text
LOW      -> AUTO_APPROVE
MEDIUM   -> REQUIRE_APPROVAL
HIGH     -> BLOCK
```

### G7. Observability

The system must provide sufficient telemetry to understand system behavior in production-like environments.

This includes:

* Structured logs.
* Metrics.
* Distributed traces.
* Payment latency.
* Error rates.
* Kafka consumer lag.
* Database performance.
* Redis performance.
* Outbox backlog.
* Dead-letter queue size.

### G8. Failure Tolerance

The system must remain financially correct when components experience:

* Network failures.
* Request retries.
* Service crashes.
* Database transaction failures.
* Kafka delivery failures.
* Duplicate events.
* Payment rail timeouts.
* Temporary downstream failures.

Failure recovery must not create duplicate financial transactions or corrupt the ledger.

---

## 3. Non-Goals

The following are explicitly outside the scope of the initial FinFlow implementation.

### NG1. Real Financial Transactions

FinFlow will not process real money.

All transactions will use simulated accounts and a simulated payment rail.

### NG2. Real UPI or Banking Integration

The project will not initially integrate with:

* Banks.
* UPI production infrastructure.
* Credit/debit card networks.
* Real payment processors.

A simulated payment rail will be used to reproduce realistic success and failure scenarios.

### NG3. Regulatory Certification

FinFlow is an engineering project and will not claim:

* KYC certification.
* AML compliance certification.
* PCI DSS certification.
* Banking regulatory approval.
* Production financial-institution compliance.

Relevant security and compliance concepts may be modeled, but certification is outside the project scope.

### NG4. Production Fraud Detection

The initial risk engine will use deterministic rules.

The project will not claim to provide production-grade fraud detection or guarantee that fraudulent transactions can always be identified.

Machine-learning-based fraud detection may be explored later, but it is not required for the core system.

### NG5. Fully Autonomous Financial Authority

Agents will not receive unrestricted access to user funds.

Every transaction must pass through FinFlow's authorization and control layer.

The agent can propose an action, but the deterministic FinFlow infrastructure decides whether that action is permitted.

### NG6. Blockchain or Cryptocurrency

Blockchain-based settlement and cryptocurrency payments are outside the scope of the initial system.

### NG7. Production-Grade LLM Safety Certification

LLMs or AI agents may be used to generate payment intent, but FinFlow will not attempt to certify an AI model as safe.

The financial control layer must remain deterministic regardless of how the intent was generated.

## 4. Actors

FinFlow interacts with several actors that participate in the lifecycle of an agent-initiated financial transaction.

Actors represent entities that initiate actions, provide decisions, receive payments, or interact with FinFlow from outside the internal service architecture.

### 4.1 User

The User is the owner of the financial account and the principal who delegates authority to software agents.

The User can:

* Register and manage software agents.
* Create and configure delegation policies.
* Define transaction and spending limits.
* Restrict agents to specific merchants, beneficiaries, or transaction categories.
* Define approval thresholds for transactions.
* Approve or reject transactions requiring human authorization.
* Revoke agent credentials or delegated authority.
* View payment and transaction history.
* View audit records associated with their financial activity.

The User owns the financial authority being delegated.

**The User does not directly bypass FinFlow's authorization and financial-control mechanisms when interacting with the payment system.**

---

### 4.2 Agent

The Agent is a software entity that acts on behalf of a User.

An Agent may be an AI-powered application, autonomous software process, or other program authorized by the User.

The Agent can:

* Authenticate with FinFlow.
* Submit payment intents.
* Request payments on behalf of a User.
* Provide transaction context required for policy and risk evaluation.
* Receive payment authorization, rejection, or approval-required decisions.
* Query the status of transactions that it initiated.

The Agent cannot:

* Access unrestricted user funds.
* Modify its own delegation policy.
* Increase its own spending limits.
* Bypass policy evaluation.
* Bypass risk controls.
* Approve its own transactions when human approval is required.
* Directly modify financial ledger records.
* Directly execute settlement outside FinFlow's control layer.

The Agent proposes an action; FinFlow determines whether that action is permitted.

---

### 4.3 Merchant

The Merchant is the recipient of a payment initiated through FinFlow.

For the initial implementation, merchants are represented within the simulated payment environment.

A Merchant may provide:

* Merchant identity.
* Merchant category.
* Beneficiary/payment account information.
* Transaction-related metadata.

The Merchant does not determine whether an Agent is authorized to spend the User's funds.

FinFlow evaluates the transaction against the User's delegation policy and risk controls before allowing payment processing.

---

### 4.4 Payment Rail

The Payment Rail represents the external financial infrastructure responsible for executing or settling a payment.

In the initial implementation, the Payment Rail is simulated rather than connected to real banking or UPI infrastructure.

The simulated Payment Rail can produce outcomes such as:

* `SUCCESS`
* `TEMPORARY_FAILURE`
* `PERMANENT_FAILURE`
* `TIMEOUT`
* `UNKNOWN_RESULT`
* `DUPLICATE_RESPONSE`

The Payment Rail is intentionally modeled as an unreliable external dependency so that FinFlow can test retry behavior, idempotency, failure recovery, and unknown payment outcomes.

FinFlow must not assume that a payment request sent to the Payment Rail always produces an immediate or definitive response.

---

### 4.5 Administrator

The Administrator is an operational actor responsible for managing and monitoring the FinFlow platform.

The Administrator may:

* Monitor system health.
* Investigate operational failures.
* Review system-level audit information.
* Manage operational configuration.
* Monitor payment processing and infrastructure health.
* Investigate failed or suspicious system activity.

The Administrator must operate under explicitly defined permissions and must not automatically receive authority to impersonate Users or Agents.

Administrative actions that affect security, financial processing, or system configuration must be auditable.

---

### 4.6 Actor Boundaries

The fundamental authority relationships are:

```text
User
 │
 │ delegates limited authority
 ▼
Agent
 │
 │ proposes payment
 ▼
FinFlow Trust Layer
 │
 ├── Identity
 ├── Delegation Policy
 ├── Risk Evaluation
 └── Human Approval
 │
 │ authorized transaction
 ▼
Payment Engine
 │
 ▼
Payment Rail
 │
 ▼
Merchant
```

The critical security boundary is:

```text
Agent
   ↓
Can propose financial intent
   ↓
FinFlow Trust Layer
   ↓
Determines whether the intent is authorized
   ↓
Deterministic Payment Infrastructure
   ↓
Settlement
```

No Agent, Merchant, or Administrator should be able to bypass this trust boundary to directly modify financial state or execute an unauthorized payment.

## 5. Functional Requirements

Functional requirements define the capabilities and behaviors that FinFlow must provide.

Each requirement represents a system behavior that can be implemented, tested, and verified independently.

---

### 5.1 User Management

**FR-01: User Registration**

FinFlow shall allow a User to create and manage a financial account within the system.

The system shall maintain the User's identity and account ownership information.

**FR-02: User Authentication**

FinFlow shall authenticate Users before allowing access to protected operations.

**FR-03: User Account Management**

Authenticated Users shall be able to:

* View their account information.
* View their financial accounts.
* View payment history.
* View active Agents and delegation policies.
* View relevant audit information.

---

### 5.2 Agent Management

**FR-04: Agent Registration**

FinFlow shall allow a User to register a software Agent.

Each Agent shall have a unique identity associated with its owning User.

**FR-05: Agent Credentials**

FinFlow shall issue or register credentials that allow an Agent to authenticate with FinFlow.

Agent credentials shall support:

* Authentication.
* Expiration.
* Revocation.
* Rotation.

Credentials shall not provide unrestricted access to User funds.

**FR-06: Agent Lifecycle Management**

FinFlow shall support Agent lifecycle states including:

```text
ACTIVE
SUSPENDED
REVOKED
EXPIRED
```

A revoked or expired Agent shall not be permitted to initiate new authorized payments.

---

### 5.3 Delegation Policy Management

**FR-07: Policy Creation**

A User shall be able to create a Delegation Policy defining the financial authority granted to an Agent.

A policy may specify:

* Maximum transaction amount.
* Daily spending limit.
* Monthly spending limit.
* Allowed merchants.
* Allowed merchant categories.
* Allowed beneficiaries.
* Human approval threshold.
* Policy expiration.
* Policy status.

**FR-08: Policy Modification**

A User shall be able to modify an active Delegation Policy.

Policy changes shall be versioned so that historical payment decisions remain traceable to the policy version used at the time of authorization.

**FR-09: Policy Revocation**

A User shall be able to revoke a Delegation Policy.

A revoked policy shall not authorize new transactions.

---

### 5.4 Payment Intent

**FR-10: Payment Intent Creation**

An authenticated Agent shall be able to submit a Payment Intent on behalf of its User.

A Payment Intent shall contain sufficient information to evaluate the requested transaction, including:

* Agent identity.
* User identity.
* Merchant or beneficiary.
* Amount.
* Currency.
* Transaction category.
* Client-generated idempotency key.
* Relevant transaction context.

**FR-11: Payment Intent Validation**

FinFlow shall validate the Payment Intent before authorization.

Validation shall include:

* Agent authentication.
* Agent lifecycle status.
* Policy availability.
* Merchant or beneficiary validity.
* Amount validity.
* Currency validity.
* Required transaction metadata.

---

### 5.5 Authorization and Policy Evaluation

**FR-12: Delegated Authorization**

FinFlow shall determine whether an Agent is authorized to perform a requested transaction under the User's active Delegation Policy.

**FR-13: Policy Constraint Evaluation**

FinFlow shall evaluate applicable policy constraints, including:

* Per-transaction limits.
* Daily spending limits.
* Monthly spending limits.
* Merchant restrictions.
* Category restrictions.
* Beneficiary restrictions.
* Policy expiration.
* Approval thresholds.

A transaction violating a mandatory policy constraint shall be rejected.

**FR-14: Policy Version Tracking**

Every authorized or rejected payment decision shall reference the policy version used during evaluation.

This ensures that historical authorization decisions remain explainable even after a policy is modified.

---

### 5.6 Budget and Spending Control

**FR-15: Spending Capacity Validation**

FinFlow shall determine whether sufficient delegated spending capacity exists before authorizing a payment.

**FR-16: Atomic Budget Reservation**

FinFlow shall reserve the required spending capacity atomically to prevent concurrent payment requests from exceeding the configured spending limits.

The system shall maintain the invariant:

```text
Total Authorized Spending
    <=
Delegated Spending Limit
```

even when multiple payment requests are processed concurrently.

**FR-17: Budget Release**

If a payment is rejected or permanently fails after a spending reservation has been created, FinFlow shall release or reconcile the reserved spending capacity according to the payment state.

---

### 5.7 Risk Evaluation

**FR-18: Risk Assessment**

FinFlow shall evaluate payment requests using a deterministic Risk Engine.

The initial Risk Engine shall evaluate signals such as:

* Transaction amount.
* Transaction velocity.
* Merchant.
* Merchant category.
* Agent identity.
* Agent transaction history.
* Beneficiary.
* Transaction time.
* Previous payment failures.
* Policy violations.

**FR-19: Risk Decision**

The Risk Engine shall produce a risk decision:

```text
LOW
MEDIUM
HIGH
```

The corresponding system action shall be:

```text
LOW       → AUTO_APPROVE
MEDIUM    → REQUIRE_APPROVAL
HIGH      → BLOCK
```

Risk decisions shall be recorded for auditability.

---

### 5.8 Human Approval

**FR-20: Approval Request**

FinFlow shall create a Human Approval Request when a payment requires additional authorization.

**FR-21: Approval Decision**

An authorized User shall be able to:

* Approve the payment.
* Reject the payment.

**FR-22: Approval Expiration**

Approval Requests shall expire after a defined period.

An expired approval request shall not authorize the associated payment.

**FR-23: Approval Auditability**

The system shall record:

* Approval request creation.
* Approving or rejecting User.
* Decision.
* Decision timestamp.
* Associated payment.
* Relevant policy version.
* Risk decision.

---

### 5.9 Payment Processing

**FR-24: Payment Authorization**

A payment shall only enter the processing stage after successful identity, policy, budget, and risk checks, together with human approval when required.

**FR-25: Payment State Management**

FinFlow shall maintain an explicit Payment state machine.

The system shall support states including:

```text
CREATED
VALIDATING
AUTHORIZED
PROCESSING
COMPLETED
REJECTED
FAILED
EXPIRED
CANCELLED
REVERSED
UNKNOWN
```

Only valid state transitions shall be permitted.

**FR-26: Payment Attempt Tracking**

FinFlow shall maintain payment attempt information separately from the logical Payment.

A Payment may have multiple processing attempts due to retries or recoverable failures.

---

### 5.10 Idempotency

**FR-27: Idempotent Payment Requests**

FinFlow shall support idempotency keys for payment requests.

Repeated requests using the same valid idempotency key shall not create multiple financial transactions.

**FR-28: Idempotency Conflict Detection**

If an existing idempotency key is reused with a different request payload, FinFlow shall reject the request.

**FR-29: Idempotent Payment Processing**

Retries of payment-processing operations shall not create duplicate financial effects.

---

### 5.11 Financial Ledger

**FR-30: Double-Entry Ledger**

FinFlow shall maintain a double-entry financial ledger.

Every completed financial transaction shall produce balanced ledger entries.

The system shall maintain the invariant:

```text
Total Debits = Total Credits
```

**FR-31: Immutable Ledger Entries**

Posted ledger entries shall not be modified or deleted.

Corrections shall be represented using compensating or reversal entries.

**FR-32: Account Balance Integrity**

Account balances shall remain consistent with the authoritative ledger state.

---

### 5.12 Payment Settlement

**FR-33: Payment Rail Integration**

FinFlow shall communicate with a Payment Rail abstraction for payment settlement.

The initial implementation shall use a simulated Payment Rail.

**FR-34: Payment Rail Outcomes**

The simulated Payment Rail shall support outcomes including:

```text
SUCCESS
TEMPORARY_FAILURE
PERMANENT_FAILURE
TIMEOUT
UNKNOWN_RESULT
DUPLICATE_RESPONSE
```

**FR-35: Unknown Payment Outcomes**

FinFlow shall distinguish between a confirmed payment failure and an unknown payment outcome.

The system shall not blindly retry an unknown transaction in a manner that could result in duplicate financial settlement.

---

### 5.13 Event Processing

**FR-36: Domain Events**

FinFlow shall publish events for important state changes and business actions.

Examples include:

```text
PaymentCreated
PaymentAuthorized
PaymentRejected
PaymentProcessing
PaymentCompleted
PaymentFailed
PaymentReversed
RiskEvaluated
ApprovalRequested
ApprovalGranted
ApprovalRejected
AgentCreated
AgentRevoked
PolicyCreated
PolicyUpdated
PolicyRevoked
```

**FR-37: Transactional Event Publication**

Financial state changes and their corresponding outbox events shall be committed transactionally where required to prevent loss of committed events.

**FR-38: Event Consumption**

Consumers shall process events safely under at-least-once delivery semantics.

Duplicate events shall not produce duplicate financial effects.

---

### 5.14 Audit Trail

**FR-39: Audit Events**

FinFlow shall maintain an auditable record of security-sensitive and financially significant actions.

Audit events shall include information such as:

* Actor.
* Action.
* Resource.
* Decision.
* Reason.
* Policy version.
* Payment identifier.
* Correlation or trace identifier.
* Timestamp.

**FR-40: Payment Auditability**

A payment shall be traceable from its original Agent request through:

```text
Payment Intent
      ↓
Authorization
      ↓
Policy Decision
      ↓
Risk Decision
      ↓
Approval
      ↓
Payment Processing
      ↓
Ledger
      ↓
Settlement
```

---

### 5.15 Payment Status and History

**FR-41: Payment Status**

Users and Agents shall be able to retrieve the current status of payments they are authorized to view.

**FR-42: Payment History**

Users shall be able to retrieve historical payment information, including relevant state transitions and final outcomes.

---

### 5.16 Agent Revocation

**FR-43: Immediate Revocation Enforcement**

When an Agent or Delegation Policy is revoked, subsequent authorization attempts shall be rejected.

Previously authorized transactions shall be handled according to their current payment state and defined revocation policy.

**FR-44: Revocation Auditability**

Agent and policy revocations shall be recorded in the audit trail.

---

### 5.17 Operational Controls

**FR-45: Rate Limiting**

FinFlow shall enforce rate limits on relevant APIs to protect the system from excessive or abusive requests.

**FR-46: Request Correlation**

FinFlow shall assign or propagate correlation and trace identifiers so that a transaction can be followed across system components.

**FR-47: Health Monitoring**

FinFlow shall expose health and readiness information required to determine whether system components are operational and capable of processing requests.

## 6. Non-Functional Requirements

Non-functional requirements define the quality attributes and operational characteristics that FinFlow must maintain while performing its functional responsibilities.

Unlike functional requirements, these requirements describe how the system must behave in terms of correctness, security, reliability, performance, scalability, and operational visibility.

Where appropriate, requirements are classified as:

* **Hard Requirement:** Must always be satisfied.
* **Target:** A measurable objective to be established and validated through testing.
* **Design Principle:** An architectural guideline used to make implementation decisions.

---

### 6.1 Financial Correctness

**NFR-01: No Unauthorized Financial Effect**

**Classification:** Hard Requirement

No transaction shall create a financial effect unless the transaction has passed the required authorization and control checks.

The system shall not rely on the Agent to enforce financial authorization.

---

**NFR-02: Ledger Integrity**

**Classification:** Hard Requirement

Every completed financial transaction shall produce balanced ledger entries.

The system shall maintain:

```text
Total Debits = Total Credits
```

Ledger integrity must remain valid regardless of application retries or distributed component failures.

---

**NFR-03: Spending Limit Integrity**

**Classification:** Hard Requirement

Delegated spending limits shall not be exceeded, including when multiple payment requests are processed concurrently.

The system shall enforce spending constraints using authoritative transactional state.

---

### 6.2 Consistency

**NFR-04: Strong Consistency for Financial State**

**Classification:** Hard Requirement

The following state shall be maintained using authoritative transactional storage:

* Account balances.
* Ledger entries.
* Payment authorization state.
* Payment state.
* Spending reservations.
* Delegation policies where their current state affects authorization.

Financial correctness shall take precedence over eventual consistency.

---

**NFR-05: Eventual Consistency for Non-Critical Projections**

**Classification:** Design Principle

Non-critical derived information may be eventually consistent.

Examples include:

* Analytics.
* Notifications.
* Reporting projections.
* Monitoring dashboards.

Such eventual consistency must not affect the correctness of financial settlement or authorization.

---

### 6.3 Idempotency

**NFR-06: Duplicate Request Safety**

**Classification:** Hard Requirement

Retrying a client request shall not create duplicate financial effects.

The system shall support idempotent processing for operations where duplicate execution could produce financial or security consequences.

---

**NFR-07: Duplicate Event Safety**

**Classification:** Hard Requirement

Distributed consumers shall safely handle duplicate events.

Processing the same event more than once shall not create duplicate financial effects.

---

### 6.4 Reliability

**NFR-08: Failure Recovery**

**Classification:** Hard Requirement

FinFlow shall recover safely from transient failures involving:

* Network communication.
* Service crashes.
* Database transaction failures.
* Message broker failures.
* Payment rail failures.
* Consumer failures.

Recovery mechanisms must preserve financial correctness.

---

**NFR-09: No Lost Committed Financial Events**

**Classification:** Hard Requirement

Once a financial state change has been committed, the corresponding required domain event shall remain recoverable for publication.

The system shall use a reliable event-publication mechanism such as the Transactional Outbox pattern where appropriate.

---

**NFR-10: Failure Isolation**

**Classification:** Design Principle

Failure of a non-critical downstream component should not unnecessarily prevent critical financial operations.

The system should use appropriate isolation mechanisms such as:

* Timeouts.
* Retries.
* Backoff.
* Circuit breakers.
* Dead-letter queues.
* Bulkheads.

These mechanisms must not compromise financial correctness.

---

### 6.5 Security

**NFR-11: Authentication**

**Classification:** Hard Requirement

Protected operations shall require authenticated identities.

FinFlow shall distinguish between:

* User identity.
* Agent identity.
* Service identity.
* Administrative identity.

---

**NFR-12: Authorization**

**Classification:** Hard Requirement

Authentication alone shall not grant permission to perform financial operations.

Authorization shall evaluate the authenticated identity together with applicable delegation policies and transaction context.

---

**NFR-13: Least Privilege**

**Classification:** Design Principle

Users, Agents, services, and administrators shall receive only the permissions required to perform their responsibilities.

Agents shall not receive unrestricted access to financial accounts or ledger operations.

---

**NFR-14: Credential Protection**

**Classification:** Hard Requirement

Credentials and secrets shall not be stored in source code or committed to version control.

The system shall support appropriate mechanisms for:

* Credential expiration.
* Credential rotation.
* Credential revocation.
* Secret management.

---

**NFR-15: Auditability of Security-Sensitive Actions**

**Classification:** Hard Requirement

Security-sensitive actions such as authentication, authorization changes, Agent registration, policy changes, and Agent revocation shall be auditable.

---

### 6.6 Performance

**NFR-16: Predictable Request Latency**

**Classification:** Target

FinFlow should provide predictable latency for synchronous API operations.

Performance shall be evaluated using:

```text
p50 latency
p95 latency
p99 latency
error rate
throughput
```

Specific numerical latency targets shall be established after the initial architecture and workload characteristics are defined.

---

**NFR-17: Bounded Resource Usage**

**Classification:** Target

Services shall operate within defined limits for:

* CPU.
* Memory.
* Database connections.
* Kafka consumers.
* Redis connections.
* Network resources.

Resource behavior shall be measured under representative workloads.

---

### 6.7 Scalability

**NFR-18: Horizontal Scalability**

**Classification:** Design Principle

Stateless API and processing components should be capable of horizontal scaling where practical.

Scaling a service should not require modification of authoritative financial state.

---

**NFR-19: Independent Component Scaling**

**Classification:** Design Principle

Components with substantially different workloads should be capable of scaling independently.

For example:

```text
API Traffic
    ≠
Risk Evaluation Load
    ≠
Notification Load
    ≠
Analytics Load
```

The architecture should avoid forcing all workloads to scale together unnecessarily.

---

### 6.8 Availability

**NFR-20: Critical Path Availability**

**Classification:** Target

Critical payment-path components should remain available during normal operating conditions and recover gracefully from transient failures.

Availability targets shall be defined after the initial deployment architecture and workload are established.

---

**NFR-21: Graceful Degradation**

**Classification:** Design Principle

Non-critical functionality should degrade without corrupting or bypassing financial controls.

For example:

```text
Notification Service DOWN
        ↓
Payment may continue

Ledger / Authorization unavailable
        ↓
Payment must NOT bypass controls
```

The system shall fail closed for security- or correctness-critical decisions.

---

### 6.9 Auditability

**NFR-22: End-to-End Transaction Traceability**

**Classification:** Hard Requirement

A payment shall be traceable across its lifecycle:

```text
Agent Request
      ↓
Authentication
      ↓
Policy Evaluation
      ↓
Budget Reservation
      ↓
Risk Decision
      ↓
Human Approval
      ↓
Payment Processing
      ↓
Ledger
      ↓
Settlement
```

Not every payment will require every stage, but all applicable decisions must remain traceable.

---

**NFR-23: Historical Decision Reproducibility**

**Classification:** Hard Requirement

The system shall retain sufficient information to determine why an authorization decision was made.

Historical decisions shall reference relevant versions of:

* Delegation policies.
* Risk rules.
* Approval decisions.
* Payment state.

---

### 6.10 Observability

**NFR-24: Structured Logging**

**Classification:** Hard Requirement

Services shall produce structured logs containing sufficient contextual information to diagnose failures.

Logs should include identifiers such as:

* Request ID.
* Correlation ID.
* Trace ID.
* Payment ID.
* Agent ID.
* User ID where appropriate.

Sensitive credentials and secrets must not be logged.

---

**NFR-25: Metrics**

**Classification:** Hard Requirement

FinFlow shall expose operational metrics covering areas such as:

* Request throughput.
* Request latency.
* Error rates.
* Payment success/failure rates.
* Payment rejection rates.
* Risk evaluation latency.
* Database performance.
* Redis performance.
* Kafka consumer lag.
* Outbox backlog.
* Dead-letter queue size.

---

**NFR-26: Distributed Tracing**

**Classification:** Target

FinFlow should provide distributed traces for requests crossing multiple services.

A single transaction should be traceable across relevant components using a common trace context.

---

### 6.11 Maintainability

**NFR-27: Clear Service Ownership**

**Classification:** Design Principle

Each major domain capability should have a clearly defined owner.

Services should avoid directly modifying another service's authoritative database state.

---

**NFR-28: Versioned Contracts**

**Classification:** Hard Requirement

Externally exposed APIs and inter-service event contracts shall be versioned where compatibility requirements exist.

Changes to contracts should be explicit and reviewable.

---

**NFR-29: Architecture Documentation**

**Classification:** Design Principle

Significant architectural decisions shall be documented using Architecture Decision Records (ADRs).

Each ADR should capture:

* Context.
* Decision.
* Alternatives considered.
* Consequences.
* Trade-offs.

---

### 6.12 Testability

**NFR-30: Automated Verification**

**Classification:** Hard Requirement

Critical financial and authorization behavior shall be covered by automated tests.

Testing shall include:

* Unit tests.
* Integration tests.
* API tests.
* Contract tests.
* Concurrency tests.
* Failure tests.
* Security tests.
* Load tests.

---

**NFR-31: Invariant Verification**

**Classification:** Hard Requirement

Critical system invariants shall be directly tested.

Examples include:

```text
Unauthorized payments cannot complete.

Delegated spending limits cannot be exceeded.

Duplicate payment requests cannot create
duplicate financial effects.

Total Debits = Total Credits.

Revoked Agents cannot authorize new payments.

Duplicate events cannot create duplicate
financial effects.
```

---

### 6.13 Data Protection

**NFR-32: Data Minimization**

**Classification:** Design Principle

FinFlow shall store only the information required for system functionality, auditing, security, and operational requirements.

---

**NFR-33: Sensitive Data Protection**

**Classification:** Hard Requirement

Sensitive information shall be protected both during transmission and at rest where appropriate.

The system shall avoid exposing sensitive information through:

* Logs.
* Error responses.
* Metrics.
* Event payloads.
* Debug output.

---

### 6.14 NFR Priorities

When requirements conflict, FinFlow shall prioritize them in the following order:

```text
1. Financial Correctness
2. Security
3. Data Integrity
4. Reliability
5. Auditability
6. Availability
7. Performance
8. Scalability
9. Maintainability
```

The system shall not sacrifice financial correctness or authorization guarantees solely to improve latency, throughput, or availability.

---

### 6.15 Measurement Philosophy

FinFlow shall distinguish between requirements that must always hold and performance characteristics that must be measured.

The project shall not make unsupported claims such as:

```text
"Supports 10,000 TPS"
"99.99% availability"
"Sub-10ms payment processing"
```

unless those claims are supported by reproducible measurements and documented test conditions.

Performance and scalability claims shall be established through controlled benchmarking and load testing.

## 7. System Guarantees

System guarantees define the properties that FinFlow must preserve regardless of normal operating conditions, concurrent requests, retries, service failures, or distributed-system faults.

These guarantees are treated as system invariants. Architecture and implementation decisions must be evaluated against them.

A component or optimization must not weaken a financial or security guarantee.

---

### G1. No Unauthorized Payment

FinFlow shall never allow a payment to complete unless the requesting Agent has valid authority to perform the requested transaction.

A payment authorization decision must consider, where applicable:

* Agent identity.
* Agent lifecycle status.
* User ownership.
* Active delegation policy.
* Transaction amount.
* Merchant or beneficiary restrictions.
* Spending limits.
* Risk decision.
* Human approval requirements.

The Agent itself must not be trusted to determine whether it is authorized.

```text
Agent Request
      ↓
Identity Verification
      ↓
Delegated Authorization
      ↓
Policy Evaluation
      ↓
Risk / Approval
      ↓
Payment Authorization
```

---

### G2. Delegated Spending Limits Cannot Be Exceeded

FinFlow shall guarantee that an Agent cannot authorize transactions beyond the spending authority delegated to it.

For a configured spending limit:

```text
Authorized Spending
        <=
Delegated Spending Limit
```

This guarantee must hold even when multiple payment requests are processed concurrently.

For example:

```text
Daily Limit = ₹10,000

Payment A = ₹7,000
Payment B = ₹5,000

A + B = ₹12,000
```

The system must prevent both transactions from being authorized if doing so would violate the applicable limit.

The implementation must use authoritative transactional state rather than relying on application-level checks alone.

---

### G3. Payment Requests Are Idempotent

Repeated submission of the same payment request must not create multiple financial effects.

For a given idempotency key:

```text
First Request
     ↓
Payment Created
     ↓
Retry
     ↓
Existing Payment Returned
```

Reusing the same idempotency key with a different request payload must be rejected.

```text
Same Key
+
Different Request
=
Idempotency Conflict
```

---

### G4. No Duplicate Financial Effect

A payment must not produce duplicate financial effects because of:

* Client retries.
* Service retries.
* Network failures.
* Consumer retries.
* Duplicate messages.
* Payment-processing retries.

The logical Payment and its financial effects must remain distinguishable from individual processing attempts.

---

### G5. Ledger Integrity

Every completed financial transaction must produce balanced double-entry ledger records.

The fundamental invariant is:

```text
Total Debits = Total Credits
```

A transaction must not be considered financially complete if its corresponding ledger state is invalid or incomplete.

---

### G6. Ledger Immutability

Once financial ledger entries have been posted, they shall not be modified or deleted.

Corrections shall be represented using explicit compensating or reversal entries.

For example:

```text
Original Transaction
        ↓
Incorrect Entry
        ↓
Reversal Entry
        ↓
Corrected Transaction
```

The historical financial record must remain reconstructable.

---

### G7. Revoked Authority Cannot Authorize New Payments

Once an Agent or Delegation Policy has been revoked, the revoked authority must not be used to authorize new payment requests.

```text
Active Agent
     ↓
Payment Authorization
     ✓

Agent Revoked
     ↓
Payment Authorization
     ✗
```

Previously authorized payments must be handled according to their current state and the defined revocation policy.

Revocation must not silently alter historical transactions.

---

### G8. Committed Financial State Cannot Be Lost

Once a financial state change has been successfully committed, FinFlow must retain sufficient information to recover and continue processing the associated workflow.

For example:

```text
Payment State Updated
        +
Required Outbox Event Created
        ↓
      COMMIT
```

If Kafka or another downstream system becomes unavailable after the commit, the event must remain recoverable.

The system must not rely on a best-effort sequence such as:

```text
Database Commit
      ↓
Kafka Publish
```

when failure between the two operations could result in a lost event.

---

### G9. Duplicate Events Must Be Safe

FinFlow shall assume that distributed events may be delivered more than once.

Therefore:

```text
Event A
   ↓
Consumer
   ↓
Processing

Event A again
   ↓
Consumer
   ↓
No duplicate financial effect
```

Consumers must use appropriate idempotency or deduplication mechanisms.

At-least-once event delivery must therefore be compatible with financial correctness.

---

### G10. Valid Payment State Transitions

A Payment shall only transition between explicitly permitted states.

For example:

```text
CREATED
   ↓
VALIDATING
   ↓
AUTHORIZED
   ↓
PROCESSING
   ↓
COMPLETED
```

Invalid transitions must be rejected.

For example:

```text
COMPLETED
   ↓
PROCESSING
```

must not be allowed.

Exceptional states such as:

```text
REJECTED
FAILED
EXPIRED
CANCELLED
UNKNOWN
REVERSED
```

must have explicitly defined transition rules.

---

### G11. Unknown Payment Outcomes Must Not Be Treated as Failures

If FinFlow sends a payment request to an external Payment Rail and receives an ambiguous result such as a timeout, the system must distinguish:

```text
Confirmed Failure
```

from:

```text
Unknown Outcome
```

An unknown outcome must not automatically be treated as safe to retry.

The system must first determine the appropriate recovery or reconciliation behavior.

This prevents a retry from potentially creating a duplicate financial settlement.

---

### G12. Authorization Decisions Must Be Traceable

Every financially significant authorization decision must be traceable to the information used to make that decision.

A historical decision should allow the system to identify:

```text
User
 ↓
Agent
 ↓
Payment
 ↓
Policy Version
 ↓
Risk Decision
 ↓
Approval Decision
 ↓
Payment State
 ↓
Ledger
 ↓
Settlement
```

Relevant identifiers such as Payment ID, Agent ID, Policy Version, Event ID, and Trace ID should be retained where appropriate.

---

### G13. Financial State Takes Precedence Over Non-Critical Features

Failures in non-critical components must not cause FinFlow to bypass financial or authorization controls.

For example:

```text
Notification Service DOWN
        ↓
Payment may continue if safe

Authorization Service DOWN
        ↓
Payment must NOT bypass authorization

Ledger unavailable
        ↓
Payment must NOT bypass ledger requirements
```

The system shall fail closed when required to preserve security or financial correctness.

---

### G14. Authoritative State Has a Single Source of Truth

Financial state must have a clearly defined authoritative source.

For the initial architecture:

```text
PostgreSQL
     ↓
Authoritative Financial State
```

Caches, message brokers, projections, and derived data stores must not independently become authoritative sources of financial truth.

For example:

```text
Redis
  ≠
Financial Source of Truth

Kafka
  ≠
Financial Source of Truth
```

These systems may support processing, caching, or propagation of state but must not override authoritative financial records.

---

### G15. Financial Invariants Must Be Testable

Every critical guarantee must have corresponding automated verification.

Examples:

```text
Concurrent payments
        ↓
Spending limit remains valid

Duplicate request
        ↓
One financial effect

Duplicate event
        ↓
One financial effect

Completed payment
        ↓
Balanced ledger

Revoked agent
        ↓
Authorization rejected
```

The guarantees defined in this section are therefore not merely documentation. They form the basis for FinFlow's unit, integration, concurrency, failure, and property-based tests.

---

### 7.1 Guarantee Priority

When two system objectives conflict, guarantees shall be prioritized in the following order:

```text
1. Financial Correctness
2. Authorization and Security
3. Ledger Integrity
4. Data Consistency
5. Reliability
6. Auditability
7. Availability
8. Performance
9. Scalability
```

FinFlow shall not sacrifice financial correctness or authorization guarantees merely to improve performance or availability.

---

### 7.2 Engineering Principle

Every major architectural decision in FinFlow must answer:

> **Which system guarantee does this design preserve, and what failure scenario could violate it?**

Technology choices such as PostgreSQL, Kafka, Redis, or Kubernetes are implementation decisions.

The guarantees defined above are the constraints that those technologies and architectural patterns must satisfy.

## 8. Domain Model

The domain model defines the core business concepts required to implement FinFlow's functional requirements and system guarantees.

The domain model is intentionally separated from the database schema. Entities described here represent business concepts and their relationships. Physical tables, indexes, foreign keys, partitioning, and storage-specific decisions will be defined during database design.

The domain is organized into three conceptual areas:

1. **Identity & Authority Domain** - represents users, agents, credentials, and delegated authority.
2. **Payment & Financial Domain** - represents payment intent, execution, attempts, accounts, and financial records.
3. **Control & Reliability Domain** - represents risk decisions, approvals, idempotency, auditing, and reliable event publication.

### 8.1 Domain Model Overview

```text
                              ┌──────────────┐
                              │     User     │
                              └──────┬───────┘
                                     │
                         delegates authority
                                     │
                                     ▼
                          ┌──────────────────┐
                          │ DelegationPolicy │
                          └────────┬─────────┘
                                   │
                                   ▼
                              ┌─────────┐
                              │  Agent  │
                              └────┬────┘
                                   │
                              authenticated
                                   │
                                   ▼
                         ┌───────────────────┐
                         │ AgentCredential   │
                         └───────────────────┘


 Agent
   │
   │ creates
   ▼
┌──────────────────┐
│  PaymentIntent   │
└────────┬─────────┘
         │
         │ accepted for execution
         ▼
┌──────────────────┐
│     Payment      │
└───┬────────┬─────┘
    │        │
    │        ├──────────────────────┐
    │        │                      │
    ▼        ▼                      ▼
Risk      Approval            PaymentAttempt
Assessment Request                  │
                                    │
                                    ▼
                              Payment Rail


 Payment
    │
    ▼
┌────────────────────┐
│ Financial Records  │
│                    │
│ LedgerAccount      │
│ LedgerEntry        │
└────────────────────┘


 Payment
    │
    ├───────────────► AuditEvent
    │
    └───────────────► OutboxEvent
                              │
                              ▼
                            Kafka


 Payment Request
       │
       ▼
┌──────────────────────┐
│ IdempotencyRecord    │
└──────────────────────┘
```

---

### 8.2 User

**User** represents the human or principal that owns financial authority within FinFlow.

A User can:

* create and manage agents
* delegate authority to agents
* create and manage financial accounts
* approve transactions requiring human authorization
* revoke or suspend agents
* inspect payment and audit history

Conceptual attributes:

```text
User
----
id
name
email
status
created_at
updated_at
```

The User is the source of delegated authority but does not directly represent an automated payment actor.

---

### 8.3 Agent

**Agent** represents software acting on behalf of a User.

An Agent may initiate payment intents but cannot independently obtain unrestricted authority over the user's financial resources.

Conceptual attributes:

```text
Agent
-----
id
user_id
name
status
created_at
updated_at
```

Possible lifecycle states:

```text
ACTIVE
SUSPENDED
REVOKED
```

The Agent's authority is constrained by one or more `DelegationPolicy` objects.

This separation supports:

* G1 - No unauthorized payment
* G7 - Revoked authority cannot authorize new payments

An Agent is therefore an **actor**, not an autonomous settlement authority.

---

### 8.4 Agent Credential

**AgentCredential** represents the credentials used to authenticate an Agent.

Conceptual attributes:

```text
AgentCredential
---------------
id
agent_id
credential_type
credential_hash
status
expires_at
created_at
revoked_at
```

Credentials must not be stored as plaintext secrets.

The exact authentication mechanism and cryptographic implementation will be defined during the security and authentication design.

Relationship:

```text
Agent 1 ──────── * AgentCredential
```

An Agent may have multiple credentials to support credential rotation, expiration, and revocation.

---

### 8.5 Delegation Policy

**DelegationPolicy** represents the authority explicitly delegated by a User to an Agent.

A policy defines the boundaries within which an Agent may perform financial operations.

Example:

```text
Agent: GroceryAgent

Maximum spending:
₹10,000 / day

Allowed category:
GROCERIES

Allowed currency:
INR

Approval threshold:
₹5,000

Expiration:
30 September 2026
```

Conceptual policy attributes include:

```text
DelegationPolicy
----------------
id
agent_id
status
currency
amount_limit
time_window
allowed_categories
allowed_merchants
approval_threshold
effective_from
expires_at
created_at
updated_at
```

A policy may constrain:

* maximum transaction amount
* cumulative spending over a time period
* currency
* merchant
* merchant category
* transaction type
* expiration
* human approval requirements

Authorization is therefore not simply:

```text
Agent has permission
```

but rather:

```text
Agent
+ requested amount
+ merchant
+ category
+ currency
+ time
+ active delegation policy
        ↓
Authorization decision
```

This makes `DelegationPolicy` a central component of FinFlow's trust model.

---

### 8.6 Account

**Account** represents a financial account participating in payment operations.

Conceptual attributes:

```text
Account
-------
id
owner_id
currency
status
created_at
updated_at
```

Examples include:

```text
User Account
Merchant Account
FinFlow Settlement Account
```

An Account represents the financial entity whose balance is affected by transactions.

The accounting representation of these balances is maintained through the ledger model described below.

---

### 8.7 Merchant

**Merchant** represents a recipient participating in a payment transaction.

Conceptual attributes:

```text
Merchant
--------
id
name
category
status
settlement_account_id
created_at
updated_at
```

Merchant attributes may be used by:

* delegation policies
* authorization rules
* risk evaluation
* transaction classification

For example, a delegation policy may permit an Agent to transact only with merchants belonging to the `GROCERIES` category.

---

### 8.8 Beneficiary

**Beneficiary** represents a payment destination.

A Beneficiary may represent a Merchant or another supported destination type.

Conceptual attributes:

```text
Beneficiary
-----------
id
type
merchant_id
account_reference
status
created_at
updated_at
```

The distinction between `Merchant` and `Beneficiary` is maintained at the domain level because a payment destination does not necessarily have to represent a merchant.

The exact beneficiary scope for the first implementation remains a design decision and should not introduce unnecessary complexity into the initial payment flow.

---

### 8.9 Payment Intent

**PaymentIntent** represents the request or intention to make a payment.

It captures what the Agent is asking FinFlow to do before the payment is executed.

Example:

```text
Agent A requests:

Amount: ₹7,000
Currency: INR
Destination: Merchant X
Purpose: Groceries
```

Conceptual attributes:

```text
PaymentIntent
-------------
id
agent_id
beneficiary_id
amount
currency
purpose
idempotency_key
status
created_at
updated_at
```

The Payment Intent is not itself proof that money has moved.

It represents:

> "This payment has been requested."

The intent passes through the control pipeline before execution:

```text
PaymentIntent
      │
      ▼
Authentication
      │
      ▼
Authorization / Policy Evaluation
      │
      ▼
Budget Reservation
      │
      ▼
Risk Assessment
      │
      ▼
Human Approval (if required)
      │
      ▼
Payment Execution
```

---

### 8.10 Payment

**Payment** represents the logical financial operation created from an accepted payment intent.

Conceptual attributes:

```text
Payment
-------
id
payment_intent_id
amount
currency
status
created_at
updated_at
```

A Payment is distinct from a Payment Attempt.

The distinction is:

```text
Payment
= logical financial operation

PaymentAttempt
= individual interaction with the payment rail
```

A Payment may therefore have multiple attempts.

Relationship:

```text
PaymentIntent 1 ─────── 0..1 Payment
Payment       1 ─────── * PaymentAttempt
```

The exact cardinality between `PaymentIntent` and `Payment` will be finalized when the payment state machine and retry semantics are specified.

---

### 8.11 Payment Attempt

**PaymentAttempt** represents an individual attempt to execute a Payment against the payment rail.

A single Payment may produce multiple attempts.

Example:

```text
Payment P1

Attempt #1 → TIMEOUT
Attempt #2 → UNKNOWN
Attempt #3 → SUCCESS
```

Conceptual attributes:

```text
PaymentAttempt
--------------
id
payment_id
attempt_number
rail_reference
status
failure_code
started_at
completed_at
created_at
```

This separation is required because an external payment interaction can fail independently from the logical Payment.

In particular:

```text
TIMEOUT ≠ FAILED
```

A timeout may mean that FinFlow does not know whether the external rail processed the transaction.

Therefore an attempt may enter an `UNKNOWN` state and require reconciliation or status inquiry before a final financial outcome is established.

This directly supports:

* G4 - No duplicate financial effect
* G11 - Unknown payment outcome must not be treated as failure

---

### 8.12 Risk Assessment

**RiskAssessment** represents the result of evaluating a Payment or PaymentIntent against deterministic risk rules.

Conceptual attributes:

```text
RiskAssessment
--------------
id
payment_id
risk_score
decision
rules_triggered
created_at
```

Possible decisions include:

```text
LOW_RISK
HIGH_RISK
REVIEW
BLOCK
```

The Risk Engine provides a control decision but does not directly modify financial balances or settle payments.

The conceptual separation is:

```text
Risk Engine
    │
    │ produces decision
    ▼
Control Layer
    │
    │ permits or blocks execution
    ▼
Payment Engine
```

This preserves the separation between probabilistic or analytical decision-making and deterministic financial execution.

---

### 8.13 Approval Request

**ApprovalRequest** represents a human authorization requirement triggered by policy or risk controls.

Example:

```text
Delegation Policy:

Transactions > ₹5,000
require human approval.

Requested payment:

₹7,000

Result:

PENDING_APPROVAL
```

Conceptual attributes:

```text
ApprovalRequest
---------------
id
payment_id
requested_from
status
reason
created_at
resolved_at
```

Possible states:

```text
PENDING
APPROVED
REJECTED
EXPIRED
```

The approval mechanism is part of the control layer and must complete before execution when the applicable policy requires human approval.

---

### 8.14 Ledger Account

**LedgerAccount** represents the accounting account used by the double-entry ledger.

Conceptually:

```text
Account
   │
   ▼
LedgerAccount
   │
   ▼
LedgerEntry
```

Examples include:

```text
User Ledger Account
Merchant Ledger Account
Settlement Ledger Account
Fee Ledger Account
```

The exact relationship between the operational `Account` and `LedgerAccount` will be finalized during database and accounting design.

The model should avoid unnecessary duplication if the two concepts can safely be represented by the same underlying entity.

---

### 8.15 Ledger Entry

**LedgerEntry** represents one side of a financial accounting transaction.

A financial transaction consists of at least two entries.

Example:

```text
Transaction T1

Debit:
    User Account       ₹7,000

Credit:
    Merchant Account   ₹7,000
```

Conceptual attributes:

```text
LedgerEntry
-----------
id
transaction_id
ledger_account_id
entry_type
amount
currency
created_at
```

The fundamental invariant is:

```text
Σ Debits = Σ Credits
```

Ledger entries are append-oriented and should not be edited to rewrite financial history.

Corrections should be represented using compensating or reversal entries.

Example:

```text
Original:
    User      -₹7,000
    Merchant  +₹7,000

Reversal:
    User      +₹7,000
    Merchant  -₹7,000
```

This supports:

* G5 - Ledger integrity
* G6 - Ledger immutability

---

### 8.16 Audit Event

**AuditEvent** represents a significant action, decision, or state change that must be traceable.

Example audit trail:

```text
Agent authenticated
        ↓
Policy evaluated
        ↓
Budget reserved
        ↓
Risk approved
        ↓
Human approved
        ↓
Payment submitted
        ↓
Payment succeeded
```

Conceptual attributes:

```text
AuditEvent
----------
id
actor_type
actor_id
action
resource_type
resource_id
metadata
timestamp
```

Audit events allow the system to reconstruct why a payment was allowed, denied, approved, or transitioned between important states.

This directly supports:

* G12 - Authorization decisions must be traceable
* auditability requirements

Audit records should not be treated as the authoritative financial state.

---

### 8.17 Outbox Event

**OutboxEvent** represents a domain event that must be reliably published to downstream systems.

Example:

```text
PaymentSucceeded
PaymentFailed
PaymentCreated
PaymentApproved
```

Conceptual attributes:

```text
OutboxEvent
-----------
id
event_type
aggregate_type
aggregate_id
payload
status
created_at
published_at
```

The Outbox Event is persisted in the same database transaction as the authoritative state change.

Example:

```text
BEGIN

create Payment
create OutboxEvent(PaymentCreated)

COMMIT
```

A separate publisher can then publish the event to Kafka.

This prevents the following inconsistent state:

```text
Database:
Payment created ✓

Kafka:
Event not published ✗
```

`OutboxEvent` therefore supports reliable event propagation without making Kafka the source of truth.

---

### 8.18 Idempotency Record

**IdempotencyRecord** represents the system's record of a previously processed idempotent request.

Conceptual attributes:

```text
IdempotencyRecord
----------------
idempotency_key
request_hash
response_reference
status
created_at
```

For example:

```text
Request 1:

Idempotency-Key: abc123
Amount: ₹5,000

        ↓

Payment P123 created
```

If the same request is submitted again:

```text
Idempotency-Key: abc123
Amount: ₹5,000

        ↓

Return existing Payment P123
```

However:

```text
Idempotency-Key: abc123
Amount: ₹50,000
```

must not silently reuse the previous result.

The system should detect that the same key was used for a different request and return a conflict.

The invariant is therefore:

```text
Same key + same request
        → same logical operation

Same key + different request
        → conflict
```

This supports:

* G3 - Payment requests are idempotent
* G4 - No duplicate financial effect

---

## 8.19 Domain Relationships

The primary relationships are:

```text
User
 │
 ├────────────── * Agent
 │                    │
 │                    ├──────── * AgentCredential
 │                    │
 │                    └──────── * DelegationPolicy
 │
 └────────────── * Account


Agent
 │
 └────────────── * PaymentIntent
                       │
                       └──────── 0..1 Payment
                                      │
                                      ├──────── * PaymentAttempt
                                      │
                                      ├──────── * RiskAssessment
                                      │
                                      └──────── * ApprovalRequest


Payment
 │
 ├────────────── * LedgerEntry
 │
 ├────────────── * AuditEvent
 │
 └────────────── * OutboxEvent


Payment Request
 │
 └────────────── 0..1 IdempotencyRecord
```

These cardinalities are conceptual and will be validated during the detailed schema design.

---

## 8.20 Domain Separation

The domain model can be grouped into three major areas.

### Identity & Authority

```text
User
Agent
AgentCredential
DelegationPolicy
```

Responsibility:

> Establish who is acting and what authority has been delegated.

### Payment & Financial Core

```text
Account
Beneficiary
Merchant
PaymentIntent
Payment
PaymentAttempt
LedgerAccount
LedgerEntry
```

Responsibility:

> Represent and execute financially meaningful operations while preserving financial correctness.

### Control & Reliability

```text
RiskAssessment
ApprovalRequest
IdempotencyRecord
AuditEvent
OutboxEvent
```

Responsibility:

> Enforce safety controls, survive retries and failures, and maintain traceability.

---

## 8.21 Domain Invariants

The domain model must preserve the following invariants.

### Authorization

```text
An Agent cannot initiate an authorized payment
outside its active DelegationPolicy.
```

### Revocation

```text
A revoked or suspended Agent cannot authorize
new payment operations.
```

### Spending Limit

```text
Concurrent payment requests must not collectively
exceed the applicable delegated spending limit.
```

### Ledger Balance

```text
For every financial transaction:

Total Debits = Total Credits
```

### Ledger Immutability

```text
Historical ledger entries are not modified to
rewrite financial history.

Corrections use compensating entries.
```

### Idempotency

```text
A repeated request with the same idempotency key
must not create another financial effect.
```

### Payment Attempts

```text
A Payment represents the logical operation.

PaymentAttempts represent individual external
execution attempts.
```

### Unknown Outcome

```text
A timeout or communication failure must not
automatically be interpreted as payment failure.
```

### State Transitions

```text
Payment state transitions must follow the
defined payment state machine.

Invalid transitions are rejected.
```

### Source of Truth

```text
PostgreSQL remains authoritative for financial
and security-sensitive state.

Redis and Kafka do not become authoritative
financial stores.
```

---

## 8.22 Design Principles

The domain model follows these principles:

1. **Separate intent from execution.**
   A request to pay is not the same as an executed payment.

2. **Separate logical payments from external attempts.**
   A single payment may require multiple interactions with the payment rail.

3. **Separate authorization from settlement.**
   Permission to execute a transaction is distinct from actually executing it.

4. **Separate financial history from derived state.**
   The ledger provides the authoritative historical record.

5. **Treat financial invariants as first-class domain rules.**
   Correctness must not depend solely on application conventions.

6. **Model uncertainty explicitly.**
   External timeouts can produce unknown outcomes rather than assumed failures.

7. **Make retries safe.**
   Payment requests and event consumers must tolerate duplicates.

8. **Keep the financial core deterministic.**
   Risk and agent behavior may be complex, but the final authorization and settlement path must obey explicit rules.

9. **Avoid premature entity proliferation.**
   Domain concepts should exist because they represent meaningful business behavior, not merely because they might eventually become database tables.

---

## 8.23 Open Design Questions

The following decisions should remain open until the detailed design phase:

1. Whether `Account` and `LedgerAccount` should remain separate entities or share an underlying representation.
2. Whether `Beneficiary` is necessary for the first implementation or can initially be represented through `Merchant`.
3. Exact cardinality between `PaymentIntent` and `Payment`.
4. Whether policies are versioned as immutable revisions or updated in place with historical audit records.
5. Exact representation of spending windows and budget reservations.
6. Exact payment state machine and legal state transitions.
7. Whether risk assessments belong to `PaymentIntent`, `Payment`, or both.
8. Exact representation of multi-currency accounting.
9. Exact ledger transaction grouping model.
10. Retention and archival policies for audit and idempotency records.

These questions should be resolved through detailed design and documented as Architecture Decision Records where the decision has significant architectural consequences.

---

## 8.24 Domain Model Summary

The FinFlow domain model establishes the following core flow:

```text
User
  │
  │ delegates bounded authority
  ▼
Agent
  │
  │ authenticated using
  ▼
AgentCredential
  │
  │ constrained by
  ▼
DelegationPolicy
  │
  │ initiates
  ▼
PaymentIntent
  │
  │ passes control checks
  ├────────► Authorization
  ├────────► Budget Reservation
  ├────────► Risk Assessment
  └────────► Human Approval
                    │
                    ▼
                 Payment
                    │
                    ▼
             PaymentAttempt
                    │
                    ▼
              Payment Rail
                    │
                    ▼
                Settlement
                    │
                    ▼
              Ledger Entries
                    │
                    ├────────► AuditEvent
                    │
                    └────────► OutboxEvent
```

The domain model therefore provides the conceptual foundation for the next design stages:

```text
Requirements
     ↓
System Guarantees
     ↓
Domain Model          ← Current section
     ↓
Domain Relationships
     ↓
State Machines
     ↓
Data Model / Schema
     ↓
Service Boundaries
     ↓
API Contracts
     ↓
Implementation
```

## 9. Payment Cycle

The Payment Cycle defines the complete lifecycle of an agent-initiated financial transaction, from the initial request through authorization, risk evaluation, execution, settlement, and final financial recording.

The cycle is designed around FinFlow's central architectural principle:

> **The Agent proposes intent. The deterministic control layer decides whether the intent is permitted. The payment engine executes the authorized operation. The ledger records the resulting financial effect.**

The payment cycle must preserve the system guarantees defined in Section 7, particularly authorization correctness, concurrent spending limits, idempotency, financial integrity, safe handling of unknown outcomes, and traceability.

---

### 9.1 High-Level Payment Flow

```text
Agent
  │
  │ 1. Submit Payment Intent
  ▼
API / Payment Service
  │
  │ 2. Authenticate Agent
  ▼
Authorization Layer
  │
  │ 3. Evaluate Delegation Policy
  ▼
Budget Manager
  │
  │ 4. Reserve Spending Capacity
  ▼
Risk Engine
  │
  │ 5. Evaluate Transaction Risk
  ▼
Approval Engine
  │
  │ 6. Human Approval if Required
  ▼
Payment Engine
  │
  │ 7. Execute Payment
  ▼
Payment Rail
  │
  │ 8. Return Outcome
  ▼
Payment Engine
  │
  ├── SUCCESS
  ├── FAILURE
  └── UNKNOWN
  │
  ▼
Settlement / Financial Recording
  │
  ▼
Ledger
  │
  ├──────────────► Audit Trail
  │
  └──────────────► Domain Event / Outbox
```

Not every payment necessarily requires human approval. The approval stage is conditional on the applicable delegation policy and risk controls.

---

## 9.2 Stage 1: Payment Intent Creation

The Agent begins by submitting a payment request.

Example:

```text
Agent: GroceryAgent

Amount: ₹7,000
Currency: INR
Merchant: Merchant X
Purpose: Grocery purchase
Idempotency-Key: abc123
```

FinFlow creates or retrieves the corresponding `PaymentIntent`.

The request must include sufficient information for the control layer to determine whether the transaction is authorized.

Conceptually:

```text
PaymentIntent
-------------
agent
beneficiary
amount
currency
purpose
idempotency_key
```

At this stage:

```text
Intent ≠ Executed Payment
```

The system has only recorded the requested operation.

---

## 9.3 Stage 2: Agent Authentication

FinFlow first establishes the identity of the requesting Agent.

```text
Request
   │
   ▼
Agent Credential
   │
   ▼
Authenticated Agent
```

The system verifies:

* credential validity
* credential status
* credential expiration
* Agent status
* request integrity

If authentication fails, the payment must not proceed.

```text
Authentication Failed
        │
        ▼
Payment Rejected
```

Authentication answers:

> **Who is making this request?**

It does not answer whether the Agent is permitted to perform the requested transaction.

---

## 9.4 Stage 3: Delegation Policy Evaluation

After authentication, FinFlow determines whether the authenticated Agent has sufficient delegated authority.

The policy evaluation considers relevant transaction attributes such as:

```text
Agent
Amount
Currency
Merchant
Merchant Category
Transaction Type
Current Time
Policy Status
Policy Expiration
```

Example policy:

```text
Agent: GroceryAgent

Daily limit: ₹10,000
Allowed category: GROCERIES
Currency: INR
Approval required above: ₹5,000
```

Requested transaction:

```text
₹7,000
GROCERIES
INR
```

The policy may produce:

```text
ALLOW
DENY
REQUIRE_APPROVAL
```

The authorization decision must be recorded in a traceable manner.

---

## 9.5 Stage 4: Budget Reservation

Authorization of an individual transaction is not sufficient when multiple requests can execute concurrently.

Example:

```text
Daily delegated limit = ₹10,000

Request A = ₹7,000
Request B = ₹6,000
```

If both requests independently observe:

```text
Available = ₹10,000
```

both may be approved even though:

```text
₹7,000 + ₹6,000 = ₹13,000
```

which violates the delegated limit.

Therefore FinFlow must reserve spending capacity atomically with appropriate concurrency control.

Conceptually:

```text
Current available budget
        │
        ▼
Atomic reservation
        │
        ▼
Reserved amount
        │
        ▼
Remaining available budget
```

For example:

```text
Initial limit:       ₹10,000
Existing reserved:   ₹2,000
Available:           ₹8,000

New request:         ₹7,000

Reservation succeeds

Remaining:           ₹1,000
```

The reservation mechanism must ensure that concurrent requests cannot collectively exceed the applicable spending limit.

The exact implementation using database transactions, row-level locking, optimistic concurrency, or another mechanism will be determined during the detailed concurrency design.

---

## 9.6 Stage 5: Risk Assessment

After authorization and budget validation, the transaction is evaluated by the Risk Engine.

The Risk Engine may evaluate factors such as:

```text
Transaction amount
Merchant
Merchant category
Agent behavior
Transaction frequency
Velocity
Historical activity
Policy violations
```

The result is represented by a `RiskAssessment`.

Example:

```text
RiskAssessment
--------------
Decision: LOW_RISK
Score: 12
Rules triggered: none
```

Possible outcomes:

```text
LOW_RISK
REVIEW
BLOCK
```

Risk evaluation does not itself settle or modify the financial ledger.

It produces a control decision that influences whether execution may proceed.

---

## 9.7 Stage 6: Human Approval

Some transactions may require explicit human approval.

For example:

```text
Delegation Policy:

Transactions > ₹5,000
require human approval.
```

Requested transaction:

```text
₹7,000
```

Therefore:

```text
Payment
   │
   ▼
Approval Required
   │
   ▼
PENDING_APPROVAL
```

The user may then:

```text
APPROVE
REJECT
```

If approved:

```text
Approval
   │
   ▼
Payment Execution
```

If rejected:

```text
Approval
   │
   ▼
Payment Rejected
```

The approval decision must be associated with the relevant payment and recorded in the audit trail.

---

## 9.8 Stage 7: Payment Creation and Execution

Once all required control checks have passed, the Payment Engine can execute the logical payment.

Conceptually:

```text
PaymentIntent
      │
      ▼
Authorized
      │
      ▼
Payment
      │
      ▼
PROCESSING
```

The Payment Engine is responsible for interacting with the payment rail.

The Agent does not directly communicate with the settlement mechanism.

```text
Agent
  │
  ✗
  │ direct settlement access
  │
  └──────────── not allowed

Agent
  │
  ▼
FinFlow Control Layer
  │
  ▼
Payment Engine
  │
  ▼
Payment Rail
```

This maintains the separation between agent intent and financial execution.

---

## 9.9 Stage 8: Payment Attempt

Each interaction with the payment rail is represented as a `PaymentAttempt`.

Example:

```text
Payment P123

Attempt #1
    ↓
Payment Rail
    ↓
TIMEOUT

Attempt #2
    ↓
Payment Rail
    ↓
SUCCESS
```

The Payment remains the logical financial operation while individual attempts represent external execution interactions.

This allows FinFlow to distinguish:

```text
Logical Payment
        │
        ├── Attempt 1
        ├── Attempt 2
        └── Attempt 3
```

rather than treating every retry as a new payment.

---

## 9.10 Stage 9: Handling Payment Outcomes

The payment rail may return three broad categories of outcomes.

### Success

The payment rail confirms that the transaction succeeded.

```text
PROCESSING
    │
    ▼
SUCCEEDED
```

The system can proceed with final financial recording and downstream processing.

---

### Failure

The payment rail definitively confirms that the transaction failed and no financial effect occurred.

```text
PROCESSING
    │
    ▼
FAILED
```

The appropriate reservation and control state can then be resolved according to the failure semantics.

---

### Unknown

The system cannot determine whether the payment succeeded.

Example:

```text
FinFlow
   │
   │ submit payment
   ▼
Payment Rail
   │
   │ processes request
   │
   X──── network connection lost
```

FinFlow receives:

```text
TIMEOUT
```

But a timeout does not prove that the payment failed.

Therefore:

```text
TIMEOUT
   ≠
FAILED
```

Instead:

```text
PROCESSING
    │
    ▼
UNKNOWN
```

The transaction may then require:

* payment status inquiry
* reconciliation
* rail-side reference lookup
* controlled recovery

A payment with an unknown outcome must not be blindly retried in a way that could produce duplicate financial effects.

---

## 9.11 Stage 10: Reconciliation

When a payment enters `UNKNOWN`, FinFlow must eventually determine its actual financial outcome.

Conceptually:

```text
UNKNOWN
   │
   ▼
Reconciliation
   │
   ├── confirmed success
   │
   └── confirmed failure
```

Example:

```text
Payment Attempt
      │
      ▼
UNKNOWN
      │
      ▼
Query Payment Rail
      │
      ▼
SUCCESS
```

The reconciliation mechanism is particularly important because external systems can produce ambiguous outcomes.

The exact reconciliation strategy and retry schedule will be defined during the payment reliability design.

---

## 9.12 Stage 11: Settlement

Once the external payment outcome is definitively established as successful, FinFlow records the corresponding financial effect.

Conceptually:

```text
Payment Succeeded
       │
       ▼
Settlement
       │
       ▼
Ledger Transaction
```

The financial recording must preserve the double-entry invariant:

```text
Total Debits = Total Credits
```

For example:

```text
Transaction T123

Debit:
    User Account       ₹7,000

Credit:
    Merchant Account   ₹7,000
```

The exact settlement semantics depend on whether the simulated rail is modeled as:

* immediate settlement
* asynchronous settlement
* authorization followed by later capture
* another explicitly defined model

The first implementation should select one model and document it rather than attempting to simulate every real-world payment behavior.

---

## 9.13 Stage 12: Ledger Recording

The ledger records the financial effect of the completed transaction.

A successful payment produces the corresponding accounting entries.

```text
Payment
   │
   ▼
Ledger Transaction
   │
   ├── Debit Entry
   │
   └── Credit Entry
```

The ledger is authoritative for historical financial records.

Ledger entries should not be edited to rewrite history.

If a financial correction is required:

```text
Original Transaction
        │
        ▼
Compensating / Reversal Transaction
```

This preserves the historical audit trail.

---

## 9.14 Stage 13: Event Publication

Important state changes generate domain events.

Example:

```text
PaymentCreated
PaymentAuthorized
PaymentApproved
PaymentProcessing
PaymentSucceeded
PaymentFailed
PaymentUnknown
```

The event is first persisted through the transactional outbox mechanism.

Conceptually:

```text
Database Transaction
        │
        ├── Update Payment
        ├── Write Ledger
        └── Write OutboxEvent
                 │
                 ▼
              COMMIT
                 │
                 ▼
          Outbox Publisher
                 │
                 ▼
               Kafka
```

This prevents the authoritative database state and event stream from diverging because of a failure between separate operations.

---

## 9.15 Stage 14: Audit Recording

Important decisions and transitions are recorded in the audit trail.

For example:

```text
Agent authenticated
       ↓
Policy evaluated
       ↓
Budget reserved
       ↓
Risk approved
       ↓
Human approved
       ↓
Payment submitted
       ↓
Payment succeeded
       ↓
Ledger recorded
```

The audit trail allows the system to answer:

```text
Who initiated the payment?
Which Agent acted?
Which policy authorized it?
What risk decision was produced?
Was human approval required?
Who approved it?
Which payment attempt was executed?
What was the rail outcome?
What ledger transaction recorded the financial effect?
```

This is essential for traceability and debugging.

---

## 9.16 Successful Payment Cycle

The complete successful path is:

```text
1. Agent
      │
      ▼
2. Payment Intent
      │
      ▼
3. Authentication
      │
      ▼
4. Policy Evaluation
      │
      ▼
5. Budget Reservation
      │
      ▼
6. Risk Assessment
      │
      ▼
7. Human Approval (if required)
      │
      ▼
8. Payment Created
      │
      ▼
9. Payment Attempt
      │
      ▼
10. Payment Rail
      │
      ▼
11. SUCCESS
      │
      ▼
12. Settlement
      │
      ▼
13. Ledger Entries
      │
      ├──────────► Audit Event
      │
      └──────────► Outbox Event
```

---

## 9.17 Failed Payment Cycle

A definitively failed payment follows:

```text
Payment
   │
   ▼
PaymentAttempt
   │
   ▼
Payment Rail
   │
   ▼
FAILED
   │
   ├── resolve reservation
   ├── update payment state
   ├── record failure
   ├── create audit event
   └── publish domain event
```

A failed payment must not create a successful financial effect.

The exact handling of reserved budget depends on where the failure occurs and will be specified in the payment state and budget reservation design.

---

## 9.18 Unknown Payment Cycle

An unknown outcome follows a different path:

```text
Payment
   │
   ▼
PaymentAttempt
   │
   ▼
Payment Rail
   │
   ▼
TIMEOUT / CONNECTION FAILURE
   │
   ▼
UNKNOWN
   │
   ▼
Reconciliation
   │
   ├───────────────┐
   │               │
   ▼               ▼
SUCCESS          FAILURE
   │               │
   ▼               ▼
Settlement      Resolve
   │            Reservation
   ▼
Ledger
```

The system must not interpret an unknown outcome as a definitive failure.

---

## 9.19 Payment Cycle Invariants

The payment cycle must preserve the following invariants.

### Authorization invariant

```text
A payment cannot enter execution unless
the Agent has sufficient active delegated authority.
```

### Budget invariant

```text
The sum of concurrent reservations must not
exceed the applicable delegated spending limit.
```

### Approval invariant

```text
If policy requires human approval, payment execution
cannot proceed until the required approval is obtained.
```

### Idempotency invariant

```text
A repeated payment request with the same idempotency key
must not create another logical financial operation.
```

### Execution invariant

```text
A Payment may have multiple PaymentAttempts,
but retries must not create multiple logical Payments.
```

### Unknown-outcome invariant

```text
An unknown external outcome must not be treated as
a definitive failure.
```

### Ledger invariant

```text
Every completed financial transaction must satisfy:

Total Debits = Total Credits
```

### State-transition invariant

```text
Payment states may only transition through
explicitly defined legal transitions.
```

### Audit invariant

```text
Security-sensitive authorization and payment decisions
must be traceable through audit records.
```

---

## 9.20 Payment Cycle and System Guarantees

| Payment Cycle Stage | Primary Guarantees |
| ------------------- | ------------------ |
| Authentication      | G1, G7             |
| Policy Evaluation   | G1, G7             |
| Budget Reservation  | G2                 |
| Idempotency Check   | G3, G4             |
| Payment Execution   | G4, G10, G11       |
| Reconciliation      | G4, G11            |
| Settlement          | G5                 |
| Ledger Recording    | G5, G6             |
| Outbox Publication  | G8, G9             |
| Audit Recording     | G12                |
| Failure Handling    | G13                |
| PostgreSQL State    | G14                |

The payment cycle therefore provides the operational path through which the guarantees defined earlier are enforced.

---

## 9.21 Core Architectural Principle

The payment cycle intentionally separates three responsibilities:

```text
┌───────────────────────────────────────────────┐
│                INTENT / ORCHESTRATION         │
│                                               │
│ Agent request                                 │
│ Payment Intent                                │
└───────────────────────┬───────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────┐
│              CONTROL / AUTHORIZATION          │
│                                               │
│ Authentication                               │
│ Delegation Policy                             │
│ Budget Reservation                            │
│ Risk Assessment                               │
│ Human Approval                                │
└───────────────────────┬───────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────┐
│                 SETTLEMENT                    │
│                                               │
│ Payment Engine                                │
│ Payment Attempt                               │
│ Payment Rail                                  │
│ Settlement                                    │
│ Ledger                                        │
└───────────────────────────────────────────────┘
```

The Agent may influence the **intent**, but it does not directly control the **settlement**.

The Control Layer acts as the deterministic trust boundary between the two.

This separation is fundamental to FinFlow's architecture and should remain intact as the system evolves toward distributed service boundaries.


## 10. Agent Authorization

Agent Authorization defines how FinFlow determines whether an authenticated software Agent is permitted to perform a requested financial operation on behalf of a User.

Authorization is a deterministic control performed between Agent intent and payment execution.

The core principle is:

> **Authentication establishes who the Agent is. Authorization determines whether that Agent is permitted to perform the requested operation.**

An Agent never receives unrestricted authority over a User's financial resources. All authority is explicitly delegated by the User and constrained by one or more active `DelegationPolicy` objects.

---

### 10.1 Authorization Model

FinFlow uses a policy-based authorization model.

The authorization decision is based on the combination of:

```text id="l1b7pz"
Agent Identity
      +
Delegated Authority
      +
Requested Operation
      +
Transaction Attributes
      +
Current Policy State
      +
Applicable Security Controls
      ↓
Authorization Decision
```

The decision must be deterministic and reproducible for the same relevant inputs.

Possible outcomes are:

```text id="j7wq6s"
ALLOW
DENY
REQUIRE_APPROVAL
```

The authorization layer must not execute the payment itself.

Its responsibility is to determine:

> **"Is this operation permitted to proceed?"**

---

## 10.2 Authentication vs Authorization

These concepts must remain separate.

### Authentication

Answers:

```text id="j6i6tr"
Who is making this request?
```

Example:

```text id="1c7i2v"
Credential
   ↓
Agent A
```

### Authorization

Answers:

```text id="d7c9k3"
Is Agent A allowed to perform this operation?
```

Example:

```text id="y5oxjv"
Agent A
   ↓
Requested: ₹7,000
   ↓
Delegation Policy
   ↓
ALLOW
```

Therefore:

```text id="8b2gq1"
Authentication = Identity
Authorization  = Permission
```

A successfully authenticated Agent can still be denied authorization.

---

## 10.3 Delegated Authority

The User explicitly delegates a bounded set of permissions to an Agent.

Example:

```text id="2y4fsk"
User
 │
 │ delegates
 ▼
Agent: GroceryAgent

Policy:
────────────────────────
Maximum: ₹10,000/day
Currency: INR
Category: GROCERIES
Approval threshold: ₹5,000
Expires: 30 Sep 2026
```

The Agent's effective authority is therefore:

```text id="d8oywr"
Agent Authority
    =
Delegated Policy Constraints
```

The Agent cannot extend its own authority.

For example, if the policy allows:

```text id="b11m3c"
₹10,000/day
```

the Agent cannot request:

```text id="hmn6o9"
₹50,000
```

and expect authorization simply because its credential is valid.

---

## 10.4 Policy-Based Authorization

Authorization is evaluated using transaction attributes rather than only static roles.

Relevant attributes may include:

```text id="a4t6u3"
Agent
User
Amount
Currency
Merchant
Merchant Category
Transaction Type
Time
Policy Status
Policy Expiration
Current Spending
Risk Decision
Approval Requirement
```

Conceptually:

```text id="qz9lpu"
authorize(
    agent,
    operation,
    amount,
    currency,
    beneficiary,
    merchant,
    time,
    applicable_policy
)
```

returns:

```text id="z4v9op"
ALLOW
DENY
REQUIRE_APPROVAL
```

This is closer to **attribute-based / policy-based authorization** than simple role-based access control.

---

## 10.5 Why RBAC Alone Is Insufficient

Role-Based Access Control can answer questions such as:

```text id="w6p5a4"
Does Agent A have permission:
"create_payment"?
```

But FinFlow needs to answer more specific questions:

```text id="7ax1g2"
Can Agent A:

spend ₹7,000?
spend it today?
spend it on this merchant?
spend it in INR?
spend it on this category?
spend it after reaching its daily limit?
spend it without human approval?
```

These decisions depend on transaction attributes.

Therefore RBAC may be used for coarse-grained application permissions, while delegation policies provide fine-grained financial authorization.

Conceptually:

```text id="8j7bq0"
RBAC
  ↓
Can Agent access payment operation?

Policy Authorization
  ↓
Can Agent perform THIS specific payment?
```

---

## 10.6 Authorization Evaluation Pipeline

The authorization process follows a deterministic sequence.

```text id="1u4u6d"
Incoming Request
      │
      ▼
Authenticate Agent
      │
      ▼
Validate Agent Status
      │
      ▼
Load Active Delegation Policy
      │
      ▼
Validate Operation
      │
      ▼
Evaluate Amount Limits
      │
      ▼
Evaluate Spending Limits
      │
      ▼
Evaluate Merchant / Category Rules
      │
      ▼
Evaluate Currency / Time Constraints
      │
      ▼
Determine Approval Requirement
      │
      ▼
Authorization Decision
```

A request failing a mandatory authorization check must not proceed to payment execution.

---

## 10.7 Agent Status

An Agent's lifecycle state affects authorization.

Possible states:

```text id="p6c7v8"
ACTIVE
SUSPENDED
REVOKED
```

### ACTIVE

The Agent may perform operations subject to its policies.

```text id="7s5v0k"
ACTIVE
  ↓
Policy Evaluation
  ↓
Continue
```

### SUSPENDED

New financial operations are not authorized.

Existing operations may follow their own state-machine rules.

```text id="d8t5jg"
SUSPENDED
  ↓
New Payment
  ↓
DENY
```

### REVOKED

The Agent's delegated authority is permanently invalidated.

```text id="uh2t8n"
REVOKED
  ↓
New Payment
  ↓
DENY
```

This directly supports:

> **G7: Revoked authority cannot authorize new payments.**

The exact handling of payments that were already processing when an Agent was suspended or revoked is a separate state-management decision.

---

## 10.8 Policy Lifecycle

Delegation policies have their own lifecycle.

Conceptually:

```text id="5y2n0g"
CREATED
   │
   ▼
ACTIVE
   │
   ├──────────────► EXPIRED
   │
   ├──────────────► REVOKED
   │
   └──────────────► SUPERSEDED
```

An authorization decision must only use an applicable policy.

A policy that is:

```text id="quq4z4"
expired
revoked
inactive
```

must not authorize a new payment.

---

## 10.9 Amount Authorization

The requested amount must be within the applicable transaction-level limit.

Example:

```text id="90pvqf"
Policy:
Maximum transaction = ₹5,000

Request:
₹4,000

→ Allowed
```

But:

```text id="h1s0w6"
Policy:
Maximum transaction = ₹5,000

Request:
₹7,000

→ Denied or approval-controlled,
depending on the policy definition
```

The authorization model must distinguish between:

```text id="1aj9x4"
Transaction limit
```

and:

```text id="x5kqf2"
Cumulative spending limit
```

A transaction can individually satisfy the transaction limit while still exceeding the Agent's remaining delegated budget.

---

## 10.10 Cumulative Spending Authorization

Suppose:

```text id="v3y7x8"
Daily limit = ₹10,000

Already reserved/spent = ₹8,000

Requested = ₹4,000
```

Although:

```text id="9b4c2n"
₹4,000 < ₹10,000
```

the request cannot be authorized because:

```text id="5p8n0r"
₹8,000 + ₹4,000 = ₹12,000
```

which exceeds the cumulative limit.

Therefore authorization must consider the Agent's current financial usage.

The check must be concurrency-safe.

A simple:

```text id="u7h4lq"
read balance
→ check limit
→ update balance
```

sequence is insufficient under concurrent requests.

The detailed implementation will be defined in the Budget Reservation and Concurrency Control design.

---

## 10.11 Merchant and Category Restrictions

Delegation policies may restrict where an Agent can spend.

Example:

```text id="qj5x3s"
Allowed categories:
    GROCERIES
    PHARMACY
```

Request:

```text id="0c7b8e"
Merchant Category:
    GROCERIES

→ ALLOW
```

Request:

```text id="4p1s7q"
Merchant Category:
    ELECTRONICS

→ DENY
```

The policy may optionally restrict specific merchants:

```text id="k9r5hx"
Allowed merchants:
    Merchant A
    Merchant B
    Merchant C
```

The exact level of merchant restriction will be finalized during policy schema design.

---

## 10.12 Currency Restrictions

A policy may constrain the currencies in which an Agent may transact.

Example:

```text id="x7w3bq"
Policy:
Allowed currency = INR
```

Request:

```text id="q6d1nm"
INR → permitted
USD → denied
```

Currency restrictions are important because financial authority should be explicit rather than inferred from the Agent's default account.

Multi-currency accounting is outside the initial implementation scope unless required by a later design decision.

---

## 10.13 Time-Based Restrictions

Policies may include temporal constraints.

Example:

```text id="q9d8f2"
Agent may spend:
09:00 - 18:00
Monday - Friday
```

A request at:

```text id="h4v7x2"
23:30 Saturday
```

would be denied.

Policies may also have an absolute expiration:

```text id="d1o9wp"
Effective:
01 Sep 2026

Expires:
30 Sep 2026
```

An expired policy cannot authorize new transactions.

Time evaluation must use a consistent timezone and clearly defined temporal semantics.

---

## 10.14 Approval Thresholds

Authorization may determine that a transaction requires human approval rather than immediately allowing execution.

Example:

```text id="5qj7t4"
Policy:

≤ ₹5,000
    → automatic execution

> ₹5,000
    → human approval required
```

Request:

```text id="p8m2xa"
₹4,000
```

Result:

```text id="6l0q3y"
ALLOW
```

Request:

```text id="e4n8qs"
₹7,000
```

Result:

```text id="4j8w1m"
REQUIRE_APPROVAL
```

The distinction is important:

```text id="2k9h3f"
DENY
```

means the operation cannot proceed.

Whereas:

```text id="6r2q8w"
REQUIRE_APPROVAL
```

means the operation may proceed if the required human control is successfully completed.

---

## 10.15 Authorization Decision

The authorization engine produces a structured decision.

Conceptually:

```text id="9y5k1z"
AuthorizationDecision
--------------------
decision
reason_codes
policy_id
policy_version
agent_id
request_id
evaluated_at
```

Possible decisions:

```text id="v3r7c2"
ALLOW
DENY
REQUIRE_APPROVAL
```

The decision should contain sufficient information to explain why it was produced without exposing sensitive information unnecessarily.

Example:

```text id="4h6x0q"
Decision:
    DENY

Reason:
    DAILY_LIMIT_EXCEEDED

Policy:
    policy_123

Agent:
    agent_456
```

---

## 10.16 Authorization Traceability

Every security-sensitive authorization decision must be traceable.

For a payment request, FinFlow should be able to reconstruct:

```text id="2b0s6q"
Agent Identity
      │
      ▼
Credential
      │
      ▼
Delegation Policy
      │
      ▼
Policy Version
      │
      ▼
Transaction Attributes
      │
      ▼
Authorization Decision
      │
      ▼
Reason Codes
```

This information should be associated with the relevant audit trail.

The goal is to answer:

> **"Why was this payment allowed or denied?"**

without relying on transient application logs alone.

---

## 10.17 Policy Versioning

Authorization decisions should be associated with the specific policy version that was evaluated.

Consider:

```text id="v5p8yd"
Policy v1:
Daily limit = ₹10,000
```

Later:

```text id="s7j1kq"
Policy v2:
Daily limit = ₹5,000
```

A historical payment authorized under v1 should remain explainable according to v1.

Therefore policy changes should not make historical authorization decisions impossible to reconstruct.

The preferred design is to treat policy versions as immutable once they have participated in an authorization decision.

The exact storage and lifecycle mechanism will be finalized during policy persistence design.

---

## 10.18 Revocation Semantics

Revocation must take effect for new authorization decisions.

Example:

```text id="6x9z0j"
Agent A
   │
   ▼
ACTIVE
   │
   ▼
Policy allows payment
```

User revokes Agent A:

```text id="f4j2q8"
Agent A
   ↓
REVOKED
```

A new payment request must result in:

```text id="m6s1r9"
DENY
```

Revocation must not rely on stale authorization information from caches.

This is particularly important because Redis is not the authoritative source of financial or security-sensitive state.

The authoritative state remains in PostgreSQL.

---

## 10.19 Authorization and Caching

Authorization decisions may eventually benefit from caching, but cached authorization data must not compromise security guarantees.

Potentially cacheable information includes:

```text id="k2n8fw"
Policy metadata
Agent metadata
Non-sensitive configuration
```

However, revocation and other security-critical state changes create cache invalidation requirements.

Therefore:

```text id="9z2c4k"
Cache
  ≠
Authoritative Authorization State
```

The exact caching strategy will be determined after performance requirements and consistency requirements are measured.

---

## 10.20 Authorization Failure Behavior

FinFlow must fail closed when authorization cannot be established safely.

For example:

```text id="r3x8vm"
Authorization Service unavailable
        │
        ▼
Cannot establish permission
        │
        ▼
Do NOT execute payment
```

The system must never interpret:

```text id="4t5n8c"
authorization unavailable
```

as:

```text id="7y6m2p"
authorization granted
```

This supports:

> **G13: Non-critical failures cannot bypass financial or security controls.**

Availability is subordinate to authorization correctness.

---

## 10.21 Authorization Example

Consider:

```text id="r7x9k2"
Agent:
    GroceryAgent

Policy:
    Daily limit: ₹10,000
    Transaction limit: ₹5,000
    Allowed category: GROCERIES
    Currency: INR
    Approval threshold: ₹3,000
```

Request:

```text id="1k4s8v"
Amount: ₹2,500
Currency: INR
Category: GROCERIES
```

Evaluation:

```text id="w9n5b1"
Agent active?              YES
Policy active?             YES
Transaction limit valid?   YES
Daily budget available?    YES
Currency allowed?          YES
Category allowed?          YES
Approval required?         NO

                ↓

            ALLOW
```

Another request:

```text id="2q8m4x"
Amount: ₹4,000
Currency: INR
Category: GROCERIES
```

Evaluation:

```text id="7x1n6c"
Agent active?              YES
Policy active?             YES
Transaction limit valid?   YES
Daily budget available?    YES
Currency allowed?          YES
Category allowed?          YES
Approval required?         YES

                ↓

       REQUIRE_APPROVAL
```

Another request:

```text id="4v7p2z"
Amount: ₹4,000
Currency: INR
Category: ELECTRONICS
```

Evaluation:

```text id="2m9x5k"
Category allowed?
        NO

        ↓

      DENY
```

---

## 10.22 Authorization and the Payment Cycle

Agent Authorization occupies the control stage of the payment cycle.

```text id="1g6r9v"
Agent
  │
  ▼
Payment Intent
  │
  ▼
Authentication
  │
  ▼
┌──────────────────────────────┐
│      AUTHORIZATION           │
│                              │
│ Agent Status                 │
│ Delegation Policy            │
│ Transaction Limits           │
│ Spending Limits              │
│ Merchant Restrictions        │
│ Currency Restrictions        │
│ Time Restrictions            │
│ Approval Thresholds          │
└──────────────┬───────────────┘
               │
       ┌───────┼─────────┐
       ▼       ▼         ▼
     ALLOW    DENY   REQUIRE_APPROVAL
       │                 │
       │                 ▼
       │            Human Approval
       │                 │
       └────────┬────────┘
                ▼
         Payment Execution
```

The authorization layer is therefore the primary trust boundary between agent-generated intent and financial execution.

---

## 10.23 Authorization Invariants

The following invariants must hold.

### A1. No implicit authority

```text id="v7k3p9"
An Agent has no financial authority unless
that authority has been explicitly delegated.
```

### A2. Active policy required

```text id="w2m8q4"
A payment requires an applicable active
DelegationPolicy.
```

### A3. Policy constraints cannot be bypassed

```text id="p6x1r8"
An Agent cannot authorize an operation outside
the constraints of its applicable policy.
```

### A4. Revocation takes effect for new requests

```text id="z5n9c2"
A revoked or suspended Agent cannot authorize
new financial operations.
```

### A5. Concurrent limits are enforced atomically

```text id="q8m3v6"
Concurrent requests must not collectively
exceed the delegated spending limit.
```

### A6. Approval requirements are enforced

```text id="r4y7k1"
A transaction requiring human approval cannot
enter execution until approval is completed.
```

### A7. Authorization failure is fail-closed

```text id="n3c8w5"
If authorization cannot be established safely,
the payment must not execute.
```

### A8. Decisions are traceable

```text id="h6p2x9"
Every security-sensitive authorization decision
must be associated with sufficient information
to reconstruct the decision.
```

---

## 10.24 Authorization Design Boundary

The Authorization Engine is responsible for:

```text id="a8v2m5"
✓ Agent identity validation result
✓ Agent status
✓ Policy selection
✓ Policy evaluation
✓ Transaction constraints
✓ Spending authority
✓ Approval requirement
✓ Authorization decision
✓ Decision reason
```

It is not responsible for:

```text id="e1q7s4"
✗ Moving money
✗ Writing financial ledger entries
✗ Calling the payment rail directly
✗ Performing settlement
✗ Making autonomous financial decisions
```

The separation is intentional:

```text id="b5m9k3"
Agent
  ↓
Intent
  ↓
Authorization
  ↓
Payment Engine
  ↓
Settlement
```

The Agent proposes.

The Authorization Layer controls.

The Payment Engine executes.

The Ledger records.

---

## 10.25 Design Principles

1. **Authentication and authorization remain separate.**
2. **Authority must always be explicitly delegated.**
3. **Authorization is policy-based rather than solely role-based.**
4. **Financial authorization must consider transaction-specific attributes.**
5. **Spending limits must be enforced under concurrency.**
6. **Revocation must prevent new unauthorized operations.**
7. **Authorization failures must fail closed.**
8. **Authorization decisions must be explainable and auditable.**
9. **Historical decisions must remain reproducible through policy versioning.**
10. **Authorization must never directly perform settlement.**
11. **Caches must not become authoritative sources for security-sensitive state.**
12. **The authorization layer must remain deterministic.**

---

## 10.26 Open Design Questions

The following decisions remain open for detailed design:

1. Exact policy expression format.
2. Whether policies are represented as structured database fields, a policy DSL, or both.
3. Policy precedence when multiple policies apply.
4. How overlapping policies are combined.
5. Whether explicit deny rules override allow rules.
6. Exact budget reservation mechanism.
7. Whether spending limits are calculated from settled transactions, active reservations, or both.
8. Policy versioning and effective-date semantics.
9. Exact revocation propagation mechanism.
10. Authorization decision persistence requirements.
11. Authorization cache strategy.
12. Maximum complexity permitted in policy evaluation.
13. Handling of policies changed while payments are already processing.
14. Whether risk decisions are inputs to authorization or a separate control stage after authorization.

These decisions should be resolved before implementing the Authorization Service and documented through ADRs where appropriate.

---

## 10.27 Summary

FinFlow's Agent Authorization model establishes a bounded authority relationship:

```text id="t9m4x7"
User
  │
  │ delegates
  ▼
Agent
  │
  │ operates under
  ▼
DelegationPolicy
  │
  │ evaluates
  ▼
Payment Request
  │
  ▼
Authorization Engine
  │
  ├── ALLOW
  ├── DENY
  └── REQUIRE_APPROVAL
```

The resulting decision determines whether the payment may proceed, but authorization itself does not execute the financial transaction.

This maintains FinFlow's central trust boundary:

```text id="y8p3q1"
Probabilistic / Agent Layer
          │
          │ Intent
          ▼
Deterministic Control Layer
          │
          │ Authorized operation
          ▼
Deterministic Settlement Layer
          │
          ▼
Financial Ledger
```

The Agent therefore receives **bounded delegated authority**, not direct control over settlement.

## 11. Risk Decision Model

## 12. Human Approval Model

## 13. Financial Ledger Model

## 14. Consistency Model

## 15. Failure Scenarios

## 16. Security and Threat Model

## 17. High-Level Architecture

## 18. Service Boundaries

## 19. API Boundaries

## 20. Event Boundaries

## 21. Data Ownership

## 22. Observability

## 23. Architecture Decision Records