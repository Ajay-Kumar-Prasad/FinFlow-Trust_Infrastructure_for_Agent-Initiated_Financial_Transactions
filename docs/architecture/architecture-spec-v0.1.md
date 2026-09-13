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

The Risk Decision Model defines how FinFlow evaluates the risk associated with an otherwise authorized payment before allowing it to proceed toward execution.

The Risk Engine is a deterministic control component. It evaluates transaction and contextual attributes against explicitly defined risk rules and produces a structured `RiskAssessment`.

The core principle is:

> **Authorization determines whether the Agent has permission to perform an operation. Risk evaluation determines whether the authorized operation should be permitted based on transaction risk controls.**

Risk evaluation does not grant authority and does not execute payments.

---

### 11.1 Risk Model Overview

The risk pipeline is:

```text id="4k6m8p"
Payment Intent
      │
      ▼
Authorization
      │
      │ authorized
      ▼
Risk Evaluation
      │
      ├── Transaction attributes
      ├── Agent context
      ├── Merchant context
      ├── Velocity
      ├── Historical behavior
      └── Risk rules
      │
      ▼
Risk Assessment
      │
      ├── ALLOW
      ├── REVIEW
      └── BLOCK
      │
      ▼
Control Decision
      │
      ▼
Payment Execution
```

The Risk Engine operates only after the system has established that the Agent is authorized to perform the requested operation.

---

## 11.2 Authorization vs Risk

Authorization and risk are intentionally separate.

### Authorization

Answers:

> **"Is the Agent allowed to perform this transaction?"**

Example:

```text id="z8q3y1"
Agent:
    GroceryAgent

Policy:
    Maximum = ₹10,000/day
    Category = GROCERIES

Request:
    ₹4,000 groceries

Authorization:
    ALLOW
```

### Risk

Answers:

> **"Does this authorized transaction satisfy FinFlow's risk controls?"**

For example:

```text id="p6w2r9"
Authorization:
    ALLOW

Risk:
    BLOCK

Reason:
    Excessive transaction velocity
```

Therefore:

```text id="v4n8s2"
Authentication
      ↓
Authorization
      ↓
Risk
      ↓
Approval
      ↓
Execution
```

A transaction must not proceed merely because authorization succeeded.

---

## 11.3 Risk Assessment

`RiskAssessment` represents the result of evaluating a payment against the configured risk rules.

Conceptual attributes:

```text id="c5m9x7"
RiskAssessment
--------------
id
payment_id
decision
risk_score
rules_triggered
evaluation_version
created_at
```

Possible decisions:

```text id="q2h7k4"
ALLOW
REVIEW
BLOCK
```

The exact fields are conceptual and will be finalized during the data-model design.

---

## 11.4 Risk Inputs

The Risk Engine may consider several classes of input.

### Transaction attributes

```text id="x8r3m5"
Amount
Currency
Merchant
Merchant Category
Transaction Type
Timestamp
```

### Agent attributes

```text id="n7k2p4"
Agent identity
Agent status
Agent transaction history
Agent transaction frequency
Agent recent spending
```

### User / account context

```text id="s4v9j6"
Account status
Recent account activity
Available financial capacity
```

### Merchant context

```text id="a6y1q8"
Merchant category
Merchant status
Historical interaction
```

### Velocity

Velocity represents the rate at which transactions occur over a period.

Example:

```text id="m3w7k9"
Agent makes:

1 transaction in 10 minutes
        ↓
normal

50 transactions in 10 minutes
        ↓
potentially suspicious
```

Velocity rules can operate over:

```text id="r8p2v5"
Number of transactions
Total transaction amount
Time window
Merchant diversity
Category diversity
```

The exact risk signals will be defined incrementally rather than attempting to build a complete fraud-detection system in v0.1.

---

## 11.5 Deterministic Risk Rules

The initial Risk Engine will use deterministic rules.

Example:

```text id="h5q9w2"
RULE R001

IF
    transaction_amount > ₹50,000

THEN
    decision = REVIEW
```

Another example:

```text id="k7m3x8"
RULE R002

IF
    transaction_count_last_5_minutes > 20

THEN
    decision = BLOCK
```

Another:

```text id="p4y8n6"
RULE R003

IF
    merchant_category = HIGH_RISK
    AND transaction_amount > ₹10,000

THEN
    decision = REVIEW
```

The important property is that the same input state and same rule version should produce the same result.

---

## 11.6 Rule Evaluation

A transaction may trigger multiple rules.

Example:

```text id="v9x4k2"
Payment:
    Amount = ₹25,000
    Category = ELECTRONICS
    Velocity = 15 transactions / 5 minutes

Triggered Rules:
    R002 - High velocity
    R004 - Large transaction
```

The Risk Engine records the triggered rules.

```text id="j6p8w3"
RiskAssessment:

decision = REVIEW

rules_triggered:
    R002
    R004
```

This provides an explanation for the decision.

The engine should not simply return:

```text id="s3k7m1"
risk_score = 87
```

without being able to explain how that score or decision was produced.

---

## 11.7 Risk Score

A numerical risk score may be used to aggregate risk signals.

Example:

```text id="q8m5v2"
Risk Score

0 ──────────────── 100
│                   │
Low Risk         High Risk
```

A conceptual mapping could be:

```text id="r3x7n9"
0 - 30   → ALLOW
31 - 70  → REVIEW
71 - 100 → BLOCK
```

These thresholds are **illustrative design values**, not fixed project requirements.

The initial implementation should not assume that a numerical score is necessary.

A purely rule-based decision model may be preferable initially:

```text id="y5k2p8"
Rules
  ↓
Triggered Conditions
  ↓
Decision
```

A score can be introduced later if it provides measurable value.

---

## 11.8 Risk Decision Precedence

Multiple rules may produce different outcomes.

For example:

```text id="n6w3r8"
Rule A → ALLOW
Rule B → REVIEW
Rule C → BLOCK
```

FinFlow needs deterministic precedence.

A proposed precedence is:

```text id="c9p4x7"
BLOCK
  >
REVIEW
  >
ALLOW
```

Therefore:

```text id="z2m8k5"
Any BLOCK rule
      ↓
BLOCK
```

Otherwise:

```text id="g7q3v1"
Any REVIEW rule
      ↓
REVIEW
```

Otherwise:

```text id="b5n9x2"
ALLOW
```

This is a **proposed design choice** and should be validated when the detailed rule engine is designed.

---

## 11.9 Risk Decision States

The Risk Engine produces one of three control outcomes.

### ALLOW

The transaction has not triggered a blocking or review rule.

```text id="f8m3q6"
Risk
 ↓
ALLOW
 ↓
Continue payment cycle
```

### REVIEW

The transaction requires additional control.

```text id="w4n7p2"
Risk
 ↓
REVIEW
 ↓
Human Approval / Additional Review
```

### BLOCK

The transaction is considered unacceptable under the configured risk policy.

```text id="k9r5x3"
Risk
 ↓
BLOCK
 ↓
Payment cannot execute
```

Risk `REVIEW` and policy `REQUIRE_APPROVAL` may eventually converge on the same approval workflow, but they represent different reasons f_

## 12. Human Approval

Human Approval provides a controlled mechanism for requiring an explicit human decision before an agent-initiated payment can proceed to execution.

Human approval is triggered when the applicable delegation policy or risk controls determine that automated execution is insufficient.

The core principle is:

> **An Agent may initiate a payment, but a transaction requiring human approval cannot be executed until the required human decision has been successfully recorded and validated.**

Human approval is part of FinFlow's deterministic control layer. It does not execute the payment, modify the ledger, or directly communicate with the payment rail.

---

### 12.1 Human Approval Model

The approval flow is:

```text id="k7p4m2"
Payment Intent
      │
      ▼
Authentication
      │
      ▼
Authorization
      │
      ▼
Risk Evaluation
      │
      ▼
Approval Required
      │
      ▼
Approval Request
      │
      ▼
Human Decision
      │
      ├──────────────┐
      │              │
      ▼              ▼
   APPROVE        REJECT
      │              │
      ▼              ▼
Payment          Payment
Execution        Rejected
```

Approval is conditional.

