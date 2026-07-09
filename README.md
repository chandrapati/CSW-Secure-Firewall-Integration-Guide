# Cisco Secure Workload — Secure Firewall Integration Guide

![Visitors](https://visitor-badge.laobi.icu/badge?page_id=chandrapati.CSW-Secure-Firewall-Integration-Guide&left_text=visitors)

Step-by-step reference for integrating **Cisco Secure Workload (CSW)** with **Cisco Secure Firewall** (FTD/ASA + FMC): **NetFlow / NSEL visibility** and **FMC policy enforcement**.

Written for security engineers, network/firewall teams, and POV delivery staff who need agentless east-west segmentation or network enforcement alongside host agents.

> **Disclaimer:** Companion learning material — not official Cisco product documentation. Validate design, supported versions, and limits against your tenant and the [CSW 4.0 connector documentation](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-saas-v40/m-connectors.html) before production.

**Executive summary (1 page, CISO / network leadership):** [`docs/EXECUTIVE-SUMMARY.md`](docs/EXECUTIVE-SUMMARY.md) · [**PDF**](docs/CSW-Secure-Firewall-Executive-Summary.pdf) · [**Word**](docs/CSW-Secure-Firewall-Executive-Summary.docx)

**Full guide:** [`docs/INTEGRATION-GUIDE.md`](docs/INTEGRATION-GUIDE.md) · **Scope / query / ACP mapping:** [`docs/SCOPE-ACP-QUERY-MAPPING.md`](docs/SCOPE-ACP-QUERY-MAPPING.md) · **Architecture:** [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) · **Word/PDF:** [`docs/`](docs/)

> Sources: [FMC Integration Guide](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/integration/fmc/cisco-secure-workload-and-fmc-integration-guide.html) · [Secure Workload & Firewall deep dive](https://secure.cisco.com/secure-workload/docs/secure-workload-whitepaper)

---

## Executive summary

CSW + Secure Firewall unifies **agentless east-west microsegmentation** for workloads that cannot run host agents (legacy OS, appliances, restricted environments). Two connectors work together:

| Path | Connector | Outcome |
|------|-----------|---------|
| **Visibility** | Secure Firewall Connector (NSEL on Ingest) | Discover flows, run ADM, model policy — no agents |
| **Enforcement** | FMC Connector | Push L3/L4 rules + Dynamic Objects to FTD via FMC |

Extended use cases from the white paper: **Virtual Patch** (CVE → FMC IPS) and **Rapid Threat Containment** (FMC Remediation Module → CSW quarantine guardrails).

---

## Architecture

**All diagrams are in one place:** [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — high-level integration, NSEL/FMC sequence flows, scope↔ACP mapping, insertion options (L2/L3/ACI/cloud), virtual patch, and rapid threat containment.

| Path | Connector | Data direction |
|------|-----------|----------------|
| **Visibility (NSEL)** | Secure Firewall Connector on Ingest | Firewall → Ingest → CSW |
| **Enforcement** | FMC Connector (Cisco Secure Firewall in UI) | CSW → FMC → FTD |

**SaaS:** Secure Connector required between CSW cloud and on-prem Ingest/FMC. **NetFlow connector** (switches/routers) and **ACI connector** are separate paths — not FTD NSEL.

---

## Jorge Quintero — 3-part video series (start here)

Cisco TME **[Jorge Quintero](https://www.youtube.com/@ciscosecureworkload)** published the definitive walkthrough of CSW + Secure Firewall integration on the official Cisco Secure Workload channel. Watch in order before the step-by-step sections below.

| Part | Focus | Watch |
|------|--------|-------|
| **1** | Introduction, design & architecture | [🎬 YouTube](https://youtu.be/vdHjAl48SuI) |
| **2** | Deployment patterns & policy flow | [🎬 YouTube](https://www.youtube.com/watch?v=xpbg3s0vrcI) |
| **3** | Enforcement, telemetry & operations | *(original link unavailable — search [🎬 Cisco Secure Workload channel](https://www.youtube.com/@ciscosecureworkload))* |

**Also recommended:** [Secure Workload & Secure Firewall Integration Updates (2025–2026)](https://youtu.be/IEqbz44YvOQ) — current product behavior on the same channel.

**Scope → ACP mapping detail:** [`docs/SCOPE-ACP-QUERY-MAPPING.md`](docs/SCOPE-ACP-QUERY-MAPPING.md)

---

**Jason Maynard** ([@jasonmaynard8773](https://www.youtube.com/@jasonmaynard8773)) covers scopes, labels, inventory filters, and policy workflow. Required before scope→ACP mapping.

| Video | Watch |
|-------|-------|
| Scopes | [🎬 YouTube](https://www.youtube.com/watch?v=3KBmanCNm4U) |
| Labels | [🎬 YouTube](https://www.youtube.com/watch?v=NLoZq0wiTU8) |
| Inventory Filters | [🎬 YouTube](https://www.youtube.com/watch?v=fJd6V15UiZM) |
| ADM & Policy Analysis | [🎬 YouTube](https://www.youtube.com/watch?v=Jzzblea25UA) |
| Dynamic Workloads & Policy | [🎬 YouTube](https://www.youtube.com/watch?v=Aajlx7JT2G4) |

Full steps: [`docs/SCOPE-ACP-QUERY-MAPPING.md`](docs/SCOPE-ACP-QUERY-MAPPING.md)

---

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

**Limits (CSW 4.0):** max **100** Secure Firewall connectors per Ingest appliance; **100** per tenant; **100** across the cluster.

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

**Videos (Jorge Quintero):** [Part 1 — design](https://youtu.be/vdHjAl48SuI) · [Part 2 — deployment](https://www.youtube.com/watch?v=xpbg3s0vrcI) · [2025–2026 updates](https://youtu.be/IEqbz44YvOQ)

### Step A3b — FTD NSEL via FlexConfig (FMC UI)

FTD does **not** expose NSEL configuration in the standard FMC UI — you must use **FlexConfig**. The commands are identical to ASA but are pushed through FMC's template engine.

1. **Objects → Object Management → FlexConfig → FlexConfig Objects** → **Add FlexConfig Object** (type: **Prepended**).
2. Paste the NSEL config below, substituting the literal `<INTERFACE>` (e.g., `inside`) and `<INGEST_CONNECTOR_IP>`:

```text
flow-export destination <INTERFACE> <INGEST_CONNECTOR_IP> 4729
flow-export template timeout-rate 1
policy-map global_policy
  class class-default
    flow-export event-type flow-create destination <INGEST_CONNECTOR_IP>
    flow-export event-type flow-teardown destination <INGEST_CONNECTOR_IP>
    flow-export event-type flow-denied destination <INGEST_CONNECTOR_IP>
    flow-export event-type flow-update destination <INGEST_CONNECTOR_IP>
    user-statistics accounting
```

> **Do not** add `service-policy global_policy global` — it is already active by default on FTD and will cause a conflict if re-added.

3. **Devices → FlexConfig** → create or open a FlexConfig Policy → add the object → **Assign** to target FTD(s) → **Save**.
4. **Deploy → Deployment** → select FTDs → **Deploy**.
5. Verify: FTD CLI (`show flow-export counters`) and CSW **Flow Analysis** (flows appear within ~2 min).

> **FTD 7.4+:** Check FMC **Platform Settings** for a native NetFlow tab — if present, use it instead of FlexConfig. FlexConfig remains supported as a fallback.

**Full step-by-step with screenshots:** [`docs/INTEGRATION-GUIDE.md — Step A3b`](docs/INTEGRATION-GUIDE.md) · **Video (covers FlexConfig):** [Part 2](https://www.youtube.com/watch?v=xpbg3s0vrcI)

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

1. **Manage → Workloads → Connectors → Firewall → Cisco Secure Firewall**.
2. Enter FMC **hostname**, **REST API credentials** (admin), **CA certificate** (or Disable SSL), **Secure Connector** if SaaS.
3. Verify **Event Log** tab after **Create**.

| FMC field | Notes |
|-----------|-------|
| Server IP/FQDN | FMC primary; include **standby FMC IP** for HA |
| Port | **443** (HTTPS REST API) |
| Proxy | `<host>:<port>` if required |

4. Full procedure: [FMC Integration Guide](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/integration/fmc/cisco-secure-workload-and-fmc-integration-guide.html).

### Step B3 — Map scope to Access Control Policy

1. **Segmentation** tab → **+ Add** → select **ACP** and **one Scope** (1:1 only).
2. **Use Secure Workload Catch-All** — CSW catch-all or FMC default action.
3. **Enforcement Mode:** **Merge** (dual-management) or **Override**.
4. Set **Absolute/Default** rule priority (above/below Mandatory/Default sections).
5. Enable **Topology Awareness** — rules only on firewalls in traffic path.

**CSW creates in FMC:** `Workload_golden__`, `Workload__`, `Workload_ca__` rules; `WorkloadObj__` dynamic objects. Do not edit these in FMC.

| Mode | Behavior |
|------|----------|
| **Merge** | CSW rules coexist with FMC rules; configure top/bottom priority |
| **Override** | CSW replaces user-created rules in mapped sections |

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

| Order | Video | Presenter / channel | Link |
|------:|-------|---------------------|------|
| 1 | **Secure Workload & Firewall Integration — Part 1** (design & architecture) | Jorge Quintero · [🎬 @ciscosecureworkload](https://www.youtube.com/@ciscosecureworkload) | [🎬 Watch](https://youtu.be/vdHjAl48SuI) |
| 2 | **Part 2** (deployment patterns & policy flow) | Jorge Quintero | [🎬 Watch](https://www.youtube.com/watch?v=xpbg3s0vrcI) |
| 3 | **Part 3** (enforcement, telemetry & operations) | Jorge Quintero | *(original link unavailable — search [🎬 channel](https://www.youtube.com/@ciscosecureworkload))* |
| 4 | Secure Workload & Secure Firewall Integration Updates (2025–2026) | Cisco Secure Workload channel | [🎬 Watch](https://youtu.be/IEqbz44YvOQ) |
| 5 | Connector Overview | Cisco Secure Workload channel | [🎬 Watch](https://youtu.be/H6QxuouzeC8) |
| 6 | Connector Deployment | Cisco Secure Workload channel | [🎬 Watch](https://youtu.be/H0as2ppS84Q) |
| 7 | FMC + Edge / Ingest / Appliance | Cisco Secure Workload channel | [🎬 Watch](https://youtu.be/13AZ33dpCxU) |
| 8 | Where to Enforce | Cisco Secure Workload channel | [🎬 Watch](https://youtu.be/urFJyDERMFs) |
| 9 | Policy Enforcement Overview | Cisco Secure Workload channel | [🎬 Watch](https://youtu.be/A8rOXQ-y4Cw) |

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

## Firewall insertion options (summary)

| Mode | Segmentation | NSEL | Best for |
|------|--------------|------|----------|
| **L2 Transparent** | Intra + inter-subnet | Full | Fine-grained, legacy OS |
| **L3 Routed** | Inter-subnet only | Inter-subnet only | Quick zone segmentation |
| **ACI Service Graph** | Intra + inter EPG/ESG | Full (FW in path) | ACI fabric estates |
| **Cloud hub (AWS/Azure/GCP)** | Inter-VPC/VNet | Cloud logs + NSEL | Centralized cloud east-west |

Details: [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) only · Full guide: [`docs/INTEGRATION-GUIDE.md`](docs/INTEGRATION-GUIDE.md)

---

## Official Cisco documentation

| Document | URL |
|----------|-----|
| **FMC Integration Guide** | [📄 cisco-secure-workload-and-fmc-integration-guide.html](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/integration/fmc/cisco-secure-workload-and-fmc-integration-guide.html) |
| **Deep dive white paper** | [📄 secure-workload-whitepaper](https://secure.cisco.com/secure-workload/docs/secure-workload-whitepaper) |
| CSW 4.0 SaaS — Connectors | [📄 m-connectors.html](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-saas-v40/m-connectors.html) |
| Marketing white paper | [📄 sec-workload-firewall-wp.html](https://www.cisco.com/c/en/us/products/collateral/security/secure-workload/sec-workload-firewall-wp.html) |
| ASA NSEL configuration | [📄 ASA NetFlow Implementation Guide](https://www.cisco.com/c/en/us/td/docs/security/asa/asa-netflow/asa-netflow.html) |

---

## Cisco Secure Workload — companion repositories

This repo is the **Secure Firewall integration** chapter of the public CSW practitioner toolkit. Use the suggested path for a new customer engagement.

| Repository | What it covers | When to use |
|------------|----------------|-------------|
| [📘 **CSW-User-Education**](https://github.com/chandrapati/CSW-User-Education) | Intro, 62-video library, onboarding runbook, POV evidence checklist | First stop for anyone new to CSW |
| [📘 **CSW-Secure-Firewall-Integration-Guide**](https://github.com/chandrapati/CSW-Secure-Firewall-Integration-Guide) | **This repo** — NSEL ingest + FMC enforcement step-by-step | Firewall + NetFlow in scope |
| [📘 **CSW-Agent-Installation-Guide**](https://github.com/chandrapati/CSW-Agent-Installation-Guide) | Host agent install: Linux, Windows, cloud, containers, agentless | Deploying agents (complements firewall path) |
| [📘 **CSW-Policy-Lifecycle**](https://github.com/chandrapati/CSW-Policy-Lifecycle) | ADM → Monitor → Simulate → Enforce + day-2 ops | After visibility; before enforce |
| [📘 **CSW-Identity-Integration-Guide**](https://github.com/chandrapati/CSW-Identity-Integration-Guide) | AD, Entra ID, DC user-identity reporting | User/group labels in policy |
| [📘 **CSW-ServiceNow-Connector-Guide**](https://github.com/chandrapati/CSW-ServiceNow-Connector-Guide) | ServiceNow inventory enrichment connector | CMDB-driven labels |
| [📘 **csw-ise-integration**](https://github.com/chandrapati/csw-ise-integration) | ISE/pxGrid integration: user-identity–aware microsegmentation via pxGrid 2.0 | Identity & Zero Trust |
| [📘 **csw-splunk-integration**](https://github.com/chandrapati/csw-splunk-integration) | CSW → Splunk via Syslog + Security Cloud App | SOC / SIEM integration |
| [📘 **CSW-Compliance-Mapping**](https://github.com/chandrapati/CSW-Compliance-Mapping) | 30+ framework mappings (HIPAA, PCI, NIST, ISO, etc.) | GRC / audit / CISO reporting |
| [📘 **CSW-Operations-Toolkit**](https://github.com/chandrapati/CSW-Operations-Toolkit) | Reusable POV engagement toolkit | Starting a new POV |
| [📘 **csw-logs-check**](https://github.com/chandrapati/csw-logs-check) | Cursor skill: analyze agent diagnostic bundles | Host enforce timing / validation |
| [📘 **csw_blast_radius_demo**](https://github.com/chandrapati/csw_blast_radius_demo) | Hands-on blast-radius reduction demo | Customer demos and labs |
| [📘 **CSW-SE-Helper-Repo**](https://github.com/chandrapati/CSW-SE-Helper-Repo) | SE utilities and scratch tooling | Internal SE workflows |

### Suggested learning path

```text
CSW-User-Education
    → CSW-Agent-Installation-Guide (if agents in scope)
    → CSW-Secure-Firewall-Integration-Guide (if firewall in scope)  ← you are here
    → CSW-Identity-Integration-Guide (if user identity in scope)
    → CSW-Policy-Lifecycle
    → csw-splunk-integration
    → CSW-Compliance-Mapping
    → CSW-Operations-Toolkit (wrap-up)
```

For **host enforcement timing** validation after agents are deployed, use [**csw-logs-check**](https://github.com/chandrapati/csw-logs-check) alongside this guide when both agents and firewalls are in the same POV.

---

## Repository layout

| Path | Description |
|------|-------------|
| `README.md` | Overview, step-by-step integration, video links, companion repos |
| `docs/EXECUTIVE-SUMMARY.md` | **One-page executive summary** for CISO / network leadership |
| `docs/CSW-Secure-Firewall-Executive-Summary.pdf` | PDF export of executive summary |
| `docs/INTEGRATION-GUIDE.md` | Full customer reference (steps, FAQs, virtual patch, insertion options) |
| `docs/SCOPE-ACP-QUERY-MAPPING.md` | Scope queries, inventory filters, ACP mapping, Jason Maynard videos |
| `docs/ARCHITECTURE.md` | Mermaid architecture diagrams for customer presentations |
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

---

## Step-by-Step Guides

Hands-on integration and deployment guides — follow these top to bottom to build out a deployment:

| Guide | Description | Best for |
|-------|-------------|---------|
| [📘 Agent Installation](https://github.com/chandrapati/CSW-Agent-Installation-Guide) | Deploy CSW agents on Linux / Windows / cloud | Day-1 sensor deployment |
| [📘 Policy Lifecycle](https://github.com/chandrapati/CSW-Policy-Lifecycle) | Policy discovery → enforcement workflow | Policy management |
| [📘 ISE / pxGrid](https://github.com/chandrapati/csw-ise-integration) | ISE/pxGrid: user-identity–aware microsegmentation | Identity & Zero Trust |
| [📘 AnyConnect NVM](https://github.com/chandrapati/csw-anyconnect-nvm) | Endpoint process flows + user identity via NVM | Endpoint telemetry |
| [📘 ServiceNow CMDB](https://github.com/chandrapati/csw-servicenow-integration) | ServiceNow CMDB label enrichment for workload scopes | CMDB-driven policy |
| [📘 Infoblox](https://github.com/chandrapati/csw-infoblox-integration) | Infoblox IPAM/DNS extensible-attribute label enrichment | IPAM/DNS-driven policy |
| [📘 F5 BIG-IP](https://github.com/chandrapati/csw-f5-integration) | F5 virtual-server labels, policy enforcement, IPFIX flow visibility | Load balancer segmentation |
| [📘 NetScaler ADC](https://github.com/chandrapati/csw-netscaler-integration) | NetScaler LB virtual-server labels + ACL policy enforcement | Load balancer segmentation |
| [📘 AWS Connector](https://github.com/chandrapati/csw-aws-connector) | EC2 tag ingestion + VPC flow logs + Security Group enforcement | AWS workloads |
| [📘 Azure Connector](https://github.com/chandrapati/csw-azure-connector) | Azure VM tag ingestion + VNet flow logs + NSG enforcement | Azure workloads |
| [📘 GCP Connector](https://github.com/chandrapati/csw-gcp-connector) | GCE label ingestion + VPC flow logs + firewall enforcement | GCP workloads |
| [📘 NetFlow](https://github.com/chandrapati/csw-netflow-integration) | NetFlow v9/IPFIX agentless flow ingestion from switches | Network fabric visibility |
| [📘 ERSPAN](https://github.com/chandrapati/csw-erspan-integration) | Agentless packet mirroring for legacy / OT / IoT devices | Deep agentless visibility |
| [📘 Secure Firewall](https://github.com/chandrapati/CSW-Secure-Firewall-Integration-Guide) | NSEL flow ingestion from Cisco Secure Firewall (FTD/ASA) | Firewall flow visibility |
| [📘 Splunk Integration](https://github.com/chandrapati/csw-splunk-integration) | CSW syslog alerts → Splunk SIEM | SecOps / SIEM teams |

## Resources

Learning paths, reference material, and day-2 tooling:

| Resource | Description | Best for |
|----------|-------------|---------|
| [📘 User Education](https://github.com/chandrapati/CSW-User-Education) | Onboarding guides, concept explainers, and curated video library | New CSW users |
| [📘 Compliance Mapping](https://github.com/chandrapati/CSW-Compliance-Mapping) | Map CSW controls to NIST, PCI-DSS, HIPAA, CIS | Compliance & audit |
| [📘 Tenant Insights](https://github.com/chandrapati/CSW-Tenant-Insights) | Tenant-level reporting and analytics | Visibility metrics |
| [📘 Operations Toolkit](https://github.com/chandrapati/CSW-Operations-Toolkit) | Day-2 ops scripts: health checks, reporting, policy analysis | Ongoing operations |

> **Suggested customer journey:**
> User Education → Agent Installation → Policy Lifecycle → ISE/pxGrid → ServiceNow CMDB → Infoblox → F5 BIG-IP → NetScaler ADC → Splunk Integration → Compliance Mapping → Operations Toolkit
