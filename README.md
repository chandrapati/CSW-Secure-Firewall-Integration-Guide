# Cisco Secure Workload — Secure Firewall Integration Guide

Step-by-step reference for integrating **Cisco Secure Workload (CSW)** with **Cisco Secure Firewall** (FTD/ASA + FMC): **NetFlow / NSEL visibility** and **FMC policy enforcement**.

Written for security engineers, network/firewall teams, and POV delivery staff who need agentless east-west segmentation or network enforcement alongside host agents.

> **Disclaimer:** Companion learning material — not official Cisco product documentation. Validate design, supported versions, and limits against your tenant and the [CSW 4.0 connector documentation](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-saas-v40/m-connectors.html) before production.

**Full guide (Markdown):** [`docs/INTEGRATION-GUIDE.md`](docs/INTEGRATION-GUIDE.md) · **Word:** [`docs/CSW-Secure-Firewall-Integration-Guide.docx`](docs/CSW-Secure-Firewall-Integration-Guide.docx) · **PDF:** [`docs/CSW-Secure-Firewall-Integration-Guide.pdf`](docs/CSW-Secure-Firewall-Integration-Guide.pdf)

---

## Two integrations — do not mix them

| Integration | Connector | Purpose | Data direction |
|-------------|-----------|---------|----------------|
| **Flow visibility (NetFlow / NSEL)** | **Secure Firewall Connector** on **Ingest appliance** | CSW sees flows from FTD/ASA without agents | Firewall → Ingest → CSW |
| **Policy enforcement** | **FMC Connector** | CSW pushes segmentation rules to FTD via FMC | CSW → FMC → FTD |