Transactions that do not require human intervention may proceed without creating an `ApprovalRequest`.

---

## 12.2 Why Human Approval Exists

Agent authorization and automated risk controls are not always sufficient for every transaction.

A delegation policy may intentionally define an approval boundary.

Example:

```text id="x4m8q1"
Policy:

Agent may spend up to ₹10,000/day.

Transactions:
    ≤ ₹3,000 → automatic
    > ₹3,000 → human approval
```

The Agent can therefore operate autonomously for low-value transactions while requiring human confirmation for higher-value operations.

This creates a bounded autonomy model:

```text id="m9p2k5"
Low-risk / low-value
        │
        ▼
Automated execution

High-value / higher-risk
        │
        ▼
Human approval
```

---

## 12.3 Approval Request

An `ApprovalRequest` represents a specific requirement for a human decision.

Conceptual attributes:

```text id="v6x3n8"
ApprovalRequest
---------------
id
payment_id
requested_from
status
reason
created_at
expires_at
resolved_at
resolved_by
```

The exact fields will be finalized during data-model design.

An ApprovalRequest should identify:

* the Payment requiring approval
* the human or approval group responsible
* why approval is required
* the current approval state
* when approval was requested
* when approval expires
* who resolved it
* when it was resolved

---

## 12.4 Approval States

The initial approval lifecycle is:

```text id="q8m4y1"
PENDING
   │
   ├──────────────► APPROVED
   │
   ├──────────────► REJECTED
   │
   └──────────────► EXPIRED
```

### PENDING

An approval request has been created but no valid decision has been recorded.

```text id="n5p7x2"
Payment
   │
   ▼
ApprovalRequest
   │
   ▼
PENDING
```

The payment must not execute while a mandatory approval remains pending.

### APPROVED

The required human has explicitly approved the payment.

```text id="r3k9m6"
PENDING
   │
   ▼
APPROVED
   │
   ▼
Payment may proceed
```

### REJECTED

The human explicitly rejected the payment.

```text id="w2q6v8"
PENDING
   │
   ▼
REJECTED
   │
   ▼
Payment cannot execute
```

### EXPIRED

The approval request passed its validity period without a valid approval.

```text id="j7m4p9"
PENDING
   │
   ▼
EXPIRED
   │
   ▼
Payment cannot execute
```

The exact expiration and re-request semantics are open design decisions.

---

## 12.5 When Approval Is Required

Human approval can be triggered by multiple control conditions.

### Policy threshold

Example:

```text id="y6x3n8"
Transaction amount > ₹5,000
        ↓
Approval required
```

### Risk review

Example:

```text id="p4m8q2"
Risk Decision:
    REVIEW

        ↓

Approval required
```

### Combined control

A policy may require approval for a particular merchant category:

```text id="c9v5k1"
Merchant Category:
    HIGH_VALUE_ELECTRONICS

        ↓

Approval required
```

The system should represent the reason for approval explicitly rather than simply storing:

```text id="h8x2m5"
approved = false
```

---

## 12.6 Approval Reason

Every ApprovalRequest should contain a machine-readable reason.

Examples:

```text id="n3k7p4"
AMOUNT_THRESHOLD
RISK_REVIEW
MERCHANT_RESTRICTION
HIGH_VALUE_TRANSACTION
POLICY_REQUIREMENT
```

An approval request may contain multiple reasons:

```text id="q6m2x8"
Reasons:
    AMOUNT_THRESHOLD
    HIGH_VELOCITY
```

This allows the user and audit system to understand why the payment was held for approval.

---

## 12.7 Approval Authority

Not every human should necessarily be allowed to approve every payment.

The approval authority must be determined by the applicable policy and system permissions.

Conceptually:

```text id="x7p4m2"
Approval Request
      │
      ▼
Eligible Approver
      │
      ▼
Human Decision
```

An approver may be:

```text id="k9m3v6"
Payment Owner
Authorized User
Administrator
Designated Approval Group
```

The exact approval hierarchy is a future design decision.

The initial implementation should avoid unnecessarily complex organizational approval workflows.

---

## 12.8 Approval Authentication

A valid approval must be associated with an authenticated human identity.

The system must establish:

```text id="r5x8n2"
Who approved the transaction?
```

An approval request cannot be considered valid merely because a client submits:

```text id="m7q3k9"
approved = true
```

The approval operation must be authenticated and authorized.

Conceptually:

```text id="c4p8y1"
Human
  │
  ▼
Authentication
  │
  ▼
Approver Authorization
  │
  ▼
Approval Decision
```

This prevents an Agent or unauthorized client from approving its own transaction.

---

## 12.9 Separation of Duties

A critical security principle is that the Agent initiating a transaction must not be able to satisfy its own human approval requirement.

Conceptually:

```text id="v8m3q6"
Agent
  │
  │ initiates
  ▼
Payment
  │
  │ requires approval
  ▼
Human
```

Not:

```text id="f2k7p9"
Agent
  │
  ├── initiates payment
  │
  └── approves payment
```

This preserves the purpose of human-in-the-loop control.

The exact separation-of-duties rules will be finalized during the security design.

---

## 12.10 Approval and Payment State

Human approval affects the Payment state machine.

Example:

```text id="y4p8m2"
AUTHORIZED
    │
    ▼
RISK_EVALUATED
    │
    ▼
PENDING_APPROVAL
    │
    ├──────────────► REJECTED
    │
    ├──────────────► EXPIRED
    │
    ▼
APPROVED
    │
    ▼
PROCESSING
```

The Payment cannot transition to `PROCESSING` while a mandatory approval remains unresolved.

Therefore:

```text id="n6x3q9"
PENDING_APPROVAL
        │
        ✗
        │
        ▼
PROCESSING
```

is an invalid transition.

Only:

```text id="j8m4v2"
PENDING_APPROVAL
        │
        ▼
APPROVED
        │
        ▼
PROCESSING
```

is permitted.

---

## 12.11 Approval Decision

A human approval operation should produce a structured decision.

Conceptually:

```text id="p7k3x9"
ApprovalDecision
---------------
approval_request_id
decision
approver_id
reason
timestamp
```

Possible decisions:

```text id="m4q8y2"
APPROVE
REJECT
```

The decision must be associated with the specific ApprovalRequest.

This prevents a generic approval response from being reused across unrelated payments.

---

## 12.12 Approval Expiration

Approval requests may expire.

Example:

```text id="x9m5p3"
Approval requested:
10:00

Expiration:
11:00

No decision by 11:00
        ↓
EXPIRED
```

An expired approval cannot automatically authorize payment execution.

This prevents stale approvals from being used after the underlying transaction context may have changed.

The system must define whether an expired approval requires:

```text id="q6k2v8"
New ApprovalRequest
```

or whether the payment itself is permanently rejected.

For the initial design, creating a new approval request is preferable because it preserves a clear audit trail.

---

## 12.13 Approval and Policy Changes

Consider:

```text id="c8m4y1"
10:00
Policy:
    ₹5,000 threshold

Payment:
    ₹7,000

Approval requested
```

At 10:15, the User changes the policy:

```text id="n3p7x9"
New threshold:
₹10,000
```

The existing ApprovalRequest must not become ambiguous.

The system should associate the approval request with the relevant policy version and authorization context.

This allows the system to determine:

> Why was approval required when the payment was created?

Historical decisions should remain explainable even after policies change.

---

## 12.14 Approval and Risk Decisions

Risk and human approval interact as follows:

```text id="y7m2q4"
Risk
 │
 ├── ALLOW ────────────► Continue
 │
 ├── REVIEW ───────────► Approval Required
 │
 └── BLOCK ────────────► Stop
```

A `REVIEW` result does not mean the transaction is automatically rejected.

It means:

> Additional control is required before execution.

A `BLOCK` result means the transaction cannot proceed under the applicable risk policy.

Therefore:

```text id="p5x8n3"
REVIEW
  ≠
BLOCK
```

and:

```text id="k4m9q2"
APPROVAL
  ≠
AUTHORIZATION
```

Approval is an additional control step, not a replacement for authorization.

---

## 12.15 Approval and Authorization

Human approval cannot grant authority that was never delegated.

