# AWS Digital Banking Architecture

### Solution Architecture Case Study | AWS | C4 | ADD 3.0 | Security | Resilience | Event-Driven Integration

A portfolio case study showing how I design a **secure and resilient digital-banking platform on AWS** from architectural drivers, quality attributes and business constraints—not from a predefined technology stack.

The design keeps the **Core Banking System and other corporate platforms as systems of record** while adding a digital experience and integration layer for web/mobile channels, transactions, onboarding, notifications, audit and observability.

> **Portfolio disclaimer:** this is an independent architecture exercise for professional demonstration. It is not an official, deployed or approved architecture of any financial institution. Scenario identifiers such as “BP” are retained inside the original diagrams. Production use would require formal validation of BIA/RTO/RPO, workload sizing, AWS Region, security, privacy, contracts and current regulatory requirements.

## Explore the case study

| Resource | What it shows |
|---|---|
| **[Original PDF Report](docs/Resolucion_Ejercicio_C4_Banca_BP.pdf)** | Complete 36-page architecture report with requirements, decisions, diagrams and conclusions |
| **[Master Editable Draw.io](diagrams/pages/Banca_BP_Arquitectura_Completa.drawio)** | Complete multi-page editable architecture source |
| **[Full Architecture Case Study](docs/full-architecture-report.md)** | Web-friendly architecture narrative covering requirements, drivers, critical flows, AWS deployment, security, regulation, cost and risks |
| **[Architecture Decision Matrix](docs/architecture-decisions.md)** | 18 ADRs with alternatives, rationale and trade-offs |
| **[ASR Traceability](docs/asr-traceability.md)** | Architectural driver → pattern/tactic → design consequence → technology |
| **[Editable Diagram Index](docs/diagram-index.md)** | 16 individual Draw.io views covering C4, sequences, AWS deployment and observability |
| **[Open Risks & Production Preconditions](docs/open-risks.md)** | Decisions that require evidence before production |
| **[Disclaimer](DISCLAIMER.md)** | Portfolio / non-production scope |

---

## What this project demonstrates

- **Attribute-Driven Design (ADD 3.0)** and Architecturally Significant Requirements (ASRs)
- **C4 architecture** from System Context to Components
- **Architecture Decision Records (ADRs)** with alternatives and trade-offs
- AWS cloud architecture, hybrid connectivity and multi-account governance
- OAuth 2.0 / OIDC, BFF, PKCE, MFA and biometric-onboarding separation
- Transactional consistency using idempotency, durable state and **Transactional Outbox**
- Event-driven integration using **EventBridge + SQS + DLQ**
- Anti-Corruption Layer around legacy/core banking systems
- Multi-AZ availability and **warm-standby multi-region DR**
- Business audit, immutable evidence, observability and security controls
- Explicit management of assumptions, unknowns and production risks

---

## Business context

The target digital platform supports customer/product queries, account movements, transfers and payments, web/mobile channels, OAuth/OIDC authentication, biometric onboarding/KYC, business audit, independent notification mechanisms, HA/DR, monitoring and integration with existing corporate banking systems.

The architecture **does not replace the Core Banking System**. It acts as an experience, orchestration and integration layer around authoritative enterprise systems.

```mermaid
flowchart LR
    C[Customer] --> DB[Digital Banking Platform]
    OPS[Operations / Security / Audit] -. monitors .-> DB
    DB --> IDP[Corporate IdP<br/>OAuth2 / OIDC / MFA]
    DB --> CORE[Core Banking<br/>Customers / Products / Balances]
    DB --> DETAIL[Customer Detail System]
    DB --> PAY[Corporate Payment Hub]
    DB --> KYC[KYC / Biometric Provider]
    DB --> NOTIF[Notification Providers<br/>E-mail / SMS]
    PAY --> BCE[Interbank Payment Rail / BCE]
```

---

## Target solution architecture

```mermaid
flowchart TB
    WEB[React + TypeScript SPA] --> EDGE[CloudFront + WAF]
    MOB[Flutter Mobile App] --> APIGW[Amazon API Gateway]
    EDGE --> APIGW

    APIGW --> BFF[Web BFF]
    APIGW --> CQ[Customer Query Service]
    APIGW --> MQ[Movement Query Service]
    APIGW --> TX[Transaction Service]
    APIGW --> KYC[KYC / Onboarding Service]

    CQ --> REDIS[ElastiCache / Redis]
    CQ --> ACL[Integration Adapters / ACL]
    MQ --> ACL
    TX --> AURORA[Aurora PostgreSQL<br/>State + Idempotency + Outbox]
    TX --> ACL
    KYC --> ACL

    TX --> EB[Amazon EventBridge]
    EB --> SQS1[SQS Notifications]
    EB --> SQS2[SQS Audit]
    SQS1 --> NS[Notification Service]
    SQS2 --> AS[Audit Service]
    AS --> AUDIT[Audit DB + S3 Object Lock]

    ACL --> CORP[Core / Customer Detail / Payment Hub / IdP / KYC]
```

### Main technology decisions

