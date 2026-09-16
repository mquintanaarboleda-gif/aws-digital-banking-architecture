# Open Risks & Production Preconditions

The source exercise intentionally leaves several production parameters unspecified. The architecture therefore treats these items as **open decisions**, not as facts to be invented.

| Risk / open question | Validation required before production |
|---|---|
| Ecuador -> AWS -> Core latency | Run a PoC with bank carriers; compare candidate regions and measure p50/p95/p99 with representative traffic |
| Cloud regulatory approval | Validate current Superintendencia de Bancos requirements, provider evidence, certifications, contracts, auditability, data treatment and exit strategy |
| Actual IdP capabilities | Confirm OIDC, PKCE, MFA, step-up, passkeys, refresh-token rotation and provisioning APIs |
| Interbank payment mechanism | Review the existing payment hub, applicable central-bank mechanism, formats, operational windows, contingency and reconciliation rules |
| BIA / RTO / RPO | Approve recovery objectives per business capability before finalizing replication and secondary-region capacity |
| Personal-data operating model | Define controller/processor roles, contracts, purposes, retention and cross-border-processing rules with Legal/DPO |
| Biometric onboarding | Perform impact/risk assessment; validate provider, thresholds, human-review path, alternatives and deletion policy |
| DevSecOps / SRE maturity | If maturity is insufficient, reduce initial deployable units while preserving logical boundaries for future evolution |
| Load and concurrency | Run performance/load tests before sizing; the source exercise does not provide TPS or concurrent-user targets |
| AWS Region | Select from measured latency, service availability, cost, continuity and compliance evidence rather than assuming a region |
| Service quotas | Review and monitor AWS quotas before load testing and scale-up |
| Cost commitment | Establish a stable utilization baseline before Savings Plan or other capacity commitments |

## Why these stay open

A professional architecture should distinguish **design assumptions** from **approved production requirements**. In this case, the missing BIA, transaction volume, regulatory approval, corporate-system capabilities and measured connectivity data materially change the final architecture and operating cost. Closing them without evidence would create false precision.