- **NetFlow connector** (switches/routers) is a **third** path — not for firewall NSEL.
- **ACI integration** is separate — see [ACI and CSW Integration](https://www.youtube.com/watch?v=u7jh3Zw1hlg).

---

## Architecture

```text
                    VISIBILITY PATH (NSEL)
  ┌─────────────────┐     NSEL (NetFlow v9)      ┌──────────────────────┐
  │ Secure Firewall │ ─────────────────────────▶ │ Secure Firewall       │
  │ FTD / ASA       │     port 4729 typical       │ Connector (Docker on  │
  └─────────────────┘                             │ Ingest appliance)     │
                                                    └──────────┬───────────┘
                                                               │
                                                    Secure Connector (SaaS)
                                                               ▼
                                                    ┌──────────────────────┐
                                                    │ Cisco Secure Workload │
                                                    └──────────┬───────────┘
                                                               │
                    ENFORCEMENT PATH (FMC)                       ▼
  ┌─────────────────┐     REST API + deploy      ┌──────────────────────┐
  │ Secure Firewall │ ◀───────────────────────── │ FMC Connector         │
  │ FTD (managed)    │     Dynamic objects + ACP   │ (CSW UI)              │
  └─────────────────┘                             └──────────────────────┘
           ▲
  ┌────────┴────────┐
  │ FMC / cdFMC      │
  └─────────────────┘
```

**SaaS CSW:** Secure Connector (or corporate proxy) is required between the Ingest appliance and the tenant.

---

## Prerequisites

| # | Requirement | Notes |
|---|-------------|--------|
| 1 | CSW tenant (SaaS or on-prem) with admin access | Scopes and workspaces ready |
| 2 | **Secure Workload Ingest** virtual appliance | Provisioned and **Active** |
| 3 | **Secure Connector** (SaaS) or network path to CSW | Mandatory for SaaS ingest |
| 4 | **FMC** or **cdFMC** | For enforcement only |
| 5 | **FMC REST API user** with admin privileges | Required for FMC connector |
| 6 | **FTD or ASA** in scope | NSEL for visibility; FTD for enforcement |
| 7 | **Access Control Policy (ACP)** mapped to CSW scope | Per enforcement design |
| 8 | Capacity planning | ~45k fps per Secure Firewall connector (Cisco white paper) |

---

## Part A — NetFlow / NSEL visibility (Secure Firewall Connector)

**Goal:** CSW receives bidirectional flow records from the firewall **without** host agents.

### Step A1 — Plan ingest placement

1. Confirm **Ingest appliance** is deployed: **Manage → Workloads → Virtual Appliances**.
2. Note the **connector slot IP** and **listening port** (default **4729**).
3. Ensure firewalls can reach Ingest on **UDP 4729**.
4. If using VRFs: **Manage → Agents → Configuration → Agent Remote VRF Configurations** (one connector per VRF).

**Videos:** [Connector Overview](https://youtu.be/H6QxuouzeC8) · [Connector Deployment](https://youtu.be/H0as2ppS84Q)

### Step A2 — Enable Secure Firewall Connector in CSW

1. **Manage → Workloads → Connectors**.
2. Select **Cisco Secure Firewall Connector** (formerly ASA connector).
3. **Enable** on the **Ingest appliance**.
4. Wait for status **Enabled**; record connector **ID** and **IP:port** for firewall config.

**Limits (CSW 4.0):** max **1** Secure Firewall connector per Ingest appliance; **10** per tenant.

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

**FTD:** Configure NSEL export toward the same Ingest endpoint per FMC/device template and the [ASA NetFlow Implementation Guide](https://www.cisco.com/c/en/us/td/docs/security/asa/asa-netflow/asa-netflow.html).

**Videos:** [Part 1 — design](https://youtu.be/vdHjAl48SuI) · [Part 2 — deployment](https://www.youtube.com/watch?v=xpbg3s0vrcI) · [Part 3 — operations](https://www.youtube.com/watch?v=X65mwN7kJGg) · [**2025–2026 updates**](https://youtu.be/IEqbz44YvOQ)

### Step A4 — Validate flow ingestion

1. Generate permitted and **denied** test traffic through the firewall.
2. Open **Flow Analysis** in CSW — confirm firewall-sourced flows.
3. Check connector heartbeat: **Manage → Workloads → Connectors**.

**Video:** [Flow Analysis](https://www.youtube.com/watch?v=Tuw06kPjeyQ)

### Step A5 — Label workloads and run discovery

1. Assign **labels** (CMDB, IPAM, manual) for firewall-visible subnets.
2. Run **Application Dependency Mapping (ADM)** on a scoped workspace.
3. Stay in **Monitor** mode until app owners validate dependencies.

**Videos:** [Basic Application Discovery](https://youtu.be/HGvtBonFiE4) · [Enhancing Application Discovery](https://youtu.be/4wa7PiHGUnM) · [ADM & Policy Analysis](https://www.youtube.com/watch?v=Jzzblea25UA)

---

## Part B — Policy enforcement on Secure Firewall (FMC Connector)

**Goal:** CSW pushes segmentation policy to **FTD** via FMC — dynamic objects and ACP rules update as inventory changes.

### Step B1 — Onboard firewalls in FMC

1. Ensure **FTD devices** are managed by **FMC** or **cdFMC**.
2. Confirm **Access Control Policy (ACP)** exists for the segmentation zone.
3. Document **CSW scope → ACP** mapping (one ACP per scope).

**Video:** [FMC Integration (Edge / Ingest / Appliance)](https://youtu.be/13AZ33dpCxU)

### Step B2 — Create FMC Connector in CSW

1. **Manage → Workloads → Connectors → Cisco Secure Firewall Management Center Connector**.
2. Provide FMC **hostname** and **REST API credentials** (admin privileges).
3. Complete connectivity test; resolve proxy/TLS if needed.
4. Full procedure: [CSW and FMC Integration Guide](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/integration/guide/b-csw-fmc-integration-guide.html).

### Step B3 — Map scope to Access Control Policy

1. Map **Scope → ACP** on the FMC connector.
2. Choose enforcement mode:

| Mode | Behavior |
|------|----------|
| **Merge** | CSW rules coexist with existing FMC rules |
| **Override** | CSW rules take precedence in the mapped section |
| **Rule ordering** | CSW rules at **top** or **bottom** of ACP |

3. Enable **Topology Awareness** so CSW pushes only to firewalls on the traffic path.

**Videos:** [Policy Enforcement Overview](https://youtu.be/A8rOXQ-y4Cw) · [Where to Enforce](https://youtu.be/urFJyDERMFs) · [Policy Ordering](https://youtu.be/fG1Kn1C7QRM)

### Step B4 — Model policy, then enforce

1. Build or accept **ADM-recommended** policies in a **workspace** tied to the scope.
2. **Simulate / monitor** — review Policy Visual impact.
3. In a change window, **enable enforcement** on the workspace.
4. Verify CSW updates **Dynamic Objects** in FMC and **deploys** to FTD.

**Videos:** [Policy Lifecycle](https://youtu.be/Cm-cUwRorDc) · [Policy Validation and Analysis](https://youtu.be/DgaZpQ0lnAI)

### Step B5 — Validate enforcement

| Test | Expected result |
|------|-----------------|
| Allowed business flow | Passes; CSW/FMC show permit |
| Intentionally denied flow | Blocked at FTD; CSW denied-flow evidence |
| New workload in scope | Dynamic object membership updates |
| FMC deployment | CSW-driven changes deploy without manual FMC redeploy |

---

## Recommended video learning path

| Order | Video | Link |
|------:|-------|------|
| 1 | Connector Overview | [Watch](https://youtu.be/H6QxuouzeC8) |
| 2 | Connector Deployment | [Watch](https://youtu.be/H0as2ppS84Q) |
| 3 | Firewall Integration Part 1 | [Watch](https://youtu.be/vdHjAl48SuI) |
| 4 | Part 2 | [Watch](https://www.youtube.com/watch?v=xpbg3s0vrcI) |
| 5 | Part 3 | [Watch](https://www.youtube.com/watch?v=X65mwN7kJGg) |
| 6 | **2025–2026 integration updates** | [Watch](https://youtu.be/IEqbz44YvOQ) |
| 7 | FMC + Edge / Ingest / Appliance | [Watch](https://youtu.be/13AZ33dpCxU) |
| 8 | Where to Enforce | [Watch](https://youtu.be/urFJyDERMFs) |
| 9 | Policy Enforcement Overview | [Watch](https://youtu.be/A8rOXQ-y4Cw) |

---

## POV evidence checklist

| Evidence | Proves |
|----------|--------|
| Connector enabled + heartbeat healthy | Ingest path live |
| Sample flows in Flow Analysis | NSEL visibility |
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
| Expecting **NSEL alone** to enforce | Enforcement requires **FMC connector** + workspace enforce |
| Wrong Ingest **IP:port** on firewall | Copy from CSW connector details page |
| Bulk **`flowsearch`** API timeouts | Paginate (`limit` ~500); ≤24h windows; 1h chunks — not a substitute for connector ingest |
| Skipping **Secure Connector** on SaaS | Ingest cannot reach tenant without it |

---

## Official Cisco documentation

| Document | URL |
|----------|-----|
| CSW 4.0 SaaS — Connectors | [m-connectors.html](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-saas-v40/m-connectors.html) |
| CSW + Secure Firewall white paper | [sec-workload-firewall-wp.html](https://www.cisco.com/c/en/us/products/collateral/security/secure-workload/sec-workload-firewall-wp.html) |
| CSW + FMC integration guide | [b-csw-fmc-integration-guide.html](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/integration/guide/b-csw-fmc-integration-guide.html) |
| ASA NSEL configuration | [ASA NetFlow Implementation Guide](https://www.cisco.com/c/en/us/td/docs/security/asa/asa-netflow/asa-netflow.html) |

---

## Cisco Secure Workload — companion repositories

This repo is the **Secure Firewall integration** chapter of the public CSW practitioner toolkit. Use the suggested path for a new customer engagement.

| Repository | What it covers | When to use |
|------------|----------------|-------------|
| [**CSW-User-Education**](https://github.com/chandrapati/CSW-User-Education) | Intro, 62-video library, onboarding runbook, POV evidence checklist | First stop for anyone new to CSW |
| [**CSW-Secure-Firewall-Integration-Guide**](https://github.com/chandrapati/CSW-Secure-Firewall-Integration-Guide) | **This repo** — NSEL ingest + FMC enforcement step-by-step | Firewall + NetFlow in scope |
| [**CSW-Agent-Installation-Guide**](https://github.com/chandrapati/CSW-Agent-Installation-Guide) | Host agent install: Linux, Windows, cloud, containers, agentless | Deploying agents (complements firewall path) |
| [**CSW-Policy-Lifecycle**](https://github.com/chandrapati/CSW-Policy-Lifecycle) | ADM → Monitor → Simulate → Enforce + day-2 ops | After visibility; before enforce |
| [**CSW-Identity-Integration-Guide**](https://github.com/chandrapati/CSW-Identity-Integration-Guide) | AD, Entra ID, DC user-identity reporting | User/group labels in policy |
| [**CSW-ServiceNow-Connector-Guide**](https://github.com/chandrapati/CSW-ServiceNow-Connector-Guide) | ServiceNow inventory enrichment connector | CMDB-driven labels |
| [**csw-splunk-integration**](https://github.com/chandrapati/csw-splunk-integration) | CSW → Splunk via Syslog + Security Cloud App | SOC / SIEM integration |
| [**CSW-Compliance-Mapping**](https://github.com/chandrapati/CSW-Compliance-Mapping) | 30+ framework mappings (HIPAA, PCI, NIST, ISO, etc.) | GRC / audit / CISO reporting |
| [**CSW_POV_Template**](https://github.com/chandrapati/CSW_POV_Template) | Reusable POV engagement toolkit | Starting a new POV |
| [**csw-logs-check**](https://github.com/chandrapati/csw-logs-check) | Cursor skill: analyze agent diagnostic bundles | Host enforce timing / validation |
| [**csw_blast_radius_demo**](https://github.com/chandrapati/csw_blast_radius_demo) | Hands-on blast-radius reduction demo | Customer demos and labs |
| [**CSW-SE-Helper-Repo**](https://github.com/chandrapati/CSW-SE-Helper-Repo) | SE utilities and scratch tooling | Internal SE workflows |

### Suggested learning path

```text
CSW-User-Education
    → CSW-Agent-Installation-Guide (if agents in scope)
    → CSW-Secure-Firewall-Integration-Guide (if firewall in scope)  ← you are here
    → CSW-Identity-Integration-Guide (if user identity in scope)
    → CSW-Policy-Lifecycle
    → csw-splunk-integration
    → CSW-Compliance-Mapping
    → CSW_POV_Template (wrap-up)
```

For **host enforcement timing** validation after agents are deployed, use [**csw-logs-check**](https://github.com/chandrapati/csw-logs-check) alongside this guide when both agents and firewalls are in the same POV.

---

## Repository layout

| Path | Description |
|------|-------------|
| `README.md` | Overview, step-by-step integration, video links, companion repos |
| `docs/INTEGRATION-GUIDE.md` | Full Markdown guide (same content, expanded sections) |
| `docs/CSW-Secure-Firewall-Integration-Guide.docx` | Word export for customer hand-off |
| `docs/CSW-Secure-Firewall-Integration-Guide.pdf` | PDF export |
| `LICENSE` | MIT |

---

## Regenerate Word / PDF

After editing `docs/INTEGRATION-GUIDE.md`:

```bash
cd docs
pandoc INTEGRATION-GUIDE.md --from gfm --to docx --toc --toc-depth=2 \
  -o CSW-Secure-Firewall-Integration-Guide.docx

rm -rf /tmp/lo_csw_fw_profile
soffice --headless \
  -env:UserInstallation=file:///tmp/lo_csw_fw_profile \
  --convert-to pdf CSW-Secure-Firewall-Integration-Guide.docx
```

---

## Disclaimer

Patterns and steps in this repository are for **informational and reference purposes only**. They are not a substitute for official Cisco Secure Workload documentation, your change-management process, or a qualified consulting engagement. When this guide and the User Guide disagree, **the User Guide wins**.

For release-specific behavior, sizing, or licensing, contact your **Cisco Secure Workload account team**.
