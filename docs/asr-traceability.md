# Architecturally Significant Requirements (ASR) Traceability

The architecture is driven by quality attributes and constraints that materially affect the solution structure. The source report maps the main drivers to patterns/tactics and implementation mechanisms as follows.

| ID | Architectural driver | Pattern / tactic | Design consequence | Technology / mechanism |
|---|---|---|---|---|
| ASR-01 | Availability during failures | Redundancy + health checks | Capacity distributed across Availability Zones with automatic task replacement | ECS Fargate Multi-AZ; Aurora Multi-AZ |
| ASR-02 | Performance for frequent queries | Cache-Aside | Cache only data that tolerates eventual consistency; Core remains authoritative | ElastiCache / Redis |
| ASR-03 | Decoupling and resilience | Event-driven + queue | Separate the monetary transaction from secondary effects such as notification and later consumers | EventBridge + SQS + DLQ |
| ASR-04 | Consistency between state and event | Transactional Outbox | Persist business-state change and publication intent in the same local transaction | Aurora + Outbox Worker |
| ASR-05 | Legacy interoperability | Anti-Corruption Layer + Adapter | Isolate digital contracts from legacy protocols, formats and payment-hub behavior | Integration adapters / APIs |
| ASR-06 | Secure authentication | BFF + Authorization Code + PKCE | Integrate the corporate IdP while keeping browser tokens server-side and using a native-app-safe flow on mobile | OAuth2/OIDC IdP; API Gateway/BFF |
| ASR-07 | Auditability and evidence | Append-only / immutability | Separate business audit from operational logs and preserve protected evidence | Audit Service; S3 Object Lock; CloudTrail |
| ASR-08 | Regional continuity | Controlled DR | Maintain a secondary-region recovery strategy whose final tier is driven by BIA and RTO/RPO | Backups/replication; warm standby hypothesis |

## Prioritized quality drivers

1. **Security and privacy** — financial, identity and biometric information creates regulatory and reputational impact.
2. **Consistency and idempotency** — a duplicated monetary operation is more severe than a slower response.
3. **Availability and resilience** — the channel must tolerate partial failures, task/AZ loss and degraded dependencies.
4. **Auditability** — sensitive actions require traceable user/action/result/reference/correlation evidence.
5. **Interoperability** — the platform depends on Core, customer detail, IdP, KYC, payment and notification systems.
6. **Performance** — digital queries need predictable latency without overloading legacy systems.
7. **Modifiability** — provider and contract changes should remain localized.
8. **Cost and operability** — operational complexity should not be added without a justifiable benefit.

## Design rule

If a component cannot be traced to a relevant requirement, architectural driver or concern, its inclusion should be challenged. This rule is used to avoid unnecessary architecture and uncontrolled service fragmentation.