Example:

```text id="v8p3m6"
Delegation Policy:
Maximum = ₹5,000

Payment:
₹10,000

Authorization:
DENY
```

A human clicking "Approve" must not transform this into an authorized payment unless the system's explicit policy model defines a separate, higher-level authority for that approver.

The default FinFlow model is:

```text id="q2m7x9"
Authorization = required
Approval = additional control
```

Therefore:

```text id="n6p4k8"
Authorization DENY
        +
Human APPROVE
        ↓
Still DENY
```

This prevents human approval from becoming an accidental mechanism for bypassing delegated authority.

---

## 12.16 Approval and Idempotency

Approval operations may also be retried.

For example:

```text id="x7m3q9"
Approve Request A
       ↓
Network timeout
       ↓
Client retries
       ↓
Approve Request A
```

The system must not create multiple conflicting approval decisions.

Approval operations should therefore be idempotent with respect to the ApprovalRequest.

For example:

```text id="p5k8m2"
PENDING
   │
   ▼
APPROVED
```

Repeating the same approval should result in the same final state.

A conflicting decision should be rejected:

```text id="c9m4x7"
APPROVED
   │
   ✗
REJECT
```

unless an explicitly defined administrative override mechanism exists.

---

## 12.17 Concurrent Approval Decisions

Multiple approval requests or clients may attempt to resolve the same approval concurrently.

Example:

```text id="j6p2n8"
Approver A → APPROVE
Approver B → REJECT
```

The system must define deterministic behavior.

The initial design should enforce that an ApprovalRequest can transition out of `PENDING` only once.

Conceptually:

```text id="w4m9q3"
PENDING
   │
   ├── APPROVE → APPROVED
   │
   └── REJECT  → REJECTED
```

After the first valid terminal transition:

```text id="v8x2k6"
APPROVED
```

a subsequent conflicting decision must be rejected.

This transition must be concurrency-safe.

---

## 12.18 Approval Notifications

Creating an ApprovalRequest may generate a notification.

Conceptually:

```text id="m3q7p9"
Payment
   │
   ▼
ApprovalRequest
   │
   ▼
Notification Event
   │
   ▼
User / Approver
```

Notification delivery is not itself the source of truth for approval state.

For example:

```text id="x6k2m8"
Notification failed
```

must not cause:

```text id="p9v4q3"
ApprovalRequest = APPROVED
```

The authoritative approval state remains in PostgreSQL.

Notification delivery may be eventually consistent.

---

## 12.19 Approval Audit Trail

Every approval action must be auditable.

An audit record should capture:

```text id="q5m8x2"
Payment
ApprovalRequest
Approver
Decision
Reason
Timestamp
Relevant policy/risk context
```

Example:

```text id="v7p3k9"
Payment P123

Approval:
    APPROVED

Approver:
    User U456

Reason:
    Confirmed purchase

Timestamp:
    2026-09-13T10:30:00
```

This allows the system to answer:

> Who approved this transaction, when, and under what approval requirement?

---

## 12.20 Approval Failure Behavior

If the approval service or approval state cannot be safely established, the payment must not proceed.

Example:

```text id="k3m7x9"
Payment requires approval
        │
        ▼
Approval state unavailable
        │
        ▼
Do not execute
```

The system must not interpret:

```text id="n8p2q4"
approval service unavailable
```

as:

```text id="y5m9x3"
approved
```

This follows FinFlow's fail-closed security principle.

---

## 12.21 Approval and Unknown Payment Outcomes

Human approval occurs **before** payment execution.

Therefore an external payment timeout should not be resolved by simply asking for another approval.

Example:

```text id="q7m3x8"
Approval
   ↓
APPROVED
   ↓
Payment Attempt
   ↓
TIMEOUT
   ↓
UNKNOWN
```

The problem is now an execution/reconciliation problem, not an approval problem.

The existing approval should remain historically associated with the payment while the payment outcome is reconciled.

This preserves separation between:

```text id="p4x8m2"
Control decision
```

and:

```text id="j9q3v6"
Execution outcome
```

---

## 12.22 Approval Example

Consider the following policy:

```text id="m8p4x2"
Agent:
    ShoppingAgent

Policy:
    Daily limit = ₹20,000
    Transaction limit = ₹10,000
    Approval threshold = ₹5,000
```

Agent requests:

```text id="v3k7q9"
₹7,000
Merchant = Merchant X
```

Authorization:

```text id="n5m2x8"
Agent active?            YES
Policy active?           YES
Transaction limit?       YES
Budget available?        YES

Authorization:
    ALLOW
```

Risk:

```text id="q8p3m6"
Risk:
    LOW_RISK
```

Policy:

```text id="x4m7k2"
₹7,000 > ₹5,000

Approval required:
    YES
```

Approval request:

```text id="j6p9v3"
ApprovalRequest:
    status = PENDING
    reason = AMOUNT_THRESHOLD
```

Human approves:

```text id="w2m8q5"
APPROVE
```

Payment proceeds:

```text id="k7x3p9"
APPROVED
    ↓
PROCESSING
    ↓
Payment Rail
    ↓
SUCCESS
    ↓
SETTLEMENT
    ↓
LEDGER
```

---

## 12.23 Rejected Approval Example

```text id="p4m8x2"
Payment:
    ₹8,000

Authorization:
    ALLOW

Risk:
    REVIEW

Approval:
    REQUIRED
```

Human rejects:

```text id="v7q3n9"
PENDING
   ↓
REJECTED
```

Payment cannot proceed:

```text id="m2x8k5"
REJECTED
   ↓
No payment execution
   ↓
No successful settlement
```

The rejection and its reason are recorded in the audit trail.

---

## 12.24 Approval State Machine

The conceptual ApprovalRequest state machine is:

```text id="r6m3x9"
                    ┌──────────────┐
                    │              │
                    ▼              │
                 APPROVED          │
                    ▲              │
                    │              │
PENDING ────────────┤              │
    │               │              │
    │               │              │
    ├──────────────►REJECTED       │
    │                              │
    └──────────────►EXPIRED        │
                                   │
                                   │
                        No further transitions
```

More explicitly:

```text id="q8p4m2"
PENDING
  │
  ├── approve ──► APPROVED
  │
  ├── reject ───► REJECTED
  │
  └── timeout ──► EXPIRED
```

Terminal states:

```text id="m7x3k9"
APPROVED
REJECTED
EXPIRED
```

Terminal approval states cannot normally transition to another state.

---

## 12.25 Approval Guarantees

The Human Approval subsystem must preserve the following guarantees.

### H1. Required approval blocks execution

```text id="x4m8p2"
A payment requiring approval cannot enter
execution while approval remains pending.
```

### H2. Only authorized humans can approve

```text id="q7n3m6"
An approval must be associated with an
authenticated and authorized approver.
```

### H3. Agent cannot self-approve

```text id="p5x9k2"
An Agent must not be able to satisfy
its own human approval requirement.
```

### H4. Approval cannot grant unauthorized authority

```text id="m8q4x7"
Human approval does not override a failed
authorization decision by default.
```

### H5. Approval is single-resolution

```text id="v3p7n9"
An ApprovalRequest may transition from
PENDING to one terminal state only.
```

### H6. Approval decisions are auditable

```text id="k6x2m8"
The system must record who approved or rejected
the request and when.
```

### H7. Expired approvals cannot authorize execution

```text id="n4m9q3"
An expired ApprovalRequest is not a valid
authorization for payment execution.
```

### H8. Approval failure is fail-closed

```text id="q8p3x5"
If required approval state cannot be safely
established, payment execution must not proceed.
```

### H9. Approval is repeat-safe

```text id="j5m7x2"
Repeated delivery of the same approval command
must not create conflicting financial effects.
```

---

## 12.26 Architectural Boundary

The Human Approval component is responsible for:

```text id="p9x3m6"
✓ Creating approval requests
✓ Identifying approval requirements
✓ Identifying eligible approvers
✓ Recording approval decisions
✓ Managing approval lifecycle
✓ Enforcing approval expiration
✓ Preventing conflicting resolution
✓ Producing approval audit information
```

It is not responsible for:

