---
title: "CSW + Cisco Secure Firewall Integration Guide"
subtitle: "NetFlow (NSEL) visibility and FMC policy enforcement — step by step"
author: "Cisco Secure Workload User Education"
date: "2026-05-27"
---

# Cisco Secure Workload + Cisco Secure Firewall

**NetFlow (NSEL) ingestion and policy enforcement on the firewall**

> **Disclaimer:** Companion learning material — not official Cisco product documentation. Validate design, supported versions, and limits against your tenant and [Cisco Secure Workload 4.0 connector documentation](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-saas-v40/m-connectors.html) before production.

---

## What this integration does

Cisco Secure Workload (CSW) and Cisco Secure Firewall work together for **agentless east-west segmentation** where host agents are not feasible, or where you want **network enforcement** in addition to host agents.

There are **two separate integrations** — do not confuse them:

| Integration | Connector | Purpose | Data direction |
|-------------|-----------|---------|----------------|
| **Flow visibility (NetFlow / NSEL)** | **Secure Firewall Connector** on **Ingest appliance** | CSW sees flows from FTD/ASA without agents | Firewall → Ingest → CSW |
| **Policy enforcement** | **FMC Connector** | CSW pushes segmentation rules to FTD via FMC | CSW → FMC → FTD |

**Generic NetFlow connector** (switches/routers) is a **third** path — for NetFlow v9/IPFIX from **network devices**, not from Secure Firewall NSEL. Use **Secure Firewall Connector** when the source is **FTD or ASA NSEL**.

