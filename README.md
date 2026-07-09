# Cisco Secure Workload — ISE / pxGrid Integration Guide

![Visitors](https://visitor-badge.laobi.icu/badge?page_id=chandrapati.csw-ise-integration&left_text=visitors)

A step-by-step community reference guide for integrating **Cisco Secure Workload (CSW)** with **Cisco Identity Services Engine (ISE)** via **pxGrid 2.0** to enable user-identity–aware microsegmentation.

> **Disclaimer:** This is a reference guide prepared by Cisco Solutions Engineering. Always consult [official Cisco Secure Workload documentation](https://www.cisco.com/c/en/us/products/security/tetration/index.html) and [Cisco ISE documentation](https://www.cisco.com/c/en/us/support/security/identity-services-engine/series.html) for authoritative, release-specific guidance.

---

## What This Integration Provides

| Without ISE | With ISE |
|-------------|---------|
| Policies based on IP addresses | Policies based on **user**, **AD group**, **SGT**, **posture** |
| Static scope membership | **Dynamic scopes** — follow the user regardless of IP |
| No VPN/remote user context | Full identity context for roaming and VPN endpoints |

---

## Contents

| File | Description |
|------|-------------|
| [`CSW-ISE-Integration-Guide.md`](./CSW-ISE-Integration-Guide.md) | Full integration guide (Markdown source) |
| [`CSW-ISE-Integration-Guide.html`](./CSW-ISE-Integration-Guide.html) | Standalone HTML version (embeds all styles) |
| [`build.sh`](./build.sh) | Script to rebuild HTML from Markdown |

---

## Quick Reference

### Key requirements
- ISE 2.4+ or ISE-PIC 3.1+
- CSW 3.7+ (pxGrid SAN cert required)
- pxGrid 2.0 (WebSocket/REST — pxGrid 1.0 / XMPP is end-of-life)
- **CSW Edge appliance** (not Ingest — ISE connector runs on Edge)
- TCP 8910 open: Edge appliance → ISE pxGrid node

### What CSW enriches from ISE

```
IP: 10.20.30.41
  user/username         = jsmith
  user/ad_group         = Finance-Analysts
  user/sgt              = Finance
  device/type           = Windows-10-Workstation
  device/mdm_compliance = Compliant
```

### Limits
| Metric | Limit |
|--------|-------|
| ISE instances per connector | 20 |
| ISE connectors per Edge appliance | 1 |
| ISE connectors per tenant | 1 |
| Max ISE endpoints per connector | 400,000 |

---

## Guide Sections

1. Overview & what you get
2. Architecture diagram
3. ISE attributes ingested by CSW
4. Use cases (identity-aware policy, posture enforcement, SGT-based policy, lateral movement detection)
5. Prerequisites checklist
6. **Step A** — Enable pxGrid on ISE, verify SAN cert
7. **Step B** — Generate pxGrid client certificate (OpenSSL CSR → CA sign → PEM)
8. **Step C** — Configure ISE connector on CSW Edge appliance
9. Optional LDAP/AD user attribute enrichment
10. Scope and policy examples
11. Verification steps
12. Periodic tasks and data refresh timing
13. Limits
14. Troubleshooting

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