```text id="v6m2q8"
✗ Granting Agent authority
✗ Performing risk evaluation
✗ Reserving budgets
✗ Executing payments
✗ Calling payment rails
✗ Performing settlement
✗ Writing financial ledger entries
```

The responsibility boundary is:

```text id="k4p8m3"
Authorization
      │
      ▼
Risk
      │
      ▼
Human Approval
      │
      │ approved
      ▼
Payment Engine
      │
      ▼
Settlement
```

---

## 12.27 Approval and Control-Layer Philosophy

The control layer now has three distinct decision mechanisms:

```text id="x7m4p9"
┌─────────────────────────────────────────────┐
│              CONTROL LAYER                  │
│                                             │
│  Authorization                              │
│       │                                     │
│       │ "Is the Agent allowed?"             │
│       ▼                                     │
│  Risk                                       │
│       │                                     │
│       │ "Is the transaction acceptable?"    │
│       ▼                                     │
│  Human Approval                             │
│       │                                     │
│       │ "Does a human need to confirm?"     │
│       ▼                                     │
└───────┼─────────────────────────────────────┘
        │
        ▼
  Payment Execution
```

These controls are complementary rather than interchangeable.

```text id="m8q3x6"
Authorization
    ≠ Risk
    ≠ Approval
```

Each answers a different question.

---

## 12.28 Design Principles

1. **Human approval is a control mechanism, not a payment mechanism.**
2. **Approval is required only when explicitly triggered by policy or risk controls.**
3. **Approval requests are stateful domain objects.**
4. **Approval decisions must be authenticated and authorized.**
5. **Agents cannot satisfy their own human approval requirements.**
6. **Approval cannot grant authority that authorization denied.**
7. **Approval decisions must be auditable.**
8. **Approval requests should be associated with relevant policy and risk context.**
9. **Approval state transitions must be concurrency-safe.**
10. **Expired approvals cannot authorize payment execution.**
11. **Approval failures must fail closed.**
12. **Repeated approval requests must be safe and idempotent.**
13. **Approval state is authoritative in PostgreSQL.**
14. **Notification delivery must not determine approval state.**
15. **Approval remains conceptually separate from payment execution and settlement.**

---

## 12.29 Open Design Questions

The following decisions remain open for detailed design:

1. Exact approval eligibility model.
2. Whether a payment may require multiple approvers.
3. Whether approval groups are supported.
4. Whether approvals are sequential or parallel.
5. Whether different transaction amounts require different approver levels.
6. Exact approval expiration duration.
7. Re-approval behavior after expiration.
8. Whether policy changes invalidate existing pending approvals.
9. Whether an approver may reject and later re-approve.
10. Separation-of-duties rules.
11. Administrative override mechanisms.
12. Notification mechanism.
13. Approval API design.
14. Approval idempotency-key semantics.
15. Whether risk `REVIEW` always creates an ApprovalRequest.
16. Whether approval decisions should be persisted as immutable decision records in addition to current approval state.

These decisions should be resolved during detailed authorization, workflow, and security design and documented through ADRs where appropriate.

---

## 12.30 Summary

Human Approval provides the final human control boundary before payment execution when automated controls require additional confirmation.

```text id="n5x8m2"
Agent
  │
  ▼
Payment Intent
  │
  ▼
Authorization
  │
  ▼
Risk Evaluation
  │
  ├── BLOCK ───────────────► STOP
  │
  ├── ALLOW ───────────────► Continue
  │
  └── REVIEW ──────────────► Approval
                                  │
                         ┌────────┴────────┐
                         ▼                 ▼
                     APPROVED           REJECTED
                         │
                         ▼
                  Payment Execution
                         │
                         ▼
                     Settlement
                         │
                         ▼
                       Ledger
```

The resulting trust model is:

```text id="q7m3x9"
Agent
  ↓
"I want to do this."

Authorization
  ↓
"Are you allowed to do this?"

Risk Engine
  ↓
"Does this transaction satisfy automated risk controls?"

Human Approval
  ↓
"Does this transaction require explicit human confirmation?"

Payment Engine
  ↓
"Execute the authorized operation."

Ledger
  ↓
"Record the financial effect."
```

Human approval therefore strengthens FinFlow's bounded-autonomy model without allowing the Agent to bypass deterministic authorization, risk controls, or financial safeguards.


## 13. Financial Ledger Model

The Financial Ledger Model defines how FinFlow represents, records, and preserves financial transactions.

The ledger is the authoritative historical record of financial effects within the system. It uses a double-entry accounting model in which every financial transaction produces balanced accounting entries.

The core invariant is:

> **For every completed ledger transaction, total debits must equal total credits.**

The ledger is deliberately separated from payment execution. A Payment represents a financial operation, while the ledger records the resulting accounting effect once the applicable settlement conditions have been satisfied.

---

### 13.1 Ledger Model Overview

The financial model is:

```text id="l3m8q2"
Payment
   │
   │ successful financial outcome
   ▼
Ledger Transaction
   │
   ├──────────► Debit Ledger Entry
   │
   └──────────► Credit Ledger Entry
```

For a payment of ₹7,000:

```text id="x7p4n9"
Ledger Transaction T123

Debit:
    User Ledger Account       ₹7,000

Credit:
    Merchant Ledger Account   ₹7,000

                    ─────────
Total                     ₹7,000
```

Therefore:

```text id="q5m8x3"
Total Debits = Total Credits
             = ₹7,000
```

This invariant must hold for every balanced financial transaction.

---

## 13.2 Why Double-Entry Accounting

A simple balance update such as:

```text id="k8p2v6"
UPDATE account
SET balance = balance - 7000
```

does not provide sufficient financial history.

It tells us the current balance changed, but not:

* why it changed
* where the money went
* which transaction caused the change
* whether the corresponding recipient received the funds
* how the transaction should be reversed
* how the historical state can be reconstructed

Double-entry accounting records both sides of the financial effect.

Example:

```text id="m4x9q7"
User Account
    -₹7,000

Merchant Account
    +₹7,000
```

The ledger therefore represents the movement of value rather than merely storing mutable balances.

---

## 13.3 Ledger Transaction

A **Ledger Transaction** groups all ledger entries belonging to one accounting event.

Conceptually:

```text id="v6p3m8"
LedgerTransaction
-----------------
id
reference_type
reference_id
currency
status
created_at
```

For example:

```text id="j9x4q2"
Ledger Transaction LT123

Reference:
    Payment P123

Currency:
    INR

Entries:
    Debit  ₹7,000
    Credit ₹7,000
```

The Ledger Transaction provides the accounting boundary within which the double-entry invariant must hold.

---

## 13.4 Ledger Account

A **LedgerAccount** represents an accounting account that participates in ledger transactions.

Conceptual attributes:

```text id="p7m2x5"
LedgerAccount
-------------
id
account_reference
account_type
currency
status
created_at
```

Examples:

```text id="q4n8v3"
USER_CASH
MERCHANT_RECEIVABLE
FINFLOW_SETTLEMENT
FEE_REVENUE
```

The exact account taxonomy will be defined during detailed accounting design.

---

## 13.5 Ledger Entry

A **LedgerEntry** represents one side of a Ledger Transaction.

Conceptual attributes:

```text id="x8m4p7"
LedgerEntry
-----------
id
ledger_transaction_id
ledger_account_id
entry_type
amount
currency
created_at
```

`entry_type` represents the accounting side:

```text id="k3q9m6"
DEBIT
CREDIT
```

A Ledger Transaction must contain sufficient entries to balance the transaction.

For a basic two-party transfer:

```text id="v5p8x2"
Ledger Transaction
        │
        ├── DEBIT  User Account      ₹7,000
        │
        └── CREDIT Merchant Account  ₹7,000
```

---

## 13.6 Ledger Invariant

The fundamental ledger invariant is:

```text id="n7m3q8"
Σ(DEBIT entries)
=
Σ(CREDIT entries)
```

For example:

```text id="j4x8p2"
Debit:
    ₹7,000

Credit:
    ₹7,000
```

Valid:

```text id="q6m9v3"
₹7,000 = ₹7,000
```

Invalid:

```text id="x2p7k5"
Debit:
    ₹7,000

Credit:
    ₹6,500
```