This is **not** the same as **Cisco ACI** integration (fabric policy). ACI has its own connector and video — see [ACI and CSW Integration](https://www.youtube.com/watch?v=u7jh3Zw1hlg) in the video library.

---

## Architecture (high level)

```text
                    VISIBILITY PATH (NSEL)
  ┌─────────────────┐     NSEL (NetFlow v9)      ┌──────────────────────┐
  │ Secure Firewall │ ─────────────────────────▶ │ Secure Firewall       │
  │ FTD / ASA       │     port 4729 typical       │ Connector (Docker on  │
  └─────────────────┘                             │ Ingest appliance)     │
                                                    └──────────┬───────────┘
                                                               │
                                                    Secure Connector (SaaS)
                                                               │
                                                               ▼
                                                    ┌──────────────────────┐
                                                    │ Cisco Secure Workload │
                                                    │ (flows, ADM, policy)  │
                                                    └──────────┬───────────┘
                                                               │
                    ENFORCEMENT PATH (FMC)                       │
                                                               ▼
  ┌─────────────────┐     REST API + deploy      ┌──────────────────────┐
  │ Secure Firewall │ ◀───────────────────────── │ FMC Connector         │
  │ FTD (managed)    │     Dynamic objects + ACP   │ (CSW UI)              │
  └─────────────────┘                             └──────────────────────┘
           ▲
           │ manages
  ┌────────┴────────┐
  │ FMC / cdFMC      │
  └─────────────────┘
```

**SaaS CSW:** Secure Connector (or corporate proxy) is required between Ingest appliance and the CSW tenant.

Reference: [Cisco Secure Workload and Secure Firewall White Paper](https://www.cisco.com/c/en/us/products/collateral/security/secure-workload/sec-workload-firewall-wp.html)

---

## Prerequisites checklist

| # | Requirement | Notes |
|---|-------------|--------|
| 1 | **CSW tenant** (SaaS or on-prem) with admin access | Scopes and workspaces ready |
| 2 | **Secure Workload Ingest virtual appliance** | Provisioned and **Active** |
| 3 | **Secure Connector** (SaaS) or network path to CSW | Mandatory for SaaS ingest |
| 4 | **Secure Firewall Management Center (FMC)** or **cdFMC** | For enforcement only |
| 5 | **FMC REST API user** with **administrative privileges** | Required for FMC connector |
| 6 | **FTD or ASA** firewalls in scope | NSEL on firewall; FTD for enforcement |
| 7 | **Access Control Policy (ACP)** mapped to CSW scope** | Per enforcement design |
| 8 | **Capacity planning** | ~45k fps per Secure Firewall connector per Cisco white paper |

---

## Part A — NetFlow / NSEL visibility (Secure Firewall Connector)

**Goal:** CSW receives bidirectional flow records from the firewall **without** host agents — for discovery, ADM, and monitor-mode policy.

### Step A1 — Plan ingest placement

1. Confirm **Ingest appliance** is deployed: **Manage → Workloads → Virtual Appliances**.
2. Note the **connector slot IP** and **listening port** (default **4729** for Secure Firewall connector / NSEL).
3. Ensure firewalls can reach Ingest on **UDP 4729** (adjust if you change listening ports in CSW).
4. Configure **Agent Remote VRF** if using VRFs: **Manage → Agents → Configuration → Agent Remote VRF Configurations** (one connector per VRF).

**Video:** [Connector Overview](https://youtu.be/H6QxuouzeC8) · [Connector Deployment](https://youtu.be/H0as2ppS84Q)

### Step A2 — Enable Secure Firewall Connector in CSW

1. **Manage → Workloads → Connectors**.
2. Select **Cisco Secure Firewall Connector** (formerly ASA connector).
3. **Enable** on the **Ingest appliance** (not Edge-only for this use case).
4. Wait for Docker container status **Enabled**; open connector details and record:
   - Connector **ID**
   - **IP:port** to configure on the firewall (e.g. `172.29.142.27:4729`)

**Limits (CSW 4.0 docs):** max **1** Secure Firewall connector per Ingest appliance; **10** per tenant.

### Step A3 — Enable NSEL on Secure Firewall (ASA example)

On **ASA**, NSEL exports **stateful** flow events (create, teardown, deny, update) to the collector.

Example pattern (replace IP/port with your Ingest connector endpoint):

```text
flow-export destination outside <INGEST_CONNECTOR_IP> 4729
flow-export template timeout-rate 1
!
policy-map flow_export_policy
  class class-default
  flow-export event-type flow-create destination <INGEST_CONNECTOR_IP>
  flow-export event-type flow-teardown destination <INGEST_CONNECTOR_IP>
  flow-export event-type flow-denied destination <INGEST_CONNECTOR_IP>
  flow-export event-type flow-update destination <INGEST_CONNECTOR_IP>
  user-statistics accounting
service-policy flow_export_policy global
```

**FTD / managed devices:** Configure NSEL export toward the same Ingest endpoint per your FMC/device template and the [ASA NetFlow Implementation Guide](https://www.cisco.com/c/en/us/td/docs/security/asa/asa-netflow/asa-netflow.html) / FTD equivalent for your version.

**Video (series):** [Secure Workload & Firewall Integration — Part 1](https://youtu.be/vdHjAl48SuI) (architecture) · [Part 2](https://www.youtube.com/watch?v=xpbg3s0vrcI) (deployment) · [Part 3](https://www.youtube.com/watch?v=X65mwN7kJGg) (operations)

**Updated overview (2025–2026):** [Secure Workload & Secure Firewall Integration Updates](https://youtu.be/IEqbz44YvOQ)

### Step A4 — Validate flow ingestion in CSW

1. Generate test traffic through the firewall (permitted and **denied** if possible).
2. In CSW, open **Flow Analysis** — confirm flows appear with firewall-sourced endpoints.
3. Check connector health: **Manage → Workloads → Connectors** → heartbeat/statistics.
4. Optional: filter flows where NetFlow was the source (`user_src_NETFLOW_IDENTIFIED` / `user_dst_NETFLOW_IDENTIFIED` in API exports).

**Video:** [Flow Analysis](https://www.youtube.com/watch?v=Tuw06kPjeyQ)

**NSEL event handling (summary):** flow-create and flow-update → exported; flow-denied → exported as **rejected**; connector sends **forward + reverse** flow observations.

### Step A5 — Label workloads and run discovery

1. Import or assign **labels** (CMDB, IPAM, manual) for firewall-visible subnets.
2. Run **Application Dependency Mapping (ADM)** on a scoped workspace.
3. Stay in **Monitor** mode until app owners validate dependencies.

**Videos:** [Basic Application Discovery](https://youtu.be/HGvtBonFiE4) · [Enhancing Application Discovery](https://youtu.be/4wa7PiHGUnM) · [ADM & Policy Analysis](https://www.youtube.com/watch?v=Jzzblea25UA)

---

## Part B — Policy enforcement on Secure Firewall (FMC Connector)

**Goal:** CSW **pushes** segmentation policy to **FTD** devices managed by FMC — dynamic objects and ACP rules update as inventory changes.

### Step B1 — Onboard firewalls in FMC

1. Ensure target **FTD devices** are managed by **FMC** or **cdFMC** (Security Cloud Control).
2. Confirm **Access Control Policy (ACP)** exists for the segmentation zone.
3. Document which **CSW scope** maps to which **ACP** (one ACP can map to one scope).

**Video:** [FMC Integration with Edge / Ingest / Appliance](https://youtu.be/13AZ33dpCxU)

### Step B2 — Create FMC Connector in CSW

1. **Manage → Workloads → Connectors → Cisco Secure Firewall Management Center Connector**.
2. Provide FMC **hostname**, **REST API credentials** (admin privileges).
3. Complete connectivity test; resolve proxy/TLS if using Secure Connector path to FMC.
4. For full procedure see: [CSW and FMC Integration Guide](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/integration/guide/b-csw-fmc-integration-guide.html) (Cisco.com).

### Step B3 — Map scope to Access Control Policy

For each enforcement boundary:

1. In CSW, map **Scope → ACP** on the FMC connector.
2. Choose enforcement mode per mapping:

| Mode | Behavior |
|------|----------|
| **Merge** | CSW rules coexist with existing FMC rules; CSW honors existing firewall policy |
| **Override** | CSW rules take precedence in the mapped section |
| **Rule ordering** | CSW rules at **top** or **bottom** of ACP |
| **Catch-all** | Use CSW catch-all or keep FMC default action |

3. Enable **Topology Awareness** so CSW only pushes rules to firewalls on the **traffic path** for each scope.

**Video:** [Policy Enforcement Overview](https://youtu.be/A8rOXQ-y4Cw) · [Where to Enforce](https://youtu.be/urFJyDERMFs) · [Policy Ordering](https://youtu.be/fG1Kn1C7QRM)

### Step B4 — Model policy in CSW, then enforce on firewall

1. Build or accept **ADM-recommended** policies in a **workspace** tied to the scope.
2. **Simulate / monitor** — review impact in CSW UI and Policy Visual.
3. When ready, **enable enforcement** on the workspace (customer change window).
4. CSW updates **Dynamic Objects** in FMC and **deploys** to FTD — verify in FMC that deployment succeeds.

**Videos:** [Policy Lifecycle](https://youtu.be/Cm-cUwRorDc) · [Policy Validation and Analysis](https://youtu.be/DgaZpQ0lnAI) · [Policy Visual and Quick Analysis](https://www.youtube.com/watch?v=uBxrJaVLHy4)

### Step B5 — Validate enforcement

| Test | Expected result |
|------|-----------------|
| Allowed business flow | Passes; CSW/FMC show permit |
| Intentionally denied flow | Blocked at FTD; CSW **Denied Connections** / flow disposition |
| New workload in scope | Dynamic object membership updates after inventory refresh |
| FMC deployment | No manual redeploy needed for CSW-driven changes (per integration design) |

---

## Recommended video learning path (firewall integration)

Watch in this order:

| Order | Video | Link |
|------:|-------|------|
| 1 | Connector Overview | [Watch](https://youtu.be/H6QxuouzeC8) |
| 2 | Connector Deployment | [Watch](https://youtu.be/H0as2ppS84Q) |
| 3 | Secure Workload & Firewall Integration — Part 1 (design) | [Watch](https://youtu.be/vdHjAl48SuI) |
| 4 | Part 2 (deployment patterns) | [Watch](https://www.youtube.com/watch?v=xpbg3s0vrcI) |
| 5 | Part 3 (enforcement & operations) | [Watch](https://www.youtube.com/watch?v=X65mwN7kJGg) |
| 6 | **Secure Workload & Secure Firewall Integration Updates** (current behavior) | [Watch](https://youtu.be/IEqbz44YvOQ) |
| 7 | FMC Integration (Edge / Ingest / Appliance) | [Watch](https://youtu.be/13AZ33dpCxU) |
| 8 | Where to Enforce | [Watch](https://youtu.be/urFJyDERMFs) |
| 9 | Policy Enforcement Overview | [Watch](https://youtu.be/A8rOXQ-y4Cw) |

**Read:** [Secure Workload & Secure Firewall White Paper](https://www.cisco.com/c/en/us/products/collateral/security/secure-workload/sec-workload-firewall-wp.html) · [Integration deep dive (secure.cisco.com)](https://secure.cisco.com/secure-workload/docs/secure-workload-whitepaper)

---

## NetFlow connector vs Secure Firewall connector vs API flowsearch

| Method | Use when | Timeout / scale notes |
|--------|----------|------------------------|
| **Secure Firewall Connector (NSEL)** | Agentless visibility from **FTD/ASA** | Real-time ingest to CSW; plan ~45k fps per connector |
| **NetFlow connector** | Flows from **switches/routers** (v9/IPFIX) | Up to ~15k fps per connector (CSW 4.0 doc limit) |
| **Host agents** | Process-level + host enforcement | Best fidelity; not a NetFlow path |
| **OpenAPI `flowsearch`** | Export/query **already ingested** flows | Paginate (`limit` ~500 + `offset`); ≤24h per call; use 1h chunks for large pulls — **not** a substitute for connector ingest |

If API **`flowsearch`** times out when pulling large NetFlow-derived datasets, use **pagination and time chunks** — see CSW POV tooling guidance. Connector ingest and API export are different stages.

---

## POV evidence checklist (firewall integration)

| Evidence | Proves |
|----------|--------|
| Connector enabled + heartbeat healthy | Ingest path live |
| Sample flows in Flow Analysis (NSEL source) | Visibility |
| ADM map for scoped app | Discovery value |
| FMC connector connected | Enforcement path configured |
| Scope ↔ ACP mapping screenshot | Topology binding |
| Monitor-mode policy stats | Safe pre-enforce review |
| Post-enforce deny test + FMC hit | Enforcement works |
| Allowed transaction test | No business break |

---

## Common pitfalls

| Pitfall | Fix |
|---------|-----|
| Confusing **NetFlow connector** with **Secure Firewall connector** | Use Secure Firewall connector for NSEL from FTD/ASA |
| Expecting **NSEL alone** to enforce policy | Enforcement requires **FMC connector** + workspace enforce |
| Wrong Ingest **IP:port** on firewall | Copy from CSW connector details page |
| **Merge vs Override** misunderstood | Align with security architecture before first push |
| Pulling all flows via **API** in one call | Paginate; chunk time windows |
| Skipping **Secure Connector** on SaaS | Ingest cannot reach tenant without it |
| Treating this as **ACI** integration | ACI uses separate connector and policy model |

---

## Official Cisco documentation

| Document | URL |
|----------|-----|
| CSW 4.0 SaaS — Connectors | [Configure and Manage Connectors](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-saas-v40/m-connectors.html) |
| CSW + Secure Firewall white paper | [sec-workload-firewall-wp.html](https://www.cisco.com/c/en/us/products/collateral/security/secure-workload/sec-workload-firewall-wp.html) |
| CSW + FMC integration guide | [b-csw-fmc-integration-guide.html](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/integration/guide/b-csw-fmc-integration-guide.html) |
| ASA NSEL configuration | [ASA NetFlow Implementation Guide](https://www.cisco.com/c/en/us/td/docs/security/asa/asa-netflow/asa-netflow.html) |

---

## Related repos

- [CSW-Secure-Firewall-Integration-Guide](https://github.com/chandrapati/CSW-Secure-Firewall-Integration-Guide) — **this guide** (standalone repo)
- [CSW-User-Education](https://github.com/chandrapati/CSW-User-Education) — video library and onboarding runbook
- [CSW-Policy-Lifecycle](https://github.com/chandrapati/CSW-Policy-Lifecycle) — ADM → Monitor → Enforce workflow
- [CSW-Agent-Installation-Guide](https://github.com/chandrapati/CSW-Agent-Installation-Guide) — host agent deployment
- [CSW-Identity-Integration-Guide](https://github.com/chandrapati/CSW-Identity-Integration-Guide) — AD / Entra / DC user identity
- [CSW-ServiceNow-Connector-Guide](https://github.com/chandrapati/CSW-ServiceNow-Connector-Guide) — ServiceNow inventory enrichment
- [csw-splunk-integration](https://github.com/chandrapati/csw-splunk-integration) — Splunk / SIEM integration
- [CSW-Compliance-Mapping](https://github.com/chandrapati/CSW-Compliance-Mapping) — compliance framework mappings
- [CSW_POV_Template](https://github.com/chandrapati/CSW_POV_Template) — reusable POV toolkit
- [csw-logs-check](https://github.com/chandrapati/csw-logs-check) — host agent enforcement timing from diagnostic bundles
- [csw_blast_radius_demo](https://github.com/chandrapati/csw_blast_radius_demo) — blast-radius demo
- [CSW-SE-Helper-Repo](https://github.com/chandrapati/CSW-SE-Helper-Repo) — SE helper utilities
