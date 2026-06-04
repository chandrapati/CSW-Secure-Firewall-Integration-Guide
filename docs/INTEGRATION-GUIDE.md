---
title: "CSW + Cisco Secure Firewall Integration Guide"
subtitle: "NetFlow (NSEL) visibility and FMC policy enforcement — customer reference"
author: "Cisco Secure Workload User Education"
date: "2026-06-04"
---

# Cisco Secure Workload + Cisco Secure Firewall

**NetFlow (NSEL) ingestion and policy enforcement on the firewall**

> **Disclaimer:** Companion learning material — not official Cisco product documentation. Validate design, supported versions, and limits against your tenant, release notes, and:
> - [FMC Integration Guide (Cisco.com)](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/integration/fmc/cisco-secure-workload-and-fmc-integration-guide.html)
> - [Secure Workload & Firewall deep dive (secure.cisco.com)](https://secure.cisco.com/secure-workload/docs/secure-workload-whitepaper)

**Architecture diagrams:** [`ARCHITECTURE.md`](ARCHITECTURE.md) only — do not duplicate diagrams in other docs.

---

## Jorge Quintero — 3-part video series

Cisco Technical Marketing Engineer **Jorge Quintero** ([Cisco Secure Workload YouTube channel](https://www.youtube.com/@ciscosecureworkload)) recorded the primary integration walkthrough. Watch **before** implementing.

| Part | Title | Topics covered | Watch |
|------|-------|----------------|-------|
| **1** | Secure Workload & Firewall Integration | Introduction, design, high-level architecture, use cases | [YouTube](https://youtu.be/vdHjAl48SuI) |
| **2** | Secure Workload & Firewall Integration | Deployment patterns, NSEL ingest, FMC connector setup, policy flow | [YouTube](https://www.youtube.com/watch?v=xpbg3s0vrcI) |
| **3** | Secure Workload & Firewall Integration | Enforcement, telemetry validation, operations, troubleshooting | [YouTube](https://www.youtube.com/watch?v=X65mwN7kJGg) |

**Follow-up:** [Secure Workload & Secure Firewall Integration Updates (2025–2026)](https://youtu.be/IEqbz44YvOQ) — latest behavior on the same channel.

**Related Cisco Live session:** Jorge Quintero — *Solving the Segmentation Puzzle with Secure Workload* ([BRKSEC-2161](https://www.ciscolive.com/c/dam/r/ciscolive/global-event/docs/2024/pdf/BRKSEC-2161.pdf)).

---

## Executive summary

In hybrid multi-cloud environments, policy controls often fragment across host firewalls, network firewalls, SDN, and cloud security groups — managed by different teams. **Cisco Secure Workload (CSW)** and **Cisco Secure Firewall** together provide a unified microsegmentation path for workloads where **host agents cannot be installed**, while still supporting mixed agent + agentless estates.

Three integrated value streams:

| Use case | What it does | Primary path |
|----------|--------------|--------------|
| **Microsegmentation** | Discover agentless flows (NSEL), model policy (ADM), enforce on FTD via FMC | Secure Firewall Connector + FMC Connector |
| **Virtual Patch** | Export workload CVEs to FMC; apply compensating IPS rules until patches ship | FMC Connector (Virtual Patch tab) + agents |
| **Rapid Threat Containment** | FMC detects anomaly → Remediation Module → CSW quarantine guardrails | FMC Remediation Module + CSW labels |

---

## Solution overview

CSW delivers three platform capabilities that intersect with Secure Firewall:

1. **Zero Trust Microsegmentation** — agent and agentless discovery, ADM-based policy, enforcement on host firewalls, DPUs, **network firewalls**, load balancers, and cloud controls.
2. **Vulnerability Detection and Protection** — runtime CVE visibility from agents; **virtual patch via Secure Firewall IPS**.
3. **Behavioral Detection and Protection** — process/forensic analytics; **Rapid Threat Containment** with FMC correlation + CSW guardrails.

---

## Architecture

See **[ARCHITECTURE.md](ARCHITECTURE.md)** for all customer-facing diagrams (high-level integration, NSEL/FMC sequences, scope mapping, insertion options, virtual patch, rapid threat containment). This guide covers steps and design detail only — no duplicate diagrams here.

### Two connectors — do not mix them

| Integration | Connector | Purpose | Data direction |
|-------------|-----------|---------|----------------|
| **Flow visibility (NetFlow / NSEL)** | **Secure Firewall Connector** on **Ingest appliance** | CSW sees flows from FTD/ASA without agents | Firewall → Ingest → CSW |
| **Policy enforcement** | **FMC Connector** (Cisco Secure Firewall in UI) | CSW pushes segmentation to FTD via FMC | CSW → FMC → FTD |

| Also available | Use when |
|----------------|----------|
| **NetFlow connector** | Switches/routers export NetFlow v9/IPFIX — **not** FTD NSEL |
| **Host agents** | Process-level visibility + host enforcement |
| **ACI connector** | Fabric policy — separate from this FTD/FMC guide |

---

## Workload protection level (design first)

Before choosing firewall insertion mode, define the **security/trust boundary** your organization uses (subnet, zone, app tier, persona). The white paper recommends aligning network engineers and app owners on a measurable protection level — commonly **subnet-level** for agentless FTD insertion.

| Persona | Typical boundary | Agent vs agentless |
|---------|------------------|-------------------|
| App / DevOps | App tier, namespace | Often agent-based |
| Network / Firewall | Subnet, zone, VRF | Often agentless (this guide) |
| Cloud | VPC/VNet, security group | Mixed |

This drives whether you use **L2 transparent** (fine-grained), **L3 routed** (faster zone segmentation), or **cloud hub** patterns — see [Insertion options](#firewall-insertion-options).

---

## Prerequisites checklist

### CSW + Ingest (visibility)

| # | Requirement | Notes |
|---|-------------|--------|
| 1 | CSW tenant (SaaS or on-prem) with admin access | Scopes and workspaces ready |
| 2 | **Secure Workload Ingest** virtual appliance | **Active** before enabling connector |
| 3 | **Secure Connector** (SaaS) or network path | Mandatory for SaaS |
| 4 | FTD/ASA can reach Ingest on **UDP 4729** | Copy IP:port from connector details |
| 5 | Capacity | **45k fps** per connector; **135k fps** per Ingest appliance (on-prem) |

### FMC (enforcement)

| # | Requirement | Notes |
|---|-------------|--------|
| 1 | **FMC** or **cdFMC** with supported FTD devices | FTD managed by FMC only |
| 2 | **FMC REST API user** with **administrative privileges** | Required for connector |
| 3 | FTD associated with FMC; ACP assigned; deploy verified | Test FMC→FTD deploy before CSW |
| 4 | **Secure Connector** if SaaS CSW cannot reach FMC directly | HTTPS **443** to FMC |
| 5 | FMC HA: include **standby FMC IP** in connector | Auto-switch on failover |
| 6 | Supported versions | Segmentation: FMC **7.0.x+** (dynamic objects **7.0.1+**); Virtual Patch: **7.2+** |

### Virtual Patch (optional)

- CSW **agents** on target workloads
- Threat intelligence **CVE data pack** on CSW cluster
- Workloads use **FTD as default gateway** (traffic through IPS path)
- FMC network discovery sees hosts under **Analysis → Network Map**

---

## Part A — NetFlow / NSEL visibility (Secure Firewall Connector)

**Goal:** CSW receives bidirectional, stateful flow records from the firewall **without** host agents.

### Step A1 — Plan ingest placement

1. **Manage → Workloads → Virtual Appliances** — confirm Ingest is **Active**.
2. Record **connector slot IP** and port (**4729** default).
3. Open firewall path: **UDP 4729** from FTD/ASA to Ingest.
4. **VRF:** **Manage → Agents → Configuration → Agent Remote VRF Configurations** — one connector per VRF.

**Videos:** [Connector Overview](https://youtu.be/H6QxuouzeC8) · [Connector Deployment](https://youtu.be/H0as2ppS84Q)

### Step A2 — Enable Secure Firewall Connector in CSW

1. **Manage → Workloads → Connectors**.
2. Select **Cisco Secure Firewall Connector** (formerly ASA connector).
3. **Enable** on **Ingest appliance** (not Edge-only).
4. Record connector **ID** and **IP:port** from connector details page.

**Limits:** **1** Secure Firewall connector per Ingest appliance; **10** per tenant.

### Step A3 — Enable NSEL on Secure Firewall

**ASA example** (replace `<INGEST_CONNECTOR_IP>`):

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

**FTD:** Configure NSEL export via FMC/device template — [ASA NetFlow Implementation Guide](https://www.cisco.com/c/en/us/td/docs/security/asa/asa-netflow/asa-netflow.html).

**Videos (Jorge Quintero, 3-part series):** [Part 1](https://youtu.be/vdHjAl48SuI) · [Part 2](https://www.youtube.com/watch?v=xpbg3s0vrcI) · [Part 3](https://www.youtube.com/watch?v=X65mwN7kJGg) · [2025–2026 updates](https://youtu.be/IEqbz44YvOQ)

### Step A4 — Validate flow ingestion

1. Generate permitted and **denied** test traffic.
2. **Flow Analysis** — confirm firewall-sourced endpoints.
3. Connector heartbeat healthy under **Manage → Workloads → Connectors**.
4. Optional API filters: `user_src_NETFLOW_IDENTIFIED` / `user_dst_NETFLOW_IDENTIFIED`.

**NSEL handling:** flow-create/update → exported; flow-denied → **rejected** disposition; connector sends **forward + reverse** flows; **NAT flow-stitching** for end-to-end visibility.

**Video:** [Flow Analysis](https://www.youtube.com/watch?v=Tuw06kPjeyQ)

### Step A5 — Label workloads and run discovery

1. Import **labels** from CMDB (ServiceNow), IPAM, or manual assignment.
2. Run **ADM** on scoped workspace.
3. Remain in **Monitor** mode until app owners validate dependencies.

**Videos:** [Basic Application Discovery](https://youtu.be/HGvtBonFiE4) · [Enhancing Application Discovery](https://youtu.be/4wa7PiHGUnM) · [ADM & Policy Analysis](https://www.youtube.com/watch?v=Jzzblea25UA)

---

## Part B — Policy enforcement on Secure Firewall (FMC Connector)

**Goal:** CSW pushes **L3/L4 segmentation** to FTD via FMC using **Dynamic Objects** — inventory changes refresh membership without manual object edits.

> CSW orchestrated rules are **L3/L4 east-west microsegmentation**. Layer-7 FMC features should not be added to CSW-controlled rules (not supported).

### Step B1 — Onboard firewalls in FMC

1. FTD devices registered to **FMC** or **cdFMC**.
2. Each FTD assigned to an **Access Control Policy (ACP)**.
3. Verify FMC can **deploy** to FTD and traffic behaves as expected.
4. Document **one scope → one ACP** mapping plan (1:1 only).

**Video:** [FMC Integration (Edge / Ingest / Appliance)](https://youtu.be/13AZ33dpCxU)

### Step B2 — Create FMC Connector in CSW

1. **Manage → Workloads → Connectors → Firewall → Cisco Secure Firewall**.
2. Click **Configure your new connector here**.
3. Complete **New Connection** fields:

| Field | Description |
|-------|-------------|
| **Connector Name** | Unique name for this FMC connection |
| **Username / Password** | FMC REST API credentials (admin privileges) |
| **CA Certificate** | FMC CA cert for TLS validation, or **Disable SSL** on trusted networks |
| **Server IP/FQDN and Port** | FMC hostname or IP (**443**) |
| **HTTP Proxy** | If required: `<proxy.host>:<proxy.port>` |
| **Secure Connector** | Enable when SaaS CSW reaches FMC through Secure Connector |

4. Click **Create**; verify **Event Log** tab for errors.

**Official procedure:** [FMC Integration Guide — Connection Settings](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/integration/fmc/cisco-secure-workload-and-fmc-integration-guide.html)

### Step B3 — Map scope to Access Control Policy

1. Open FMC connector → **Segmentation** tab → **+ Add**.
2. **Add ACP Mapping:**
   - Select **Access Policy** (ACP) from FMC.
   - Map to exactly **one Scope**.
   - **Use Secure Workload Catch-All** — CSW catch-all in Default section, or use FMC default action.
   - **Enforcement Mode:**
     - **Merge** — CSW rules coexist with FMC rules; configure priority.
     - **Override** — CSW replaces user-created rules in mapped sections.
   - **Absolute / Default Policies priority** (Merge only):
     - Insert **above** or **below** existing Mandatory/Default rules per section.
3. Click **Submit**.

#### Scope mapping strategy

| Model | When | Behavior |
|-------|------|----------|
| **Child / leaf scope** | Single application behind firewall | Push that app's policies + parent guardrails |
| **Parent scope** | Multiple applications | Push parent + eligible child policies (hierarchical) |

#### Topology awareness

CSW pushes rules only to FTD devices on the **traffic path** for the mapped scope — avoids rule sprawl across unrelated firewalls.

**Videos:** [Policy Enforcement Overview](https://youtu.be/A8rOXQ-y4Cw) · [Where to Enforce](https://youtu.be/urFJyDERMFs) · [Policy Ordering](https://youtu.be/fG1Kn1C7QRM)

### Step B4 — What CSW creates in FMC

When enforcement is enabled on a workspace, CSW converts policies to FMC rules:

| FMC artifact | Prefix | ACP section | Purpose |
|--------------|--------|-------------|---------|
| Golden rules | `Workload_golden__` | Mandatory | CSW ↔ agent communication behind FW |
| Segmentation rules | `Workload__` | Mandatory / Default | Enforced workspace policies |
| Catch-all rules | `Workload_ca__` | Default | If **Use Secure Workload Catch-All** enabled |
| Dynamic objects | `WorkloadObj__` | Objects | Scope/inventory IP membership |

| CSW policy type | FMC placement |
|-----------------|---------------|
| **Absolute** | Mandatory category |
| **Default** | Default category |

> **Do not manually edit** `Workload_*` rules in FMC — CSW restores them on next push. Independent FMC rules are safe in **Merge** mode if they avoid these prefixes.

### Step B5 — Model policy, then enforce

1. Build or accept **ADM-recommended** policies in workspace tied to scope.
2. **Simulate / Monitor** — Policy Visual and flow analysis.
3. Change window: **enable enforcement** on workspace.
4. Confirm FMC deployment succeeds; verify Dynamic Objects update as inventory changes.

**Videos:** [Policy Lifecycle](https://youtu.be/Cm-cUwRorDc) · [Policy Validation and Analysis](https://youtu.be/DgaZpQ0lnAI)

### Step B6 — Validate enforcement

| Test | Expected result |
|------|-----------------|
| Allowed business flow | Passes; CSW/FMC show permit |
| Intentionally denied flow | Blocked at FTD; CSW rejected-flow evidence |
| New workload in scope | `WorkloadObj__` membership updates |
| FMC deployment | Automatic on CSW policy change |
| Compliance monitoring | Alerts on rejected flows / policy deviation |

Check **Event Log** on FMC connector for push errors (Information / Warning / Error).

---

## Part C — Virtual Patch (optional)

Compensating **IPS** control for CVEs until permanent patches are deployed.

1. FMC connector → **Virtual Patch** tab → **Create a Virtual Patching Rule**.
2. Define **Host Query** (scope/filter) and **CVE Query** (CVSS, Cisco Risk Score, etc.).
3. CSW exports matching CVEs to FMC **Third-Party Vulnerabilities**.
4. FMC admin runs **Cisco Recommended Rules** to map CVEs → Snort signatures.
5. Apply IPS policy on ACP rule for affected traffic flows.

**Prerequisites:** Agents on workloads; CVE data pack; FTD as gateway; FMC **7.2+**.

---

## Part D — Rapid Threat Containment (optional)

Automated quarantine when FMC detects malicious behavior:

1. Install **FMC Remediation Module for Secure Workload** on FMC.
2. Configure CSW cluster IP, root scope, API key (**User Data Upload** permission).
3. Define **Correlation Rules** and **Correlation Policy** responses.
4. On trigger, Remediation Module sends affected IPs to CSW via API.
5. CSW **guardrail policies** (quarantine labels) enforce across agents and FTD.

---

## Firewall insertion options

| Mode | Segmentation | NSEL visibility | Best fit |
|------|--------------|-----------------|----------|
| **L2 Transparent** | Intra- + inter-subnet | Full | Legacy OS, fine-grained, localized |
| **L3 Routed** | Inter-subnet only | Inter-subnet only | Quick zone segment, dev/non-prod |
| **ACI Service Graph PBR** | Intra + inter EPG/ESG | Full (FW in path) | ACI + FTD multi-policy |
| **AWS hub VPC** | Inter-VPC | VPC logs + NSEL | Centralized cloud east-west |
| **AWS distributed** | Intra-VPC | VPC logs + NSEL | Per-VPC enforcement |
| **Azure hub VNet** | Intra- + inter-VNet | NSG logs + NSEL | UDR steering |
| **GCP hub VPC** | Inter-VPC | VPC logs + NSEL | Centralized GCP |

All patterns support **dual management**: CSW east-west + FMC north-south on the same ACP (Merge mode).

---

## FAQs (from Cisco white paper)

| Question | Answer |
|----------|--------|
| Mixed agent + agentless behind same mapped firewall? | Works. Agent-based rules may **also** appear in ACP (hierarchical policy). Primary use case remains agentless protection. |
| Map multiple scopes to one ACP? | **No** — strictly **1 scope : 1 ACP**. |
| Add Layer-7 / FMC features to CSW rules? | **Not supported** — CSW rules are L3/L4 east-west only. |
| Dual-management (CSW + FMC rules)? | **Merge** mode + rule ordering; preserve order after go-live. |
| Modify a CSW-owned rule in FMC? | CSW **overwrites** on next push. |
| Supported FMC versions? | Dynamic objects: **7.0.1+**; Virtual Patch: **7.2+**; qualified: **7.0.1**, **7.2.5**. |

---

## Recommended video learning path

| Order | Video | Presenter / channel | Link |
|------:|-------|---------------------|------|
| 1 | **Secure Workload & Firewall Integration — Part 1** | Jorge Quintero · [@ciscosecureworkload](https://www.youtube.com/@ciscosecureworkload) | [Watch](https://youtu.be/vdHjAl48SuI) |
| 2 | **Part 2** — deployment & policy flow | Jorge Quintero | [Watch](https://www.youtube.com/watch?v=xpbg3s0vrcI) |
| 3 | **Part 3** — enforcement & operations | Jorge Quintero | [Watch](https://www.youtube.com/watch?v=X65mwN7kJGg) |
| 4 | Integration Updates (2025–2026) | Cisco Secure Workload channel | [Watch](https://youtu.be/IEqbz44YvOQ) |
| 5 | Connector Overview | Cisco Secure Workload channel | [Watch](https://youtu.be/H6QxuouzeC8) |
| 6 | Connector Deployment | Cisco Secure Workload channel | [Watch](https://youtu.be/H0as2ppS84Q) |
| 7 | FMC + Edge / Ingest / Appliance | Cisco Secure Workload channel | [Watch](https://youtu.be/13AZ33dpCxU) |
| 8 | Where to Enforce | Cisco Secure Workload channel | [Watch](https://youtu.be/urFJyDERMFs) |
| 9 | Policy Enforcement Overview | Cisco Secure Workload channel | [Watch](https://youtu.be/A8rOXQ-y4Cw) |

---

## NetFlow connector vs Secure Firewall connector vs API flowsearch

| Method | Use when | Scale notes |
|--------|----------|-------------|
| **Secure Firewall Connector (NSEL)** | FTD/ASA agentless visibility | **45k fps** per connector; **135k fps** per Ingest |
| **NetFlow connector** | Switches/routers (v9/IPFIX) | ~15k fps per connector |
| **Host agents** | Process-level + host enforce | Best fidelity |
| **OpenAPI `flowsearch`** | Query ingested flows | Paginate; ≤24h; 1h chunks — not bulk ingest |

---

## POV evidence checklist

| Evidence | Proves |
|----------|--------|
| Connector enabled + heartbeat healthy | Ingest path live |
| Sample flows in Flow Analysis (NSEL) | Visibility |
| ADM map for scoped app | Discovery value |
| FMC connector connected + Event Log clean | Enforcement path |
| Scope ↔ ACP mapping screenshot | Topology binding |
| `Workload__` / `WorkloadObj__` in FMC | Policy push working |
| Monitor-mode policy stats | Safe pre-enforce review |
| Post-enforce deny test + FMC hit | Enforcement works |
| Allowed transaction test | No business break |

---

## Common pitfalls

| Pitfall | Fix |
|---------|-----|
| Confusing **NetFlow connector** with **Secure Firewall connector** | NSEL from FTD/ASA → Secure Firewall Connector only |
| Expecting **NSEL alone** to enforce | Requires **FMC connector** + workspace enforce |
| Editing `Workload_*` rules in FMC | CSW will overwrite — change policy in CSW |
| Multiple scopes on one ACP | Redesign — **1:1 mapping only** |
| L3 routed FW for intra-subnet segment | Use **L2 transparent** or host agents |
| Bulk **`flowsearch`** API timeouts | Paginate; chunk time windows |
| Skipping **Secure Connector** on SaaS | Required for Ingest and FMC reachability |

---

## Official Cisco documentation

| Document | URL |
|----------|-----|
| **FMC Integration Guide** | [cisco-secure-workload-and-fmc-integration-guide.html](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/integration/fmc/cisco-secure-workload-and-fmc-integration-guide.html) |
| **Deep dive white paper** | [secure-workload-whitepaper](https://secure.cisco.com/secure-workload/docs/secure-workload-whitepaper) |
| CSW 4.0 SaaS — Connectors | [m-connectors.html](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-saas-v40/m-connectors.html) |
| Marketing white paper | [sec-workload-firewall-wp.html](https://www.cisco.com/c/en/us/products/collateral/security/secure-workload/sec-workload-firewall-wp.html) |
| ASA NSEL configuration | [ASA NetFlow Implementation Guide](https://www.cisco.com/c/en/us/td/docs/security/asa/asa-netflow/asa-netflow.html) |

---

## Related repos

- [CSW-Secure-Firewall-Integration-Guide](https://github.com/chandrapati/CSW-Secure-Firewall-Integration-Guide) — **this repo**
- [CSW-User-Education](https://github.com/chandrapati/CSW-User-Education) — video library and onboarding
- [CSW-Policy-Lifecycle](https://github.com/chandrapati/CSW-Policy-Lifecycle) — ADM → Monitor → Enforce
- [CSW-Agent-Installation-Guide](https://github.com/chandrapati/CSW-Agent-Installation-Guide) — host agents
- [CSW-Identity-Integration-Guide](https://github.com/chandrapati/CSW-Identity-Integration-Guide) — AD / Entra identity
- [CSW-ServiceNow-Connector-Guide](https://github.com/chandrapati/CSW-ServiceNow-Connector-Guide) — CMDB labels
- [csw-splunk-integration](https://github.com/chandrapati/csw-splunk-integration) — SIEM
- [CSW-Compliance-Mapping](https://github.com/chandrapati/CSW-Compliance-Mapping) — compliance frameworks
- [csw-logs-check](https://github.com/chandrapati/csw-logs-check) — host enforce timing from agent logs