because:

```text id="m8q4n6"
₹7,000 ≠ ₹6,500
```

The system must reject an unbalanced ledger transaction.

This invariant is one of FinFlow's strongest financial correctness guarantees.

---

## 13.7 Atomic Ledger Posting

Ledger entries belonging to the same financial transaction must be posted atomically.

Conceptually:

```text id="v3p8m2"
BEGIN TRANSACTION

Create LedgerTransaction

Create Debit Entry
Create Credit Entry

Validate:
    Debits = Credits

COMMIT
```

If any required operation fails:

```text id="q7x4n9"
ROLLBACK
```

The system must not allow:

```text id="k5m2p8"
Debit created ✓
Credit missing ✗
```

This uses the database transaction guarantees discussed earlier.

---

## 13.8 Ledger as Authoritative Financial History

PostgreSQL is the authoritative storage system for FinFlow's financial state.

The ledger therefore represents authoritative historical financial information.

```text id="x9p4m7"
PostgreSQL
    │
    └── Financial Ledger
             │
             ├── Ledger Transactions
             └── Ledger Entries
```

Redis must not become the authoritative ledger.

Kafka must not become the authoritative ledger.

Caches and events are derived or propagation mechanisms.

```text id="m3q8v5"
PostgreSQL
    ↓
Authoritative financial state

Redis
    ↓
Cache / coordination

Kafka
    ↓
Event propagation
```

---

## 13.9 Ledger Immutability

Historical ledger entries should be treated as immutable.

Once a financial transaction has been posted:

```text id="p6x2m8"
Ledger Entry
    ↓
IMMUTABLE
```

The system should not modify the original entry to correct history.

For example, this is not the preferred correction:

```text id="n4q9v7"
Original:
₹7,000

Edit:
₹7,000 → ₹5,000
```

Instead, the correction is represented by a compensating transaction.

---

## 13.10 Reversal / Compensating Transaction

Suppose the original transaction was:

```text id="x7m3p9"
Original Transaction

Debit:
    User       ₹7,000

Credit:
    Merchant   ₹7,000
```

If the transaction needs to be reversed:

```text id="k2q8m5"
Reversal Transaction

Debit:
    Merchant   ₹7,000

Credit:
    User       ₹7,000
```

The original transaction remains intact.

The financial history becomes:

```text id="v4p9x2"
Original Transaction
        +
Reversal Transaction
```

This preserves auditability and historical integrity.

---

## 13.11 Ledger and Payment Lifecycle

The ledger should represent the financial effect of a payment at the appropriate stage of the payment lifecycle.

Conceptually:

```text id="m8x3q7"
Payment Intent
      │
      ▼
Authorization
      │
      ▼
Risk
      │
      ▼
Approval
      │
      ▼
Payment Processing
      │
      ▼
Payment Outcome
      │
      ├── FAILED
      │      ↓
      │   No successful settlement entry
      │
      ├── UNKNOWN
      │      ↓
      │   Reconciliation required
      │
      └── SUCCESS
             │
             ▼
          Settlement
             │
             ▼
        Ledger Posting
```

The exact point at which ledger entries are posted depends on the settlement model selected for the simulated payment rail.

This must be explicitly defined before implementation.

---

## 13.12 Ledger and Unknown Payment Outcomes

An unknown payment outcome must not automatically create a final successful ledger entry.

Example:

```text id="q5m9x2"
Payment Attempt
      │
      ▼
TIMEOUT
      │
      ▼
UNKNOWN
```

At this point:

```text id="p7x3m8"
Financial outcome = unknown
```

Therefore the system must not blindly post:

```text id="v2q8k5"
Debit User
Credit Merchant
```

as though success were confirmed.

Instead:

```text id="n4m7x2"
UNKNOWN
   │
   ▼
Reconciliation
   │
   ▼
Confirmed Outcome
   │
   ▼
Ledger Posting
```

This protects against duplicate or incorrect financial effects.

---

## 13.13 Ledger and Duplicate Payments

Idempotency and ledger integrity work together.

Suppose a client sends:

```text id="x8p4m7"
Idempotency-Key = ABC123
Amount = ₹7,000
```

FinFlow creates:

```text id="q3m9v2"
Payment P123
```

The client retries the request.

FinFlow must not create:

```text id="k6x2p8"
Payment P124
```

for the same logical request.

Therefore the ledger should ultimately record only the intended financial effect:

```text id="m7q4n9"
P123
   ↓
One financial effect
```

not:

```text id="v8p3x5"
P123 → ₹7,000
P124 → ₹7,000
```

The ledger is the final financial safeguard, but idempotency should prevent the duplicate operation from reaching the ledger in the first place.

---

## 13.14 Ledger and Account Balance

A current balance can be derived from ledger entries.

Conceptually:

```text id="q4m8x2"
Opening Balance
      +
Credits
      -
Debits
      =
Current Balance
```

For example:

```text id="p7n3m9"
Opening balance:       ₹10,000

Payment 1:             -₹2,000
Payment 2:             -₹1,500
Refund:                +₹500

Current balance:        ₹7,000
```

The exact balance representation will be determined during database design.

A cached or materialized balance may be maintained for performance, but it must remain consistent with the authoritative ledger.

---

## 13.15 Balance as Derived State

The ledger should be treated as the historical source of financial truth.

A current balance is a derived representation of that history.

Conceptually:

```text id="x5q9m3"
Ledger Entries
      │
      ▼
Balance Calculation
      │
      ▼
Current Balance
```

If FinFlow maintains a materialized balance:

```text id="k7p2v8"
Ledger
  │
  ├── authoritative history
  │
  └──► Materialized Balance
```

the system must define how balance correctness is maintained.

A balance cache must never silently override contradictory ledger state.

---

## 13.16 Currency

Every monetary LedgerEntry must have an associated currency.

Example:

```text id="m3x8q5"
Amount:
₹7,000

Currency:
INR
```

Amounts must not be represented using binary floating-point arithmetic.

The implementation should use an exact monetary representation, such as:

```text id="p8q4n2"
integer minor units
```

For example:

```text id="v6m3x9"
₹100.50
=
10050 paise
```

This avoids floating-point rounding problems.

The exact money representation and supported precision will be defined during database design.

---

## 13.17 Currency Consistency

For the initial implementation, a Ledger Transaction should use a single currency.

Example:

```text id="q9m4x7"
Ledger Transaction
Currency = INR

Debit:
    ₹7,000 INR

Credit:
    ₹7,000 INR
```

This simplifies the initial accounting model.

Multi-currency transactions would require additional concepts such as:

```text id="x5p8m2"
Exchange Rate
FX Conversion
Valuation
Settlement Currency
Transaction Currency
```

These are outside the initial financial-core scope unless later required.

---

## 13.18 Fees

A payment may eventually involve fees.

For example:

```text id="n7x3q9"
Payment:
    ₹7,000

Fee:
    ₹50
```

A multi-entry ledger transaction can represent this explicitly.

For example:

```text id="m4p8k2"
Debit:
    User Account       ₹7,050

Credit:
    Merchant Account   ₹7,000

Credit:
    FinFlow Fee        ₹50
```

The invariant remains:

```text id="q6x2m9"
₹7,050 = ₹7,000 + ₹50
```

Fees should therefore be represented as explicit ledger entries rather than hidden balance modifications.

Fee handling is a future extension and is not required for the minimum viable financial flow.

---

## 13.19 Ledger Transaction Status

A Ledger Transaction may have a lifecycle associated with its posting process.

A conceptual model is:

```text id="p3m8x7"
PENDING
   │
   ▼
POSTED
```

Failure before posting:

```text id="q5n2m9"
PENDING
   │
   ▼
REJECTED
```

Once a financial transaction is successfully posted, its entries should not be modified.

The exact need for an explicit ledger transaction status will be validated during database design.

---

## 13.20 Ledger References

Every ledger transaction should be traceable back to the domain event that caused the financial effect.

For a payment:

```text id="x8m4p2"
LedgerTransaction
      │
      └── reference
             │
             ▼
          Payment P123
```

This allows the system to answer:

> Which payment produced this financial transaction?

