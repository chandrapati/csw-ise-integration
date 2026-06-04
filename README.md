# Cisco Secure Workload — ISE / pxGrid Integration Guide

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

## Related Cisco Secure Workload Resources

| Repository | Description | Best for |
|------------|-------------|---------|
| [CSW-User-Education](https://github.com/chandrapati/CSW-User-Education) | Onboarding guides and concept explainers | New CSW users |
| [CSW-Agent-Installation-Guide](https://github.com/chandrapati/CSW-Agent-Installation-Guide) | Deploy CSW agents on Linux/Windows/cloud | Day-1 sensor deployment |
| [CSW-Policy-Lifecycle](https://github.com/chandrapati/CSW-Policy-Lifecycle) | Policy discovery → enforcement workflow | Policy management |
| [csw-ise-integration](https://github.com/chandrapati/csw-ise-integration) | ISE/pxGrid: user-identity–aware microsegmentation | Identity & Zero Trust |
| [csw-anyconnect-nvm](https://github.com/chandrapati/csw-anyconnect-nvm) | AnyConnect NVM: endpoint process flows + user identity | Endpoint telemetry |
| [csw-servicenow-integration](https://github.com/chandrapati/csw-servicenow-integration) | ServiceNow CMDB label enrichment for workload scopes | CMDB-driven policy |
| [csw-aws-connector](https://github.com/chandrapati/csw-aws-connector) | AWS VPC label ingestion + Security Group enforcement | AWS workloads |
| [csw-azure-connector](https://github.com/chandrapati/csw-azure-connector) | Azure VNet label ingestion + NSG enforcement | Azure workloads |
| [csw-gcp-connector](https://github.com/chandrapati/csw-gcp-connector) | GCP VPC label ingestion + firewall enforcement | GCP workloads |
| [csw-netflow-integration](https://github.com/chandrapati/csw-netflow-integration) | NetFlow v9/IPFIX agentless flow ingestion from switches | Network fabric visibility |
| [csw-erspan-integration](https://github.com/chandrapati/csw-erspan-integration) | ERSPAN agentless packet mirroring for legacy/OT/IoT | Agentless deep visibility |
| [CSW-Secure-Firewall-Integration-Guide](https://github.com/chandrapati/CSW-Secure-Firewall-Integration-Guide) | NSEL from Cisco Secure Firewall (FTD/ASA) | Firewall flow visibility |
| [csw-splunk-integration](https://github.com/chandrapati/csw-splunk-integration) | CSW syslog alerts → Splunk SIEM | SecOps / SIEM teams |
| [CSW-Compliance-Mapping](https://github.com/chandrapati/CSW-Compliance-Mapping) | Map CSW to NIST, PCI-DSS, HIPAA, CIS | Compliance & audit |
| [CSW-Tenant-Insights](https://github.com/chandrapati/CSW-Tenant-Insights) | Tenant-level reporting and analytics | Visibility metrics |
| [CSW-Operations-Toolkit](https://github.com/chandrapati/CSW-Operations-Toolkit) | Day-2 ops scripts: health checks, reporting, policy analysis | Ongoing operations |

> **Suggested customer journey:**  
> CSW-User-Education → CSW-Agent-Installation-Guide → CSW-Policy-Lifecycle → csw-ise-integration → csw-servicenow-integration → csw-splunk-integration → CSW-Compliance-Mapping → CSW-Operations-Toolkit