| Area | Decision |
|---|---|
| Cloud | AWS |
| Runtime | Amazon ECS on Fargate, Multi-AZ |
| Web | React + TypeScript + S3/CloudFront + Web BFF |
| Mobile | Flutter baseline; React Native remains an organizational alternative |
| Identity | OAuth 2.0 / OIDC + Authorization Code + PKCE + MFA/step-up |
| API layer | Amazon API Gateway |
| Legacy integration | Adapters + Anti-Corruption Layer |
| Async integration | Amazon EventBridge + SQS + DLQ |
| Transaction persistence | Aurora PostgreSQL owned by Transaction Service |
| Cache | ElastiCache / Redis using Cache-Aside |
| Reliable events | Idempotency + Transactional Outbox |
| Audit | Business Audit Service + audit store + S3 Object Lock |
| Primary availability | Multi-AZ |
| DR | Warm standby hypothesis; final tier driven by BIA/RTO/RPO |
| Hybrid connectivity | Redundant Direct Connect / Hosted Connection + IPSec VPN backup |
| Governance | AWS Organizations / Landing Zone + IaC + CI/CD |

---

## Transaction consistency: the critical design problem

A monetary operation must not be duplicated simply because the network response is uncertain.

```mermaid
sequenceDiagram
    participant C as Client
    participant API as API Gateway
    participant TX as Transaction Service
    participant DB as Aurora PostgreSQL
    participant HUB as Core / Payment Hub
    participant OUT as Outbox Publisher
    participant BUS as EventBridge

    C->>API: POST transaction + Idempotency-Key
    API->>TX: Validated request
    TX->>DB: Check idempotency key
    TX->>DB: Persist PENDING state
    TX->>HUB: Execute monetary instruction
    HUB-->>TX: Confirmed result
    TX->>DB: Commit final state + Outbox record
    TX-->>C: Result
    OUT->>DB: Read unpublished Outbox
    OUT->>BUS: Publish business event
```

If the Core/payment hub times out with an **unknown outcome**, the platform records `UNKNOWN` / `PENDING_RECONCILIATION` and reconciles using a stable business reference. It does **not** blindly repeat the monetary instruction.

---

## AWS availability and disaster recovery

```mermaid
flowchart LR
    USERS[Customers] --> R53[Route 53 / Global Entry]

    subgraph PRIMARY[Primary AWS Region - Multi-AZ]
      P1[ECS Fargate - AZ A]
      P2[ECS Fargate - AZ B]
      PA[(Aurora / Redis Multi-AZ)]
      P1 --- PA
      P2 --- PA
    end

    subgraph SECONDARY[Secondary AWS Region - Warm Standby]
      S1[Reduced ECS Capacity]
      SA[(Replicated / Restorable Data)]
      S1 --- SA
    end

    R53 --> P1
    R53 -. DR failover .-> S1
    P1 --> DX[Redundant Private Connectivity]
    P2 --> DX
    S1 --> DX
    DX --> CORP[Corporate Banking Systems]
```

Warm standby is a **design hypothesis**, not a fabricated requirement. Final capacity, replication and failover targets must come from an approved Business Impact Analysis and RTO/RPO objectives.

---

## Security model

The design applies controls across multiple layers:

- **Identity:** OIDC/OAuth 2.0, PKCE, MFA/step-up, passkeys and least privilege
- **Web session:** BFF pattern; `Secure`, `HttpOnly`, `SameSite` cookies; OAuth tokens kept out of browser localStorage
- **API:** API Gateway, WAF, throttling, contract validation and authorization policies
- **Network:** private workloads, security groups, hybrid connectivity and TLS
- **Data:** KMS, Secrets Manager, minimization and encrypted backups
- **Application:** OWASP ASVS/MASVS, SAST, SCA, DAST, secret/image scanning
- **Detection:** GuardDuty, Security Hub, CloudTrail, AWS Config and corporate SIEM
- **Audit:** business audit is separated from operational logs and AWS control-plane activity

---

## Editable architecture package

The project includes both a **master multi-page Draw.io file** and **16 individual editable views**.

**[Open the master Draw.io source](diagrams/pages/Banca_BP_Arquitectura_Completa.drawio)**

The individual views cover:

1. C4 Level 1 — System Context
2. C4 Level 2A — Channels and Access
3. C4 Level 2B — Application Services
4. C4 Level 2C — Data and Messaging
5. C4 Level 2D — External Integrations / ACL
6. C4 Level 3 — Transaction Service
7. C4 Level 3 — Onboarding / KYC
8. Web Authentication — BFF sequence
9. Mobile Authentication — PKCE sequence
10. Successful Transaction sequence
11. Timeout & Reconciliation sequence
12. Cache-Aside sequence
13. AWS Landing Zone & Governance
14. Primary Region — Multi-AZ & Hybrid Connectivity
15. Disaster Recovery — Warm Standby
16. End-to-End Observability

➡️ **[Open the diagram index and individual editable sources](docs/diagram-index.md)**

---

## Open production decisions

A professional architecture should not invent missing requirements. This design therefore keeps the following explicitly open:

- AWS Region and measured Ecuador → AWS → Core latency
- BIA, RTO and RPO by business capability
- TPS, concurrency and capacity sizing
- Real IdP capabilities and provisioning model
- Corporate payment-hub behavior and reconciliation rules
- Regulatory/contractual cloud approval
- Personal-data and cross-border-processing model
- Biometric impact assessment and retention policy
- DevSecOps/SRE operational maturity

➡️ **[See the production preconditions](docs/open-risks.md)**

---

## Architecture philosophy

> Technology is selected as a consequence of architectural drivers—not as the starting point of the design.

**Context & requirements → ASRs & drivers → patterns/tactics → decisions → technologies → C4 / ADR → evaluation & iteration**

---

## Author

**Milton Quintana**  
IT Infrastructure & Solutions Leader | Solution & Cloud Architecture | AWS / Azure | Technical Pre-Sales

[GitHub Profile](https://github.com/mquintanaarboleda-gif)