Likewise, the payment should be able to reference the corresponding ledger transaction.

This creates a traceability chain:

```text id="v7q3m9"
Agent
  ↓
PaymentIntent
  ↓
Payment
  ↓
PaymentAttempt
  ↓
Settlement
  ↓
LedgerTransaction
  ↓
LedgerEntry
```

---

## 13.21 Ledger and Audit Trail

The ledger and audit trail serve different purposes.

### Ledger

Answers:

> **What financial effect occurred?**

Example:

```text id="m5x8q2"
User Account   -₹7,000
Merchant       +₹7,000
```

### Audit Trail

Answers:

> **Why and how did this financial effect occur?**

Example:

```text id="p3q7n9"
Agent authenticated
Policy approved
Risk approved
Human approved
Payment executed
Ledger posted
```

Therefore:

```text id="x6m2k8"
Ledger
    = financial truth

Audit
    = operational / decision history
```

Neither should replace the other.

---

## 13.22 Ledger and Events

A successful ledger posting may produce a domain event.

Example:

```text id="q8p4m3"
Ledger Posted
     │
     ▼
PaymentSettled
     │
     ▼
OutboxEvent
     │
     ▼
Kafka
```

The event communicates that the financial state changed.

However:

```text id="n5x9m2"
Kafka Event
    ≠
Ledger
```

Kafka is used for downstream propagation.

PostgreSQL remains authoritative.

---

## 13.23 Ledger Posting Transaction

When a payment reaches the stage where its financial effect can be recorded, the posting operation should be atomic.

Conceptually:

```text id="m7p3x8"
BEGIN

Create LedgerTransaction

Create Debit Entry
Create Credit Entry

Verify:
    Debits = Credits

Update relevant financial state

Create OutboxEvent

COMMIT
```

This ensures that the authoritative financial state and the corresponding event record are committed together.

If any required operation fails:

```text id="x4q8m2"
ROLLBACK
```

No partial ledger transaction should remain.

---

## 13.24 Ledger Constraints

The database should enforce as many financial invariants as practical.

Potential constraints include:

```text id="p9m3x7"
Amount > 0
Currency is valid
LedgerAccount exists
LedgerTransaction exists
Entry type is DEBIT or CREDIT
```

Application-level validation should not be the only protection.

Where possible, financial invariants should also be enforced at the database level.

The exact constraints will be defined during schema design.

---

## 13.25 Ledger Concurrency

Ledger posting may occur concurrently for different payments.

The system must ensure that concurrent transactions cannot corrupt financial state.

For example:

```text id="m6x2p8"
Payment A → Ledger Transaction A
Payment B → Ledger Transaction B
Payment C → Ledger Transaction C
```

Each transaction must independently satisfy:

```text id="q4n9m3"
Debits = Credits
```

Account-level balance updates, if maintained separately, must also be concurrency-safe.

The exact locking and isolation strategy will be defined during database concurrency design.

---

## 13.26 Ledger Corrections

Financial corrections should be represented as new transactions rather than destructive edits.

Example:

```text id="x8p3m7"
Original:
    Debit User       ₹5,000
    Credit Merchant  ₹5,000
```

Correction:

```text id="q5m9x2"
Compensating:
    Debit Merchant   ₹5,000
    Credit User      ₹5,000
```

The original remains unchanged.

This provides:

```text id="m4p8n6"
Historical integrity
+
Auditability
+
Reconstructable financial state
```

---

## 13.27 Refunds

A refund is conceptually a new financial transaction that reverses the economic effect of an earlier payment.

Original:

```text id="v7x3q9"
User       -₹7,000
Merchant   +₹7,000
```

Refund:

```text id="p2m8k4"
Merchant   -₹7,000
User       +₹7,000
```

The original payment remains historically intact.

A refund should reference the original payment or ledger transaction.

```text id="x6q3m9"
Refund
  │
  └── references → Original Payment
```

Partial refunds may require additional rules and are outside the initial core scope.

---

## 13.28 Ledger Security

Financial ledger data is security-sensitive.

Access should therefore be tightly controlled.

The system should prevent unauthorized actors from:

```text id="m8p4x2"
Creating arbitrary ledger entries
Modifying historical entries
Deleting financial history
Changing account ownership
Bypassing ledger validation
```

The Agent must never receive direct ledger-write permissions.

```text id="q7m3x9"
Agent
  │
  ✗
  │ direct ledger access
  │
  └────────── not permitted
```

Ledger writes occur through trusted FinFlow financial workflows.

---

## 13.29 Ledger Failure Behavior

If ledger posting fails after a payment has been confirmed successful, FinFlow must not silently treat the transaction as if nothing happened.

This creates an important recovery case:

```text id="x5m8q2"
Payment Rail
    ↓
SUCCESS
    ↓
Ledger Posting
    ↓
FAILURE
```

The financial state is now incomplete from FinFlow's perspective.

The system must retain enough durable state to retry or reconcile the ledger posting safely.

The exact recovery workflow will be defined in the settlement and reliability design.

A retry must be idempotent and must not create duplicate financial effects.

---

## 13.30 Ledger and Reconciliation

Reconciliation compares FinFlow's internal financial state with the outcome reported by the payment rail.

Conceptually:

```text id="p9x4m7"
Payment Rail Records
        │
        │ compare
        ▼
FinFlow Records
        │
        ▼
Reconciliation Result
```

Possible outcomes:

```text id="m3q8v2"
MATCHED
MISSING_INTERNAL_RECORD
MISSING_EXTERNAL_RECORD
AMOUNT_MISMATCH
STATUS_MISMATCH
```

Reconciliation is particularly important for:

* unknown payment outcomes
* asynchronous settlement
* service failures
* recovery after crashes

The initial implementation may use a simulated reconciliation mechanism.

---

## 13.31 Financial Ledger Guarantees

The ledger must preserve the following guarantees.

### L1. Double-entry balance

```text id="x7m2p9"
For every LedgerTransaction:

Total Debits = Total Credits
```

### L2. Atomic posting

```text id="q4p8n3"
A LedgerTransaction is either completely posted
or not posted.
```

### L3. Historical immutability

```text id="m6x3k8"
Posted LedgerEntries are not modified to rewrite history.
```

### L4. Compensating corrections

```text id="v9p2m5"
Corrections are represented by new compensating
transactions.
```

### L5. Traceability

```text id="k3x8q7"
Every financial transaction can be traced
back to its originating payment.
```

### L6. No unknown-outcome settlement

```text id="p5m9x2"
An UNKNOWN payment outcome cannot directly
produce a final settlement entry.
```

### L7. No duplicate financial effect

```text id="x8q4m6"
Retries must not produce duplicate ledger effects.
```

### L8. Authoritative storage

```text id="n7m3p9"
The authoritative ledger is stored in PostgreSQL.
```

### L9. Exact monetary representation

```text id="q2x8m4"
Financial amounts must use exact arithmetic
rather than binary floating-point representation.
```

### L10. Concurrency safety

```text id="m5p7x3"
Concurrent ledger operations must preserve
financial invariants and account consistency.
```

---

## 13.32 Initial Ledger Scope

### In scope

```text id="v8m3q5"
✓ Double-entry accounting
✓ Ledger transactions
✓ Ledger accounts
✓ Ledger entries
✓ Debit / credit invariant
✓ Immutable posted entries
✓ Compensating transactions
✓ Payment-to-ledger traceability
✓ Exact monetary representation
✓ INR support
✓ Atomic posting
✓ Idempotent ledger posting
✓ Reconciliation support
✓ Audit integration
```

### Initially out of scope

```text id="p4x9m2"
✗ Multi-currency accounting
✗ FX conversion
✗ Complex fee structures
✗ Tax accounting
✗ Real banking settlement
✗ Regulatory accounting
✗ Production financial reporting
✗ General ledger functionality for arbitrary businesses
```

The initial ledger is designed specifically to support FinFlow's simulated payment infrastructure rather than to implement a complete enterprise accounting platform.

---

## 13.33 Architectural Boundary

The Financial Ledger is responsible for:

