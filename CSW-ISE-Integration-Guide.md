---
title: "Cisco Secure Workload — ISE / pxGrid Integration Guide"
version: "1.0"
audience: "Enterprise security architects, Zero Trust practitioners, SecOps engineers"
prepared_by: "Cisco Solutions Engineering"
---

# Cisco Secure Workload — ISE / pxGrid Integration Guide

> **Disclaimer:** This is a community reference guide prepared by Cisco Solutions Engineering.  
> It is provided as-is for educational and planning purposes.  
> Always refer to [official Cisco Secure Workload documentation](https://www.cisco.com/c/en/us/products/security/tetration/index.html)  
> and [Cisco ISE documentation](https://www.cisco.com/c/en/us/support/security/identity-services-engine/series.html) for authoritative, release-specific guidance.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [What CSW Gets from ISE](#3-what-csw-gets-from-ise)
4. [Use Cases](#4-use-cases)
5. [Prerequisites](#5-prerequisites)
6. [Step A — ISE pxGrid Configuration](#6-step-a--ise-pxgrid-configuration)
7. [Step B — Generate the pxGrid Client Certificate](#7-step-b--generate-the-pxgrid-client-certificate)
8. [Step C — Configure the ISE Connector on CSW](#8-step-c--configure-the-ise-connector-on-csw)
9. [Optional: LDAP Integration for User Labels](#9-optional-ldap-integration-for-user-labels)
10. [How CSW Uses the ISE Data](#10-how-csw-uses-the-ise-data)
11. [Policy Examples](#11-policy-examples)
12. [Verification](#12-verification)
13. [Periodic Tasks and Data Refresh](#13-periodic-tasks-and-data-refresh)
14. [Limits](#14-limits)
15. [Troubleshooting](#15-troubleshooting)
16. [Related Resources](#16-related-resources)

---

## 1. Overview

The **ISE connector** in Cisco Secure Workload (CSW) integrates with **Cisco Identity Services Engine (ISE)** and **ISE Passive Identity Connector (ISE-PIC)** using **Cisco Platform Exchange Grid (pxGrid 2.0)** to retrieve contextual information about endpoints.

This integration brings **user identity, device posture, and Security Group Tag (SGT)** context into CSW workload policy — enabling true **user-aware microsegmentation** for Zero Trust architectures.

### What this integration enables

| Before ISE Integration | After ISE Integration |
|------------------------|----------------------|
| Policies based on IP addresses only | Policies based on **user identity**, **AD group**, **device type**, **posture** |
| Static scope membership | **Dynamic scope membership** driven by real-time ISE endpoint data |
| Network-layer segmentation | **Identity-aware segmentation** — allow/deny based on who the user is |
| No visibility into VPN/remote users | **Remote endpoint context** from ISE for VPN-connected workstations |

### ISE versions supported

| Component | Minimum Version |
|-----------|----------------|
| Cisco ISE | 2.4 |
| ISE Passive Identity Connector (ISE-PIC) | 3.1 |
| Cisco Secure Workload | 3.7 (SAN cert required from 3.7+) |
| pxGrid | **2.0 only** (pxGrid 1.0 / XMPP is end-of-life) |

---

## 2. Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Enterprise Data Center / Cloud                │
│                                                                  │
│  ┌──────────────┐    pxGrid 2.0     ┌─────────────────────────┐ │
│  │  Cisco ISE   │◄──────────────────│  CSW Edge Appliance     │ │
│  │              │   (WebSocket/     │  (ISE Connector)        │ │
│  │  • Sessions  │    REST/TLS)      │                         │ │
│  │  • SGTs      │                   │  • Registers endpoints  │ │
│  │  • Posture   │                   │  • Pulls SGT labels     │ │
│  │  • AD Groups │                   │  • Streams session events│ │
│  └──────┬───────┘                   └──────────┬──────────────┘ │
│         │                                      │                 │
│  ┌──────▼─────────────────────────────────────▼──────────────┐  │
│  │              Cisco Secure Workload (SaaS / On-Prem)        │  │
│  │                                                            │  │
│  │  Inventory Enrichment:                                     │  │
│  │    workload_ip → user, AD group, SGT, device_type, posture │  │
│  │                                                            │  │
│  │  Scope / Policy Engine:                                    │  │
│  │    Scope: "Finance Users" = ISE attribute SGT == Finance   │  │
│  │    Policy: Finance Users → App servers: allow TCP 8443     │  │
│  └────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Component roles

| Component | Role |
|-----------|------|
| **ISE** | Source of user sessions, endpoint metadata, SGT assignments, posture results |
| **pxGrid 2.0** | Secure bi-directional pub/sub channel between ISE and third-party systems |
| **CSW Edge Appliance** | Hosts the ISE connector process; manages pxGrid subscription |
| **CSW Ingest Appliance** | (Not used for ISE; used for flow connectors) |
| **CSW Cluster** | Stores enriched inventory labels; evaluates policy against labeled workloads |

---

## 3. What CSW Gets from ISE

The ISE connector subscribes to two pxGrid topics:

### 3.1 Session / Endpoint records

Each authenticated endpoint record provides:

| ISE Attribute | CSW Label Key | Example Value |
|---------------|---------------|---------------|
| IP Address | (primary key — maps to workload) | `10.20.30.41` |
| Username | `user/username` | `jsmith` |
| AD Domain | `user/domain` | `corp.example.com` |
| AD Group | `user/ad_group` | `Finance-Analysts` |
| Device Type | `device/type` | `Windows-10-Workstation` |
| Endpoint Profile | `device/profile` | `Microsoft-Workstation` |
| MDM Compliance | `device/mdm_compliance` | `Compliant` |
| Security Group Tag (SGT) | `user/sgt` | `Finance` |
| VLAN | `network/vlan` | `100` |
| NAS-IP | `network/nas_ip` | `10.0.0.1` |
| Auth Method | `user/auth_method` | `802.1X` |

### 3.2 Security Group (SGT) records

The connector also subscribes to SGT updates, maintaining a local mapping of SGT numeric value → human-readable name. This is used to translate the raw SGT integer in endpoint records to a friendly label (e.g., `SGT 10 → "Finance"`).

---

## 4. Use Cases

### Use Case 1 — Identity-Aware Microsegmentation

**Scenario:** Allow only users in the **Finance-Analysts** AD group to reach financial database servers on TCP 1521.

**Without ISE:** Requires a static list of source IP addresses — breaks when users move or use VPN.

**With ISE:**
1. ISE labels every Finance-Analyst endpoint with `user/ad_group = Finance-Analysts`
2. CSW scope `Finance-Analysts-Endpoints` = `{ user/ad_group == 'Finance-Analysts' }`
3. CSW policy: `Finance-Analysts-Endpoints → FinDB-Cluster: ALLOW TCP 1521`
4. Policy is **dynamic** — follows the user regardless of IP

---

### Use Case 2 — Posture-Based Access

**Scenario:** Block non-compliant endpoints from accessing production application servers.

**With ISE:**
1. ISE marks non-compliant devices: `device/mdm_compliance = Non-Compliant`
2. CSW scope `Non-Compliant-Endpoints` = `{ device/mdm_compliance == 'Non-Compliant' }`
3. CSW policy: `Non-Compliant-Endpoints → ProdApp-Servers: DENY ALL`

---

### Use Case 3 — SGT-Based Policy (TrustSec)

**Scenario:** Leverage existing TrustSec SGT assignments in CSW policy.

**With ISE:**
1. ISE provides SGT assignment for every authenticated endpoint
2. CSW labels endpoints with `user/sgt = Finance`
3. CSW policy references SGT labels directly — no IP re-engineering required

---

### Use Case 4 — Lateral Movement Detection by User

**Scenario:** Alert when a Finance user is connecting to servers outside the Finance scope.

**With ISE:**
1. CSW scope `Finance-Endpoints` = `{ user/sgt == 'Finance' }`
2. CSW Traffic Alert Rule: `Finance-Endpoints → non-Finance-Servers: ALERT (MEDIUM)`
3. Forwarded via Syslog to Splunk — full user identity context in alert payload

---

## 5. Prerequisites

### ISE side

- [ ] ISE 2.4+ (or ISE-PIC 3.1+) deployed and reachable from CSW Edge appliance network
- [ ] pxGrid service **enabled** on at least one ISE node
- [ ] ISE pxGrid node SSL certificate includes **Subject Alternative Names (SAN)** (required CSW 3.7+)
- [ ] Certificate Authority (CA) accessible to sign the pxGrid client certificate
- [ ] Network connectivity: CSW Edge appliance → ISE pxGrid node on **TCP 8910** (pxGrid 2.0)

### CSW side

- [ ] **CSW Edge appliance** deployed (ISE connector runs on Edge; **not** Ingest)
- [ ] Edge appliance registered and visible in CSW UI under **Manage > Virtual Appliances**
- [ ] Sufficient Edge capacity (1 ISE connector per Edge appliance, 1 per tenant)
- [ ] VRF configuration for the Edge appliance matches the subnet of ISE-managed endpoints

### Certificates required

| Certificate | Purpose | Who generates |
|-------------|---------|---------------|
| ISE pxGrid Server CA cert | Trust anchor for ISE server identity | ISE admin — export from ISE |
| pxGrid Client cert + key | CSW connector authenticates to ISE | You generate (OpenSSL CSR → CA signs) |

---

## 6. Step A — ISE pxGrid Configuration

### A1 — Enable pxGrid service on ISE

1. In ISE, navigate to **Administration > System > Deployment**
2. Select the ISE node that will serve pxGrid
3. Under **General Settings**, check the box for **pxGrid**
4. Click **Save**

> If you have multiple ISE nodes, enable pxGrid on the nodes that will handle the integration. For HA, enable on both primary and secondary pxGrid nodes.

### A2 — Verify pxGrid node certificate has SANs (CSW 3.7+ requirement)

1. Navigate to **Administration > System > Certificates > Certificate Management > System Certificates**
2. Find the certificate with **Used by: pxGrid**
3. Click **View**
4. Scroll to the **Extensions** section and confirm **Subject Alternative Names (SAN)** are present
5. The SAN should include the FQDN and/or IP of the pxGrid node

> **Important:** If SAN is missing, have your ISE admin regenerate the pxGrid certificate with proper SAN values before proceeding. CSW 3.7+ will reject certificates without SAN.

### A3 — Configure pxGrid client registration mode

1. Navigate to **Administration > pxGrid Services > Settings**
2. Choose one of:
   - **Automatically approve new accounts** (lab/POV — simpler)
   - **Manually approve new accounts** (production — more secure)
3. Click **Save**

> If using manual approval, you will need to approve the CSW connector's pxGrid account after the connector is configured in Step C.

---

## 7. Step B — Generate the pxGrid Client Certificate

The CSW ISE connector authenticates to ISE using a **client certificate**. This certificate must be signed by the same CA that signed the ISE pxGrid server certificate.

### B1 — Create the OpenSSL CSR configuration

On any Linux/macOS host with OpenSSL installed, create a file named `csw-ise-connector.cfg`:

```ini
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
x509_extensions = v3_req
prompt = no

[req_distinguished_name]
C  = US
ST = California
L  = San Jose
O  = YourOrganization
OU = NetworkSecurity
CN = csw-ise-connector.corp.example.com

[v3_req]
subjectKeyIdentifier = hash
basicConstraints     = critical,CA:false
subjectAltName       = @alt_names
keyUsage             = critical,digitalSignature,keyEncipherment
extendedKeyUsage     = serverAuth,clientAuth

[alt_names]
IP.1  = 10.x.x.x
DNS.1 = csw-ise-connector.corp.example.com
```

> Replace `10.x.x.x` with the IP of your CSW Edge appliance and update the DNS name and organization fields accordingly.

### B2 — Generate the CSR and private key

```bash
openssl req -newkey rsa:2048 \
  -keyout csw-ise-connector.key \
  -nodes \
  -out csw-ise-connector.csr \
  -config csw-ise-connector.cfg
```

This produces:
- `csw-ise-connector.csr` — submit this to your CA for signing
- `csw-ise-connector.key` — **private key — protect this file**

### B3 — Sign the CSR with your CA

**Option A — Windows CA (recommended for enterprise environments):**

```batch
certreq -submit -binary -attrib "CertificateTemplate:CiscoIdentityServicesEngine" ^
  csw-ise-connector.csr csw-ise-connector.cer
```

> The Windows CA Certificate Template must have the following extensions:
> - **Key Usage:** Digital Signature, Key Encipherment
> - **Extended Key Usage:** Server Authentication, Client Authentication

**Option B — OpenSSL self-managed CA:**

```bash
openssl x509 -req -days 730 \
  -in csw-ise-connector.csr \
  -CA root-ca.pem \
  -CAkey root-ca.key \
  -CAcreateserial \
  -extfile csw-ise-connector.cfg \
  -extensions v3_req \
  -out csw-ise-connector.pem
```

### B4 — Convert signed certificate to PEM format (if needed)

If your CA returned a `.cer` (DER binary) file:

```bash
openssl x509 -inform der -in csw-ise-connector.cer -out csw-ise-connector.pem
```

### B5 — Verify the certificate

```bash
openssl verify -CAfile root-ca.pem csw-ise-connector.pem
# Expected output: csw-ise-connector.pem: OK
```

### B6 — Export ISE root CA certificate

In ISE, navigate to **Administration > System > Certificates > Trusted Certificates**.  
Export the root CA certificate used to sign the ISE pxGrid node certificate in **PEM format**.  
Save as `ise-root-ca.pem`.

---

## 8. Step C — Configure the ISE Connector on CSW

### C1 — Navigate to connector configuration

1. In the CSW UI, navigate to **Manage > Virtual Appliances**
2. Select your **Edge appliance**
3. Click the **Connectors** tab
4. Click **+ Add Connector** and select **ISE**

### C2 — Basic connector settings

| Field | Value |
|-------|-------|
| **Connector Name** | e.g., `ise-connector-prod` |
| **Connector Type** | ISE |

### C3 — ISE Instance Configuration

Click **+ Add ISE Instance** and fill in:

| Field | Description | Example |
|-------|-------------|---------|
| **Name** | Friendly name for this ISE node | `ISE-Primary` |
| **ISE Hostname** | FQDN of the pxGrid node (**not IP — use FQDN if SAN has DNS entry**) | `ise01.corp.example.com` |
| **ISE Node Name** | pxGrid node name (typically matches hostname) | `ise01.corp.example.com` |
| **ISE Client Certificate** | Paste contents of `csw-ise-connector.pem` | (PEM text) |
| **ISE Client Key** | Paste contents of `csw-ise-connector.key` — **must be unencrypted (no passphrase)** | (PEM text) |
| **ISE Server CA Certificate** | Paste contents of `ise-root-ca.pem` | (PEM text) |
| **ISE IPv4 Subnet Filter** *(optional)* | Limit ingestion to specific subnets (CIDR) | `10.0.0.0/8` |
| **ISE IPv6 Subnet Filter** *(optional)* | IPv6 subnet filter | — |
| **Ignore ISE Attributes** *(optional)* | Attributes to exclude from CSW labels | (select if needed) |

> **Multi-node ISE HA:** Add a second ISE instance entry pointing to the secondary pxGrid node.  
> All pxGrid nodes must trust the same CA that signed the CSW client certificate.

> **Using IP instead of FQDN:** If you must use an IP address for ISE Hostname, the ISE CA certificate's SAN must include that IP address. Otherwise connection failures will occur.

### C4 — Apply and approve (if manual approval mode)

1. Click **Test and Apply** in the CSW connector UI
2. If ISE is configured for **manual approval**:
   - Log into ISE and navigate to **Administration > pxGrid Services > All Clients**
   - Find the CSW connector entry (will show as pending)
   - Click **Approve**
3. Return to CSW and confirm the connector shows **Connected** status

---

## 9. Optional: LDAP Integration for User Labels

The ISE connector can optionally connect to an LDAP/Active Directory server to enrich ISE endpoint records with additional user attributes (e.g., department, manager, title).

### LDAP Configuration fields

| Field | Description |
|-------|-------------|
| LDAP Server Hostname | FQDN of your AD / LDAP server |
| LDAP Port | 636 (LDAPS) or 389 (LDAP) — use 636 for production |
| LDAP Bind Username | Service account DN for read access |
| LDAP Bind Password | Service account password (encrypted in CSW) |
| Username Attribute | LDAP attribute that maps to ISE username (e.g., `sAMAccountName`) |
| Additional Attributes | Up to 6 additional attributes to fetch per user (e.g., `department`, `title`, `manager`) |
| LDAP Certificate | (Optional) CA cert for LDAPS |

### LDAP refresh

- CSW creates a local LDAP snapshot on connector start
- Snapshot refreshes every **24 hours** (configurable)
- User labels are updated in CSW every **2 minutes** based on the local LDAP snapshot + ISE session data

---

## 10. How CSW Uses the ISE Data

### Inventory enrichment

Once configured, ISE endpoint data flows into CSW as **user labels** on workload IP addresses:

```
IP: 10.20.30.41
  user/username     = jsmith
  user/ad_group     = Finance-Analysts
  user/sgt          = Finance
  device/type       = Windows-10-Workstation
  device/profile    = Microsoft-Workstation
  device/mdm_compliance = Compliant
  network/vlan      = 100
```

These labels are visible in **Inventory > Workloads** — search for any endpoint IP and see the enriched ISE attributes.

### VRF association

ISE endpoints are associated with a specific VRF based on the **Agent Remote VRF Configuration** on the CSW Edge appliance.  

To configure:  
**Manage > Workloads > Agents > Configuration tab > Agent Remote VRF Configurations > Create Config**

Provide:
- VRF name
- IP subnet range (CIDR) for endpoints registered by this ISE connector
- Port range for ISE endpoint registration

### Scope definition with ISE labels

Navigate to **Manage > Scopes** and create a scope using ISE-derived labels:

| Scope Name | Filter Expression |
|------------|------------------|
| `Finance-Endpoints` | `user/sgt = Finance` |
| `Compliant-Devices` | `device/mdm_compliance = Compliant` |
| `Finance-Analysts` | `user/ad_group = Finance-Analysts` |
| `Non-Compliant-Guests` | `device/profile = Guest AND device/mdm_compliance = Non-Compliant` |

---

## 11. Policy Examples

### Allow Finance users to access Finance DB only

```
Consumer:  Finance-Analysts (scope: user/ad_group = Finance-Analysts)
Provider:  Finance-DB-Cluster (scope: orchestrator label app = financedb)
Action:    ALLOW TCP 1521, TCP 5432
```

### Block non-compliant devices from production

```
Consumer:  Non-Compliant-Endpoints (scope: device/mdm_compliance = Non-Compliant)
Provider:  Production-App-Servers (scope: environment = production)
Action:    DENY ALL
```

### Lateral movement alert — Finance user on unexpected server

```
Alert Rule (Traffic):
  Source scope:      Finance-Endpoints
  Destination scope: NOT (Finance-App-Servers OR Finance-DB-Cluster)
  Protocol:          TCP
  Severity:          MEDIUM
  Notify:            Syslog → Splunk
```

### VPN remote access validation

```
Consumer:  VPN-Remote-Users (scope: user/auth_method = VPN)
Provider:  Internal-Workloads
Action:    ALLOW only to explicitly defined application scopes
           DENY all other destinations
```

---

## 12. Verification

### 12.1 Verify connector is connected

In CSW UI: **Manage > Virtual Appliances > [Edge Appliance] > Connectors**  
ISE connector should show **Status: Connected**

### 12.2 Verify endpoints are appearing in inventory

1. Navigate to **Inventory > Workloads**
2. Search for a known ISE-managed endpoint IP
3. Confirm user/device labels are populated

### 12.3 Verify pxGrid subscription in ISE

1. In ISE: **Administration > pxGrid Services > Live Logs**
2. Confirm CSW connector is listed as an active subscriber to the **session** and **sg** topics

### 12.4 Check connector logs on Edge appliance

SSH to the Edge appliance and check:

```bash
tail -f /usr/local/tet/log/ise-connector.log
```

Look for:
- `Connected to ISE pxGrid node` — successful connection
- `Fetched N endpoints from ISE snapshot` — initial data pull
- `Received session update for IP X.X.X.X` — live updates flowing

### 12.5 Verify SGT labels in CSW inventory

1. In CSW **Inventory**, filter by `user/sgt = Finance`
2. Confirm expected endpoints appear
3. Cross-reference with ISE **Operations > RADIUS > Live Logs** to confirm session is active

---

## 13. Periodic Tasks and Data Refresh

| Task | Frequency | Description |
|------|-----------|-------------|
| Endpoint snapshot | Every **20 hours** | Full pull of all active ISE sessions |
| User label update | Every **2 minutes** | Pushes LDAP + ISE endpoint label changes to CSW |
| SGT database refresh | On pxGrid event | Real-time SGT name → value map update |
| LDAP snapshot | Every **24 hours** (configurable) | Refresh local user attribute cache from AD |
| pxGrid live subscription | Continuous | Real-time session start/end events streamed from ISE |

> **Note:** After a Cisco ISE upgrade, the ISE connector must be **reconfigured** with new certificates generated by the upgraded ISE node.

---

## 14. Limits

| Metric | Limit |
|--------|-------|
| ISE instances per ISE connector | 20 |
| ISE connectors per CSW Edge appliance | 1 |
| ISE connectors per CSW tenant | 1 |
| ISE connectors per CSW cluster | 150 |
| Maximum ISE endpoints per connector | 400,000 |

> One ISE connector per tenant means: design your subnet filter and VRF assignments carefully if you have multiple ISE deployments or a large environment split across VRFs.

---

## 15. Troubleshooting

### Connector shows "Disconnected" or fails to connect

| Symptom | Check |
|---------|-------|
| SSL handshake failure | Verify client cert SAN matches the CN in the config; confirm ISE CA cert is the correct root that signed the pxGrid server cert |
| Certificate missing SAN | CSW 3.7+ requires SAN on ISE pxGrid node cert — regenerate ISE cert with SAN |
| "Connection refused" | Confirm TCP 8910 is open from Edge appliance to ISE pxGrid node; check ISE firewall rules |
| "Pending approval" in ISE | Go to ISE **pxGrid Services > All Clients** and approve the CSW connector |
| Password-protected key | Strip the passphrase: `openssl rsa -in csw-ise-connector.key -out csw-ise-connector-clear.key` |

### Endpoints not appearing in CSW inventory

| Symptom | Check |
|---------|-------|
| No endpoints in inventory | Confirm VRF configuration on Edge appliance covers the endpoint subnet |
| Wrong VRF | Verify **Agent Remote VRF Configurations** — subnet must match ISE endpoint IP range |
| Endpoints present but no labels | Confirm ISE is actively authenticating endpoints (check ISE **Live Logs**); endpoints must have an active session |

### ISE attribute fields missing

| Symptom | Check |
|---------|-------|
| `user/sgt` missing | Confirm TrustSec / SGT is enabled on ISE and endpoints are assigned SGTs |
| `device/mdm_compliance` missing | Requires MDM integration configured in ISE (e.g., Intune, JAMF) |
| `user/ad_group` missing | Requires LDAP/AD configuration in ISE connector |

### After ISE upgrade — connector fails

ISE upgrades regenerate pxGrid certificates. After upgrading ISE:
1. Delete the ISE connector in CSW
2. Export new ISE CA cert from upgraded ISE
3. Regenerate client CSR and get re-signed by CA
4. Recreate the ISE connector with updated certificates

---

## 16. Related Resources

| Repository | Description |
|------------|-------------|
| [CSW-Secure-Firewall-Integration-Guide](https://github.com/chandrapati/CSW-Secure-Firewall-Integration-Guide) | NSEL flow ingestion from Cisco Secure Firewall (FTD/ASA) via FMC |
| [csw-splunk-integration](https://github.com/chandrapati/csw-splunk-integration) | CSW syslog alert stream → Splunk SIEM |
| [CSW-Agent-Installation-Guide](https://github.com/chandrapati/CSW-Agent-Installation-Guide) | Deploy and configure CSW agents across Linux/Windows workloads |
| [CSW-Policy-Lifecycle](https://github.com/chandrapati/CSW-Policy-Lifecycle) | Policy discovery, review, approval, and enforcement workflow |
| [CSW-Compliance-Mapping](https://github.com/chandrapati/CSW-Compliance-Mapping) | Map CSW controls to NIST, PCI-DSS, HIPAA, CIS frameworks |
| [CSW-Operations-Toolkit](https://github.com/chandrapati/CSW-Operations-Toolkit) | Day-2 operations scripts: health checks, tenant reporting, policy analysis |
| [CSW-User-Education](https://github.com/chandrapati/CSW-User-Education) | Onboarding materials and concept guides for new CSW users |

> **Suggested path for a new customer:**  
> CSW-User-Education → CSW-Agent-Installation-Guide → CSW-Policy-Lifecycle → csw-ise-integration → csw-splunk-integration → CSW-Compliance-Mapping

---

*Community reference guide — Cisco Solutions Engineering. Not an official Cisco product document.*  
*For official documentation, see [Cisco Secure Workload Documentation](https://www.cisco.com/c/en/us/products/security/tetration/index.html).*
