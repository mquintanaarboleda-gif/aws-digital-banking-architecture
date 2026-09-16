# Digital Banking Architecture — Full Case Study

**Solution Architect:** Milton Quintana  
**Context:** Digital banking architecture exercise for Ecuador  
**Cloud:** AWS  
**Methods:** ADD 3.0, C4 Model, ADRs, AWS Well-Architected

> Portfolio version of the original architecture report. The design is a case study, not an official or production architecture of any financial institution.

## 1. Executive summary

The architecture builds a digital-banking layer on AWS while keeping the Core Banking System, customer-detail platform and corporate payment environment as systems of record. AWS hosts customer channels, edge security, APIs, application services, messaging, cache, audit and observability. This reduces legacy coupling and avoids exposing the Core directly to Internet-facing channels.

The design covers customer queries, movements and transactions, with separate support capabilities for onboarding/KYC, notifications and audit. Service boundaries are introduced only where responsibilities, controls or change cycles justify them; unnecessary microservice fragmentation is deliberately avoided.

Key mechanisms include ECS Fargate Multi-AZ, API Gateway, Web BFF, OAuth2/OIDC + PKCE, Anti-Corruption Layer, EventBridge + SQS + DLQ, Aurora PostgreSQL, Redis Cache-Aside, Transactional Outbox, immutable audit evidence and warm-standby DR.

## 2. Design method

The decision process is:

**Context & requirements → ASRs & drivers → patterns/tactics → decisions → technologies → C4/ADR → evaluation & iteration**

ADD 3.0 is used to review inputs, prioritize architectural drivers, refine elements, select design concepts, instantiate technologies, sketch views/ADRs and analyze trade-offs. C4 communicates the architecture; ADRs preserve decisions; AWS Well-Architected is used as a cloud review reference.

## 3. Scope and requirements

The platform must support:

- Customer/product and movement queries
- Transfers and payments
- Web and mobile channels
- OAuth 2.0 identity integration
- Biometric onboarding
- Business audit
- Two independent notification mechanisms
- HA, DR, security, monitoring and auto-healing
- Integration with Core Banking, customer-detail, IdP, KYC and the corporate interbank-payment mechanism

Important assumptions remain open: TPS/concurrency, formal SLA/BIA, RTO/RPO, final AWS Region, detailed IdP capabilities, regulatory approval and organizational DevSecOps/SRE maturity.

## 4. Architectural drivers

| Driver | Priority | Why it matters |
|---|---|---|
| Security & privacy | Very high | Financial, identity and biometric data |
| Consistency & idempotency | Very high | Duplicate monetary effects are unacceptable |
| Availability & resilience | Very high | Channel must survive task/AZ/dependency failures |
| Auditability | Very high | Sensitive actions require correlated evidence |
| Interoperability | High | Multiple corporate systems/providers |
| Performance | High | Predictable digital latency without overloading legacy |
| Modifiability | High | Provider/protocol changes must remain localized |
| Cost & operability | High | Complexity must deliver measurable value |

Verifiable scenarios include Multi-AZ failover without manual intervention, bounded Core timeouts, no duplicate effect for repeated Idempotency-Key requests, server-side protection of web OAuth tokens and correlated business-audit evidence.

## 5. Recommended architecture

The solution uses synchronous calls when the customer needs an immediate business response and asynchronous messaging for notifications, derived audit activity and other secondary effects.

### Main services

- Customer Query Service
- Movement Query Service
- Transaction Service
- Onboarding / KYC Service
- Notification Service
- Audit Service

### Platform decisions

- **Runtime:** ECS on Fargate, Multi-AZ
- **Web:** React + TypeScript, S3/CloudFront, Web BFF
- **Mobile:** Flutter baseline; React Native remains an organizational alternative
- **Identity:** OAuth2/OIDC, Authorization Code + PKCE, MFA/step-up
- **API:** Amazon API Gateway
- **Legacy isolation:** adapters + Anti-Corruption Layer
- **Messaging:** EventBridge + SQS + DLQ
- **Transactions:** Aurora PostgreSQL owned by Transaction Service
- **Cache:** Redis using Cache-Aside
- **Audit:** business Audit Service + immutable S3 evidence archive

## 6. C4 views

### Level 1 — Context

The digital platform interacts with customers, operations/security/audit teams, corporate IdP, Core Banking, customer-detail system, payment hub, KYC provider and notification providers. The external payment rail is reached through the corporate payment mechanism rather than directly by the digital channel.

### Level 2 — Containers

The architecture separates:

- Channels and API access
- Application services
- Data and messaging
- External integration through ACL/adapters

Services own the data they require. Shared tables are not used as an integration mechanism.

### Level 3 — Components

Two containers receive detailed component views because they concentrate the most material risks:

- **Transaction Service:** monetary consistency, idempotency, state, adapters, Outbox and reconciliation
- **Onboarding/KYC Service:** privacy, consent, document validation, liveness/biometric verification, customer matching and identity provisioning

See the [16 editable architecture views](diagram-index.md).

## 7. Authentication and onboarding

### Web

The SPA calls a server-side Web BFF. The BFF initiates Authorization Code + PKCE against the corporate IdP, keeps access/refresh tokens server-side and returns only a protected session cookie (`Secure`, `HttpOnly`, `SameSite`) to the browser.

### Mobile

The native app uses the external system browser and Authorization Code + PKCE with S256. Tokens are stored using secure platform mechanisms. Passkeys/FIDO2 may support re-entry, while sensitive operations can still require step-up authentication.