```text id="k7m3x9"
✓ Recording financial effects
✓ Maintaining double-entry balance
✓ Maintaining ledger history
✓ Supporting reversals and compensating entries
✓ Maintaining payment-to-ledger traceability
✓ Providing authoritative financial records
✓ Supporting reconciliation
```

It is not responsible for:

```text id="x4p8m2"
✗ Authenticating Agents
✗ Evaluating delegation policies
✗ Performing risk evaluation
✗ Requesting human approval
✗ Calling payment rails
✗ Generating payment intent
✗ Deciding whether a payment is authorized
```

The boundary is:

```text id="q9m3v7"
Control Layer
      │
      │ authorized operation
      ▼
Payment Engine
      │
      │ confirmed financial outcome
      ▼
Settlement
      │
      ▼
Financial Ledger
```

The ledger records the financial effect. It does not decide whether that effect should occur.

---

## 13.34 Example: Complete Financial Posting

Consider a successful payment:

```text id="m8x3p5"
Payment:

Amount:
₹7,000

From:
User Account

To:
Merchant Account
```

After authorization, risk evaluation, approval, and successful payment execution:

```text id="v4q9m2"
LedgerTransaction LT123

Entry 1:
    DEBIT
    User Account
    ₹7,000 INR

Entry 2:
    CREDIT
    Merchant Account
    ₹7,000 INR
```

Validation:

```text id="p7m3x8"
Debits  = ₹7,000
Credits = ₹7,000

Balanced = TRUE
```

The system then records the corresponding audit and outbox information.

```text id="x5n8q4"
LedgerTransaction
      │
      ├────────► AuditEvent
      │
      └────────► OutboxEvent
                       │
                       ▼
                     Kafka
```

---

## 13.35 Financial Data Flow

The overall financial data flow is:

```text id="q3m7x9"
Agent
  │
  ▼
PaymentIntent
  │
  ▼
Authorization
  │
  ▼
Risk
  │
  ▼
Approval
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
Confirmed Success
  │
  ▼
Settlement
  │
  ▼
LedgerTransaction
  │
  ├── Debit LedgerEntry
  │
  └── Credit LedgerEntry
```

This ensures that the ledger reflects a financial effect only after the payment lifecycle establishes the appropriate outcome.

---

## 13.36 Design Principles

1. **The ledger is the authoritative historical financial record.**
2. **Every financial transaction follows double-entry accounting.**
3. **Total debits must equal total credits.**
4. **Ledger posting is atomic.**
5. **Posted ledger history is immutable.**
6. **Corrections use compensating transactions.**
7. **Unknown payment outcomes do not directly produce final settlement entries.**
8. **Financial amounts use exact arithmetic.**
9. **Ledger transactions are traceable to originating payments.**
10. **Ledger posting must be safe under retries.**
11. **Ledger state must remain independent from Kafka and Redis.**
12. **Agents never receive direct ledger-write authority.**
13. **Balance representations are derived or materialized views of authoritative financial state.**
14. **Financial invariants should be enforced at both application and database boundaries where practical.**
15. **The ledger records financial effects but does not make authorization decisions.**

---

## 13.37 Open Design Questions

The following decisions remain open for detailed financial-system design:

1. Exact relationship between `Account` and `LedgerAccount`.
2. Exact `LedgerTransaction` schema.
3. Account taxonomy.
4. Balance representation.
5. Whether balances are derived on demand or maintained as materialized state.
6. Exact transaction-to-ledger posting boundary.
7. Settlement semantics for the simulated payment rail.
8. Handling of asynchronous settlement.
9. Fee representation.
10. Refund model.
11. Partial refund support.
12. Multi-currency support.
13. Exact monetary precision and minor-unit representation.
14. Ledger posting idempotency mechanism.
15. Reconciliation workflow.
16. Ledger archival and retention.
17. Database constraints enforcing financial invariants.
18. Concurrency strategy for account/balance updates.
19. Whether ledger entries require explicit transaction sequence numbers.
20. Administrative correction mechanisms and their authorization requirements.

These decisions should be finalized during database schema, settlement, and financial correctness design and documented as ADRs where they have significant architectural consequences.

---

## 13.38 Summary

The Financial Ledger Model establishes the financial correctness boundary of FinFlow.

```text id="n6p3x8"
Payment
   │
   │ confirmed financial effect
   ▼
LedgerTransaction
   │
   ├── Debit
   │
   └── Credit
```

with the fundamental invariant:

```text id="m4q8x2"
Total Debits = Total Credits
```

The ledger provides:

```text id="x7p3n9"
Financial history
+
Double-entry correctness
+
Immutability
+
Traceability
+
Reconciliation support
```

The complete financial trust chain is:

```text id="q5m8x3"
Agent Intent
     ↓
Authorization
     ↓
Risk
     ↓
Human Approval
     ↓
Payment Execution
     ↓
Confirmed Outcome
     ↓
Settlement
     ↓
Ledger
```

This separation ensures that **permission, risk, execution, and financial recording remain distinct responsibilities**.

The Agent proposes the transaction.

The Control Layer determines whether it may proceed.

The Payment Engine executes it.

The Ledger records what financially happened.

## 14. Consistency Model

FinFlow operates as a distributed system where different components may observe and process state at different times. Because financial correctness is more important than availability or latency, the system must explicitly distinguish between **authoritative financial state** and **derived or asynchronous state**.

The core consistency principle is:

> **FinFlow uses strong consistency for security- and financial-critical state, while allowing eventual consistency for non-critical derived state and asynchronous processing.**

The system must never sacrifice financial correctness merely to make a transaction appear available or successful.

---

### 14.1 Consistency Boundaries

FinFlow separates state into two broad categories:

1. **Authoritative state**
2. **Derived/eventually consistent state**

The authoritative state is stored in PostgreSQL and determines the actual financial and security state of the system.

Derived state may temporarily lag behind the authoritative state and must never override it.

```text
                    FinFlow
                       │
              ┌────────▼────────┐
              │ Authoritative   │
              │     Core        │
              │   PostgreSQL    │
              ├─────────────────┤
              │ Authorization   │
              │ Policies        │
              │ Budgets         │
              │ Payment State   │
              │ Approvals       │
              │ Ledger          │
              │ Idempotency     │
              └────────┬────────┘
                       │
                Transactional
                   Outbox
                       │
                       ▼
                    Kafka
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
      Analytics      Audit      Notifications
          │            │            │
          └────── Eventually Consistent ──────┘
```

---

### 14.2 Strong Consistency

Strong consistency is required for operations where stale state could result in an unauthorized transaction or incorrect financial effect.

The following state is treated as strongly consistent:

* Delegation policies
* Agent authorization state
* Agent revocation
* Budget reservations
* Payment state transitions
* Approval decisions
* Idempotency records
* Ledger transactions
* Ledger entries
* Financial account state

These operations must use PostgreSQL transactions and appropriate concurrency-control mechanisms.

For example, two concurrent payments must not both observe the same available budget and independently consume it.

```text
Budget Limit = ₹10,000

Payment A = ₹7,000
Payment B = ₹6,000

Concurrent requests
       │
       ├──────────────┐
       ▼              ▼
   Request A       Request B
       │              │
       └──────┬───────┘
              ▼
      Authoritative DB
              │
       concurrency control
              │
        ┌─────┴─────┐
        ▼           ▼
      ₹7,000      rejected
      reserved    or deferred
```

The system must never allow both operations simply because they independently observed the previous budget state.

---

### 14.3 Eventual Consistency

Not every FinFlow component needs immediate visibility of every state change.

The following components may operate with eventual consistency:

* Analytics
* Reporting
* Dashboards
* Notifications
* Search/read models
* Non-authoritative audit projections
* Metrics and monitoring views

For example, after a payment becomes `SUCCESS`, an analytics consumer may process the corresponding Kafka event slightly later.

```text
Payment Service
      │
      │ SUCCESS committed
      ▼
 PostgreSQL
      │
      ▼
Transactional Outbox
      │
      ▼
    Kafka
      │
      ├──────► Analytics
      │
      ├──────► Notifications
      │
      └──────► Read Models
```

Temporary differences between these derived systems and PostgreSQL are acceptable as long as they eventually converge and cannot affect financial correctness.

