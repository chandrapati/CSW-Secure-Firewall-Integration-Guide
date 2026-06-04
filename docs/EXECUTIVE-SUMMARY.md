---
title: "Cisco Secure Workload + Secure Firewall"
subtitle: "Executive Summary — Agentless Microsegmentation"
date: "2026-06-04"
---

# Cisco Secure Workload + Secure Firewall

**Executive summary for security and network leadership**

---

## The problem

East-west traffic between application workloads is often unrestricted once an attacker is inside the network. Perimeter firewalls do not stop lateral movement. Many workloads cannot run host agents — legacy operating systems, appliances, and restricted environments — leaving gaps in segmentation coverage.

## The solution

**Cisco Secure Workload (CSW)** and **Cisco Secure Firewall** together deliver **unified, agentless microsegmentation**: discover how applications communicate, derive least-privilege policy from observed behavior, and enforce on **Firepower Threat Defense (FTD)** devices managed by **Firewall Management Center (FMC)** — without breaking legitimate business traffic.

CSW harmonizes segmentation policy across host firewalls, network firewalls, and cloud controls into one consistent model, reducing attack surface and containing lateral movement.

---

## How it works (two paths, one platform)

| Path | What happens | Business outcome |
|------|--------------|------------------|
| **Visibility** | Firewalls export **NSEL** flow telemetry to CSW | See agentless workloads; map application dependencies before enforcing |
| **Enforcement** | CSW pushes **L3/L4 policy** to FTD via FMC | Segment east-west traffic at the network enforcement point |

```text
  Workloads ──▶ Secure Firewall (FTD) ──NSEL──▶ CSW (discover & model policy)
                                                    │
                                                    └──FMC──▶ FTD (enforce)
```

**SaaS deployments** use Cisco **Secure Connector** to link the cloud CSW tenant with on-premises Ingest appliances and FMC.

---

## Three integrated security outcomes

| Outcome | Description |
|---------|-------------|
| **Microsegmentation** | Agentless east-west segmentation where host agents are not feasible; supports mixed agent + agentless estates |
| **Virtual Patch** | Export workload CVE intelligence to FMC; apply compensating IPS rules until permanent patches are deployed |
| **Rapid Threat Containment** | FMC detects malicious behavior → automated quarantine via CSW guardrail policies across hosts and firewalls |

---

## Deployment patterns (at a glance)

| Pattern | Segmentation scope | Typical use |
|---------|-------------------|-------------|
| **Layer 2 transparent firewall** | Intra- + inter-subnet | Fine-grained control; legacy workloads |
| **Layer 3 routed firewall** | Inter-subnet (zone-level) | Faster time-to-segment; dev / non-prod zones |
| **Cloud hub firewall (AWS / Azure / GCP)** | Inter-VPC / inter-VNet | Centralized cloud east-west control |

CSW supports **dual policy management**: CSW owns east-west microsegmentation rules; the network team retains north-south and baseline FMC policies in **Merge** mode.

---

## Key design principles

- **Discover before enforce** — Application Dependency Mapping (ADM) and monitor mode validate policy before production impact.
- **Topology-aware enforcement** — Rules are pushed only to firewalls on the actual traffic path for each application scope.
- **Dynamic policy** — FMC **Dynamic Objects** update automatically as workloads are added, moved, or removed; no manual IP list maintenance.
- **One scope per access control policy** — Clear ownership boundary between application segmentation and firewall policy sets.

---

## What success looks like

| Metric | Target state |
|--------|--------------|
| Agentless visibility | Flows from firewalls visible in CSW; application dependency map validated by app owners |
| Policy safety | Monitor/simulate mode completed; allowed and denied test cases documented |
| Enforcement | Segmentation active on FTD; lateral paths blocked; business transactions pass |
| Operations | FMC deployment automated from CSW; compliance alerts on policy deviation |

---

## Prerequisites (leadership checklist)

- Cisco Secure Workload tenant (SaaS or on-premises)
- Secure Workload **Ingest appliance** and **Secure Connector** (SaaS)
- **FMC-managed FTD** devices in scope with verified deploy path
- Application scope and ownership defined before enforcement
- Change window and rollback plan for first enforce wave

---

## Official references

| Resource | Link |
|----------|------|
| Secure Workload & Firewall deep dive | [secure.cisco.com whitepaper](https://secure.cisco.com/secure-workload/docs/secure-workload-whitepaper) |
| FMC integration guide | [Cisco.com integration guide](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/integration/fmc/cisco-secure-workload-and-fmc-integration-guide.html) |
| Step-by-step practitioner guide | [GitHub — CSW-Secure-Firewall-Integration-Guide](https://github.com/chandrapati/CSW-Secure-Firewall-Integration-Guide) |

---

> **Disclaimer:** Companion summary material — not official Cisco product documentation. Validate design, supported versions, and licensing with your Cisco account team before production decisions.

**Cisco Secure Workload + Secure Firewall** · Executive Summary · June 2026