### Biometric onboarding

Identity proofing is kept separate from OAuth session authentication. The flow includes privacy notice, document validation, liveness, facial comparison, corporate-data verification, IdP provisioning, strong-factor enrollment and controlled retention of evidence.

## 8. Transaction consistency and reconciliation

Transaction Service uses a stable `Idempotency-Key`, durable transaction state and a Transactional Outbox.

The critical rule is: **an unknown monetary result is not blindly retried.** If a Core/payment-hub call times out without a known outcome, the transaction moves to `UNKNOWN` / `PENDING_RECONCILIATION`. A reconciliation worker checks the authoritative system using a stable business reference.

For interbank operations, `ACCEPTED` or `PENDING` means the bank accepted the instruction; it does not imply final Central-Bank settlement. The user interface must display the real state and update it after confirmation/reconciliation.

## 9. Data, cache and audit

| Data | Source of truth | Rule |
|---|---|---|
| Customer/products | Core Banking | May cache permitted read data with TTL |
| Detailed customer | Customer-detail system | Minimize cached attributes |
| Movements/balance | Core / ledger | Monetary authorization uses authoritative balance |
| Transaction/idempotency/outbox | Transaction Service | Private Aurora data; other services use APIs/events |
| Business audit | Audit Service | Query store + immutable evidence archive |
| Biometric onboarding | KYC/provider contract | Minimum retention and reinforced controls |

Redis is a performance optimization, not a source of truth. CloudTrail covers AWS control-plane activity; it does not replace business-level banking audit.

## 10. Resilience patterns

- Timeouts on every remote dependency
- Selective retry with backoff and jitter
- Circuit breakers for degraded dependencies
- Bulkheads to isolate saturation
- Idempotency-Key for transfers/payments
- Transactional Outbox for reliable event delivery
- DLQ for failed asynchronous messages
- Saga only when a real multi-system business transaction has valid compensations
- Eventual consistency for notifications/analytics, never as authority for available monetary balance

## 11. AWS availability, governance and DR

Production should integrate with the organization's existing AWS Landing Zone when available, using AWS Organizations/Control Tower concepts and separate responsibilities for Security, Log Archive, shared/network services, PROD and NON-PROD.

Infrastructure should be delivered as code and through controlled CI/CD. CloudFormation/CDK are AWS-native baseline options; an established corporate Terraform standard may be used instead.

The primary region distributes Fargate workloads across at least two AZs, with Multi-AZ data services and redundant private connectivity to corporate systems. IPSec VPN acts as connectivity backup.

Warm standby is used as the DR design hypothesis. Final secondary capacity, data replication and failover procedures must follow approved BIA/RTO/RPO objectives and must be tested periodically.

## 12. Security and observability

Security is layered across identity, sessions, APIs, network, data, application and governance. The design includes WAF, API throttling, authorization policies, private workloads, TLS, KMS, Secrets Manager, OWASP-aligned testing, GuardDuty, Security Hub, CloudTrail, Config and SIEM integration.

Observability uses structured logs, p50/p95/p99 latency, error rates, cache metrics, queue depth/age, DLQ metrics, OpenTelemetry/ADOT traces and propagated correlation IDs. Alerting should represent customer impact and SLOs rather than CPU alone.

Auto-healing replaces failed processes/capacity; it does not solve a failed external dependency, duplicate events, an unknown monetary outcome or a logical defect.

## 13. Ecuador regulatory considerations

The source exercise identifies privacy, biometric-data, breach-notification, third-party/cloud, operational-risk and payment-system requirements as architecture inputs. This section is not legal advice. A real implementation requires validation by Compliance, Legal, Risk, Security and the Data Protection Officer against the current regulations, provider evidence and contracts.

PCI DSS enters scope only if cardholder data is actually processed/stored/transmitted by the defined solution.

## 14. Architecture decisions

The project records 18 ADRs covering cloud platform, region, runtime, web/mobile frameworks, OAuth flow, API gateway, legacy integration, asynchronous messaging, transactional persistence, cache, reliable event delivery, audit, DR, connectivity, notifications, multi-account governance and IaC/CI-CD.

See [Architecture Decision Matrix](architecture-decisions.md).

## 15. Cost and operating criteria

- Fargate before EKS when Kubernetes is not a business/technical requirement
- EventBridge + SQS before MSK when streaming/replay/partition requirements are absent
- Warm standby before active-active without a BIA justification
- S3 + CloudFront for static SPA delivery
- Redis only when cache metrics demonstrate value
- FinOps from the start: ownership tags, Budgets, Cost Explorer and anomaly detection
- Savings Plans only after a stable consumption baseline
- Service Quotas reviewed before load testing/growth

## 16. Production preconditions

The following remain open until validated with evidence:

- Ecuador → AWS → Core latency and final region
- Cloud regulatory approval
- Real IdP capabilities
- Corporate payment-hub behavior
- BIA / RTO / RPO
- Data-controller/processor and cross-border model
- Biometric impact assessment, retention and deletion
- DevSecOps/SRE maturity
- Load, concurrency and sizing

See [Open Risks & Production Preconditions](open-risks.md).

## 17. Conclusion

The case demonstrates an architecture that is traceable from requirements and quality drivers to patterns, technologies and trade-offs. Equally important, it distinguishes design hypotheses from approved production facts. Region, recovery targets, capacity, corporate-system capabilities and cloud/biometric approvals remain explicit gates rather than invented precision.