---

### 14.4 PostgreSQL as the Authoritative Source of Truth

PostgreSQL is the authoritative source of truth for FinFlow's critical state.

Other infrastructure components must not become independent authorities for financial state.

For example:

```text
PostgreSQL:
Available Budget = ₹3,000

Redis:
Available Budget = ₹10,000
```

Redis may contain stale data because of cache expiration, delayed invalidation, failures, or replication delays.

A security-critical authorization decision must therefore not blindly trust stale cached state.

The same principle applies to:

* Kafka events
* Read models
* Analytics databases
* Application-level caches
* In-memory state

These systems may provide performance or asynchronous processing, but PostgreSQL remains authoritative.

---

### 14.5 Read-Your-Writes Consistency

User-facing operations should provide read-your-writes behavior where practical.

For example:

```text
POST /payments
        │
        ▼
Payment P123 created
        │
        ▼
GET /payments/P123
```

The subsequent read should not incorrectly report that `P123` does not exist merely because the request was routed to a stale replica.

For critical payment workflows, reads should therefore be directed to authoritative state or otherwise guarantee visibility of the committed write.

---

### 14.6 Monotonic State Observation

Payment state should not appear to move backward because different components observe different versions of the state.

For example, after observing:

```text
PROCESSING
```

a client should not subsequently receive:

```text
CREATED
```

simply because another read was served by stale state.

Payment state transitions are therefore treated as authoritative state transitions rather than independently derived observations.

The payment state machine remains the source of truth for determining valid state.

---

### 14.7 Consistency and Concurrency

Consistency cannot be achieved merely by declaring that a database is "strongly consistent."

FinFlow must also control concurrent modifications.

Consider:

```text
Initial budget = ₹10,000

Request A:
    read budget → ₹10,000

Request B:
    read budget → ₹10,000

Request A:
    reserve ₹7,000

Request B:
    reserve ₹6,000
```

A naive read-then-write implementation could incorrectly approve both transactions.

Therefore, critical operations must combine:

* Database transactions
* Appropriate isolation levels
* Row-level locking or atomic conditional updates
* Constraints where applicable
* Idempotency
* Explicit state transitions

The exact mechanism will be determined during database and concurrency design.

---

### 14.8 Consistency and Kafka

Kafka introduces asynchronous processing into FinFlow.

A payment transaction may commit in PostgreSQL before downstream consumers process the corresponding event.

Therefore, downstream services must be designed with the assumption that they may temporarily observe an older state.

Example:

```text
PostgreSQL
Payment = SUCCESS
       │
       ▼
Kafka
       │
       ├──► Ledger Consumer
       ├──► Notification Consumer
       └──► Analytics Consumer
```

Consumers must be:

* Idempotent
* Safe to retry
* Able to process duplicate events
* Able to tolerate temporary delays
* Unable to override authoritative financial state incorrectly

The transactional outbox pattern is used to ensure that a committed database state change has a durable corresponding event for asynchronous processing.

---

### 14.9 Consistency During Network Partitions

FinFlow must assume that network failures can occur between services.

For security- and financial-critical operations, the system must prefer correctness over making an uncertain decision.

If the system cannot reliably establish:

* Agent authorization
* Delegation policy
* Budget availability
* Risk decision
* Approval state
* Payment state

the operation must not silently proceed using stale or incomplete information.

The default behavior for critical authorization or financial checks is therefore:

```text
Unable to establish authoritative state
                 │
                 ▼
          Do not guess
                 │
                 ▼
       Reject / defer / retry
```

This follows the broader system guarantee:

> **When a critical control cannot establish a safe decision, FinFlow must fail closed rather than bypass the control.**

---

### 14.10 External Payment Rail Consistency

The external payment rail introduces a special form of uncertainty.

Consider:

```text
FinFlow
   │
   │ Payment Request ₹5,000
   ▼
Payment Rail
   │
   │ Payment processed
   │
   X──── network failure
```

FinFlow may not receive the response.

The system therefore cannot safely conclude that:

```text
TIMEOUT = FAILED
```

Instead:

```text
UNKNOWN
```

must be represented explicitly.

The final outcome must be determined through reconciliation or another authoritative mechanism.

This prevents dangerous behavior such as blindly retrying an operation that may already have succeeded.

---

### 14.11 Consistency Classification

| Domain / Component      | Consistency Model            | Reason                                        |
| ----------------------- | ---------------------------- | --------------------------------------------- |
| Authorization           | Strong                       | Prevent unauthorized payments                 |
| Delegation Policy       | Strong                       | Authority must be current                     |
| Agent Revocation        | Strong                       | Revoked agents cannot authorize new payments  |
| Budget Reservation      | Strong                       | Prevent concurrent overspending               |
| Payment State           | Strong                       | Preserve valid lifecycle transitions          |
| Approval State          | Strong                       | Prevent conflicting decisions                 |
| Idempotency Records     | Strong                       | Prevent duplicate financial effects           |
| Ledger                  | Strong                       | Preserve financial integrity                  |
| Account Financial State | Strong                       | Preserve authoritative financial state        |
| Kafka Events            | Eventual                     | Asynchronous event propagation                |
| Notifications           | Eventual                     | Delay does not change financial state         |
| Analytics               | Eventual                     | Historical reporting can tolerate delay       |
| Dashboards              | Eventual                     | Observability views need not be authoritative |
| Search / Read Models    | Eventual                     | Derived from authoritative state              |
| Redis Cache             | Eventual / Non-authoritative | Performance optimization                      |
| External Payment Rail   | Explicit Uncertainty         | Network failures may create UNKNOWN outcomes  |

---

### 14.12 Consistency Rules

FinFlow follows these rules:

**C1.** PostgreSQL is the authoritative source of truth for financial and security-critical state.

**C2.** Financially consequential operations must use transactional consistency and appropriate concurrency control.

**C3.** Redis and other caches are non-authoritative.

**C4.** Event-driven consumers must tolerate eventual consistency.

**C5.** Kafka consumers must be idempotent because events may be delivered more than once.

**C6.** Critical authorization decisions must not depend blindly on stale state.

**C7.** Network failures must not be interpreted as successful or failed financial operations without sufficient evidence.

**C8.** External payment uncertainty must be represented explicitly as `UNKNOWN`.

**C9.** Derived state may lag authoritative state but must never override it.

**C10.** Financial correctness takes priority over availability when the system cannot safely establish authoritative state.

---

### 14.13 Design Principle

The overall consistency strategy can be summarized as:

```text
                    Financial Core
                         │
                   Strong Consistency
                         │
             ┌───────────┼───────────┐
             │           │           │
        Authorization  Payment     Ledger
             │          State        │
             │           │           │
             └───────────┼───────────┘
                         │
                  Transactional
                     Outbox
                         │
                         ▼
                       Kafka
                         │
              Eventual Consistency
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
      Analytics     Notifications    Read Models
```

The guiding rule is:

> **Strong consistency protects financial truth. Eventual consistency provides scalability and asynchronous processing around that truth.**

This allows FinFlow to remain both financially safe and architecturally scalable without forcing every component into expensive strong-consistency coordination.

---

### 14.14 Open Design Questions

The following decisions remain intentionally open and will be resolved during detailed system design:

1. Which PostgreSQL isolation level should each critical operation use?
2. Where should row-level locking be used?
3. Which operations can use atomic conditional updates instead?
4. Should payment state transitions use optimistic or pessimistic concurrency control?
5. How should read replicas be used, if at all?
6. Which authorization data may safely be cached in Redis?
7. How should cache invalidation interact with policy revocation?
8. How should stale reads be detected?
9. How should Kafka consumer lag affect user-visible state?
10. Which events require ordering guarantees?
11. How should reconciliation resolve `UNKNOWN` payment outcomes?
12. Which derived views require read-your-writes guarantees?

These questions will be addressed during the database, service-boundary, messaging, and reliability design phases.

## 15. Failure Scenarios

## 16. Security and Threat Model

## 17. High-Level Architecture

## 18. Service Boundaries

## 19. API Boundaries

## 20. Event Boundaries

## 21. Data Ownership

## 22. Observability

## 23. Architecture Decision Records