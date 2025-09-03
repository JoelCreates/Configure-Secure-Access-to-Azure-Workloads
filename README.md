# Configure Secure Access to Azure Workloads (Networking)

## Overview
This project documents how I designed and implemented secure access controls for Azure workloads using native networking features. I configured Azure Firewall as a central egress control for a protected subnet, enforced least-privilege east–west access with NSGs and ASGs, built cross-VNet connectivity with peering, and implemented private DNS for reliable name resolution. The objective was to restrict outbound internet access to a single approved service, tighten internal access paths, and make inter-VNet communication predictable and resolvable by DNS.

**Credential:** [Microsoft Applied Skills – Configure secure access to your workloads using Azure networking](https://learn.microsoft.com/api/credentials/share/en-gb/JoelAmoaniOkanta-1857/13D79C36A349B6DB?sharingId)

---

## Objectives
- Harden outbound internet access for **Subnet1-1** so that hosts could reach **only** the approved external service **RelecloudDataService** at **131.107.3.210:9000 (TCP)** via **Azure Firewall**.
- Enforce least-privilege internal access with an **Application Security Group (ASG)** and a **Network Security Group (NSG)** so only specific app servers could reach a database tier on **TCP/1433**.
- Prepare a second landing zone (**VNet-VM4** in North Europe) and establish **VNet peering** to existing workloads.
- Provide **private DNS** resolution across VNets using a **Private DNS zone** with **auto-registration** so services could be reached by hostname (`vm2.contoso.com`) rather than by IP.

---

## Architecture
**Before**: Subnet1-1 had direct outbound access to the internet and broad internal access.  
**After**: All egress from Subnet1-1 traversed **Azure Firewall**; only a specific destination (`131.107.3.210:9000`) was allowed. Internal access to the database tier was restricted to members of an ASG over **TCP/1433**. A new VNet (North Europe) was peered to the existing hub/workload VNet, and a Private DNS zone enabled name-based connectivity.

```
      (Internet)
          ^
          | (only 131.107.3.210:9000 allowed)
    +-----+---------------------+
    |     Azure Firewall        |
    |  - AzureFirewallSubnet    |
    |  - AzureFirewallMgmtSubnet|
    +-----+---------------------+
          ^   (UDR 0.0.0.0/0 to FW)
          |
  +-------+--------+
  |   VNet1        |
  |  Subnet1-1     |---- NSG (allow 1433 from ASG)
  +-----------------+            ^
                                 |
                         +-------+---------+
                         |  ASG (VM2, VM3) |
                         +-----------------+

  +-------------------+         Peering          +-------------------+
  |     VNet2         | <----------------------> |    VNet-VM4       |
  | (West Europe)     |                          |  (North Europe)   |
  +-------------------+                          +-------------------+

         Private DNS zone: contoso.com (linked to VNet2 and VNet-VM4)


```


---

## Key Resources and Parameters
- **Virtual networks**: `VNet1` (existing), `VNet2` (existing), `VNet-VM4` (new, North Europe)
- **Subnets** (VNet1): `Subnet1-1`, `AzureFirewallSubnet` (mandatory), `AzureFirewallManagementSubnet` (for forced tunnelling/management plane)
- **Azure Firewall**: Standard/Premium SKU as available; data plane + management plane public IPs
- **Firewall Policy**: `FirewallPolicy1` (attached to the firewall)
- **Firewall Rules**:
  - **Network Rule Collection**: `Allow-RelecloudDataService` (Priority 100, Action: Allow)
    - Rule: Source = `Subnet1-1 CIDR`, Destination = `131.107.3.210`, Protocol = `TCP`, Port = `9000`
  - **Network Rule Collection**: `Deny-All-Internet` (Priority 200, Action: Deny)
    - Rule: Source = `Subnet1-1 CIDR`, Destination = `*`, Protocol = `Any`, Port = `*`
- **Route Table**: `Subnet1-1-RT` (associated to Subnet1-1)
  - Route: `0.0.0.0/0` → Next hop type `Virtual appliance` → Next hop IP = **Azure Firewall private IP**
- **ASG**: `AppServers-ASG` (members: `VM2`, `VM3`)
- **NSG**: `Subnet1-1-NSG`
  - Inbound allow: Source = `AppServers-ASG`, Destination = `Subnet1-1`, Protocol = `TCP`, Port = `1433`, Action = `Allow`, Priority < Deny
  - Default **Deny** lower down the priority stack
- **New VNet**: `VNet-VM4` (North Europe), address space e.g. `10.3.0.0/16`, subnet `10.3.1.0/24`
- **Peering**: `VNet-VM4` ↔ `VNet2` (allow forwarded traffic as required)
- **Private DNS**: `contoso.com` (Private DNS zone), linked to `VNet2` and `VNet-VM4`, **auto-registration enabled**

---

## Implementation

### 1) Azure Firewall as Controlled Egress for Subnet1-1
**Objective:** Force all outbound traffic from Subnet1-1 through Azure Firewall and allow **only** `131.107.3.210:9000/TCP`.

1. **Deployed Azure Firewall** into `VNet1`, creating both:
   - `AzureFirewallSubnet` (data plane) — **mandatory** and must be named exactly
   - `AzureFirewallManagementSubnet` (management plane) — used for forced tunnelling/management
   - Allocated public IP(s) for data/management as required by SKU.
2. **Created `FirewallPolicy1`** and attached it to the firewall.
3. **Added rule collections**:
   - `Allow-RelecloudDataService` (Network rules, Priority 100, **Allow**):
     - Source addresses: CIDR of `Subnet1-1` (e.g., `10.0.1.0/24`)
     - Destination addresses: `131.107.3.210`
     - Protocol: `TCP`, Port: `9000`
   - `Deny-All-Internet` (Network rules, Priority 200, **Deny**):
     - Source: `Subnet1-1` CIDR, Destination: `*`, Protocol: `Any`, Port: `*`
4. **Created `Subnet1-1-RT` route table** and associated it with `Subnet1-1`.
   - Route: `0.0.0.0/0` → Next hop: `Virtual appliance` → Next hop IP = **Azure Firewall private IP**
5. **Validation**:
   - From a host in `Subnet1-1`, `Test-NetConnection 131.107.3.210 -Port 9000` succeeded.
   - Generic outbound internet (e.g., `Test-NetConnection 8.8.8.8 -Port 443`) failed, as expected.

**Security rationale:** This replaced broad egress with a centralised policy enforcement point and a single approved destination. Network rule collections are suitable for IP/port destinations; Application rules are used when filtering by FQDNs.

---

### 2) Internal Least-Privilege with NSG + ASG
**Objective:** Ensure only `VM2` and `VM3` (application tier) could connect to the database tier in `Subnet1-1` over **TCP/1433**.

1. **Created ASG `AppServers-ASG`** and added `VM2` and `VM3` NICs as members.
2. **Created NSG `Subnet1-1-NSG`** and associated it to **Subnet1-1**.
3. **Added NSG inbound rule**:
   - Source: `AppServers-ASG`
   - Destination: `Subnet1-1` (or DB NICs if using NIC-level NSG)
   - Protocol: `TCP`
   - Port: `1433`
   - Action: `Allow`
   - Priority: higher than default Deny
4. **Validation**:
   - From `VM2/VM3`: `Test-NetConnection <db-host> -Port 1433` succeeded.
   - From non-ASG hosts: `Test-NetConnection <db-host> -Port 1433` failed.

**Security rationale:** ASGs let rules target *roles* rather than ever-changing IPs. The NSG then expresses intent: “Only app servers may talk to DB on 1433.”

---

### 3) New VNet and VNet Peering
**Objective:** Prepare a new workload network in **North Europe** and establish secure connectivity to existing services.

1. **Provisioned `VNet-VM4`** in **North Europe** with address space `10.3.0.0/16`, created subnet `10.3.1.0/24`.
2. **Created VNet peering** between `VNet-VM4` and `VNet2`.
   - Enabled traffic in both directions; forwarded traffic / gateway settings as required by topology.
3. **Validation**:
   - From a `VNet-VM4` host: reached `VNet2` resources by IP.

**Security rationale:** Peering provides private, low-latency VNet-to-VNet connectivity without transiting the public internet, suitable for multi-region or DR topologies.

---

### 4) Private DNS for Cross-VNet Name Resolution
**Objective:** Allow clients in `VNet-VM4` to resolve and reach services in `VNet2` by hostname under `contoso.com`.

1. **Created Private DNS zone** `contoso.com`.
2. **Linked the zone** to **both** `VNet-VM4` and `VNet2`.
   - Enabled **auto-registration** on the VNet hosting the registering hosts (per design).
3. **Ensured `VM2` registered** as `vm2.contoso.com`.
4. **Validation**:
   - From `VNet-VM4`:  
     - `nslookup vm2.contoso.com` returned the private IP of `VM2`.
     - `Test-NetConnection vm2.contoso.com -Port 1433` succeeded (subject to NSG/ASG policy).

**Security rationale:** Private DNS removes reliance on static IPs and enables policy that follows names/roles, which is essential at scale.

---

## Validation Summary
- **Egress control**: Only `131.107.3.210:9000/TCP` reachable from Subnet1-1; all other outbound internet blocked by Azure Firewall policy and UDR.
- **East–west control**: Only `VM2`/`VM3` (ASG members) could reach DB over TCP/1433 enforced by NSG.
- **Inter-VNet connectivity**: `VNet-VM4` (North Europe) successfully peered to `VNet2` and reached services.
- **DNS**: `vm2.contoso.com` resolved privately from `VNet-VM4` and was reachable in accordance with NSG/ASG rules.

---

## Troubleshooting Notes and Gotchas
- **Subnet names** for Azure Firewall are strict: `AzureFirewallSubnet` (data plane) and, when using forced tunnelling/management, `AzureFirewallManagementSubnet`. Misnaming prevents deployment.
- **UDR precedence**: Ensure the route table with `0.0.0.0/0 → FW private IP` is **associated** to `Subnet1-1`; otherwise traffic will bypass the firewall.
- **Rule ordering**: Firewall rule collections evaluate by priority; put **Allow** (priority 100) above **Deny-All** (priority 200).
- **NSG evaluation**: Confirm there are no broader “Allow Any” rules above the specific 1433 rule that would invalidate the test.
- **ASG membership**: Add the **NIC** (not the VM object) to the ASG; otherwise the NSG rule will not match.
- **Peering settings**: If you need transitive routing or NVA inspection, set “allow forwarded traffic” and design the route domain accordingly.
- **Private DNS links**: Link the zone to **each** VNet that must resolve the names; enable auto-registration on the VNet hosting registering workloads.

---

## Outcomes
- Replaced broad outbound access with a **policy-driven egress architecture** using Azure Firewall, reducing data exfiltration and C2 risk.
- Implemented **least-privilege east–west access** to the database tier using ASGs and NSGs, cutting lateral-movement paths.
- Enabled a **multi-region topology** with VNet peering that preserved private connectivity.
- Delivered **DNS that follows services** (auto-registration), removing brittle IP dependencies and reducing operational toil.

---

## Why This Matters
- **Containment and exfiltration control**: Centralised egress via a stateful firewall is a core control against data loss and command-and-control.
- **Lateral movement reduction**: ASG-backed NSG rules align access with application roles, not addresses, and meaningfully reduce internal blast radius.
- **Scalable multi-VNet design**: Peering and private DNS are foundational for modern hub-and-spoke and multi-region architectures.
- **Operational clarity**: Name-based policies and deterministic routing simplify incident response and change management.

---

## Credential
[Microsoft Applied Skills – Configure secure access to your workloads using Azure networking](https://learn.microsoft.com/api/credentials/share/en-gb/JoelAmoaniOkanta-1857/13D79C36A349B6DB?sharingId)
