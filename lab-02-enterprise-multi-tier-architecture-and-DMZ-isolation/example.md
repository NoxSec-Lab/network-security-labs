# Lab 2: Enterprise Multi-Tier Architecture & DMZ Isolation

## Overview
This lab expands our enterprise network architecture into a production-grade 3-tier topology within GNS3. Building on Lab 1's foundation, we introduce an isolated Demilitarized Zone (DMZ) alongside our existing internal LAN and Management subnets. By deploying a dedicated boundary router (`DMZ-VyOS`) connected directly to our edge security firewall (`Edge-pfSense-Firewall`), this project establishes strict perimeter controls, stateful traffic rules, and clear routing paths to safely host public-facing services without exposing internal corporate networks.

---

## Network Architecture & Topology

![Lab Topology](assets/01-gns3-topology.png)

* **Objective:** Design and validate a secure 3-tier enterprise network that isolates public web services from internal administrative and application systems.
* **Architecture Breakdown:**
  * **Edge Security Perimeter:** A pfSense firewall managing WAN connectivity (Internet NAT), internal transit (`10.0.0.1/24`), DMZ transit (`10.0.1.1/24`), and out-of-band management via a dedicated Windows Admin host (`192.168.100.0/24`).
  * **Core Routing Layer:** A `Core-VyOS` router managing inter-VLAN traffic between internal subnets (`192.168.10.0/24` Management and `192.168.20.0/24` Application).
  * **DMZ Routing Layer:** A dedicated `DMZ-VyOS` router servicing the DMZ network (`172.16.30.0/24`) and enforcing strict boundary firewall rules.
  * **Workload Endpoints:**
    * **Management Zone:** `ubuntu-mgmt-svr01` (`192.168.10.10/24`).
    * **Application Zone:** `ubuntu-app-svr02` (`192.168.20.10/24`).
    * **DMZ Public Host:** `DMZ-Public-Server` (`172.16.30.10/24`).

---

## DMZ Routing & Boundary Firewall Enforcement

### DMZ Router Interface & Gateway Provisioning

![DMZ Router Initial Configuration](assets/02-dmz-vyos-base-config.png)

* **Objective:** Assign network addresses across transit and local DMZ interfaces on `DMZ-VyOS`, and set up default outbound routing toward the pfSense edge firewall.
* **Key Implementation Details:**
  * **Transit Interface (`eth0`):** Configured with IP `10.0.1.2/24` to establish the dedicated link back to pfSense.
  * **DMZ Host Interface (`eth1`):** Configured with IP `172.16.30.1/24` to act as the default gateway for public-facing servers.
  * **Default Outbound Route:** Set a static default route (`0.0.0.0/0`) pointing all outbound traffic to next-hop `10.0.1.1` (pfSense DMZ interface `e3`).

---

### DMZ Router Interface State Verification

![DMZ Router Interface Summary](assets/03-dmz-vyos-interfaces-show.png)

* **Objective:** Verify operational link status (`u/u`), IP address bindings, and descriptions across all physical and logical interfaces on `DMZ-VyOS`.
* **Key Implementation Details:**
  * **Transit Link (`eth0`):** Confirmed active state (`u/u`) with `10.0.1.2/24` bound and standard 1500 MTU.
  * **DMZ Host Link (`eth1`):** Confirmed active state (`u/u`) with `172.16.30.1/24` bound and standard 1500 MTU.
  * **Unused Interfaces:** Confirmed that interfaces `eth2` through `eth9` remain unassigned and administratively down (`u/D`) to shrink the attack surface.

---

### Boundary Firewall Ruleset Construction & Forward Filter Binding

![DMZ Firewall Policy & Forward Filter Hierarchy](assets/04-dmz-firewall-ruleset.png)

* **Objective:** Construct a stateful firewall policy (`DMZ_ISOLATION`) on `DMZ-VyOS` to ensure a compromised DMZ server cannot pivot into internal subnets, while still allowing internal management access and outbound internet updates.
* **Key Implementation Details & Rationale:**
  * **Stateful Flow Tracking (Rule 10):** Allowed `established` and `related` connection states. This lets return traffic flow back for sessions initiated by internal admins (like SSH management) without opening permanent backdoors from the DMZ back to internal networks.
  * **Strict Internal Subnet Isolation (Rules 20 & 30):** Configured explicit `drop` rules targeting internal subnets `192.168.10.0/24` (Management) and `192.168.20.0/24` (Application). Even if an attacker gains control of the public web server, any connection attempt toward internal networks is dropped immediately at the boundary interface.
  * **Default Pass for Internet Egress:** Applied `default-action accept` on the `DMZ_ISOLATION` chain. Because rules 20 and 30 explicitly block internal destinations, all remaining unmatched traffic (such as outbound web traffic for updates) can freely pass out through pfSense.
  * **Forward Filter Mapping (`eth1` Ingress):** Bound `forward filter rule 10` to `inbound-interface eth1`, jumping directly to `DMZ_ISOLATION`. Inspecting traffic the moment it enters the router stops unauthorized packets before they ever reach the routing engine.

---

## Edge Gateway & Transit Interface Setup (pfSense Console)

### Interface Mapping & Network Overview

![pfSense Interface Mapping & Network Overview](assets/05-pfsense-dmz-interface-config.png)

* **Objective:** Map physical-to-logical interfaces on the pfSense firewall and assign static transit IP addresses to tie the multi-tier topology together.
* **Key Implementation Details:**
  * **Interface Allocation:** Bound `em0` to WAN (`192.168.42.140/24 via DHCP`), `em1` to LAN (`10.0.0.1/24`), `em3` as **OPT1** (`10.0.1.1/24`) for the DMZ transit link, and `em2` as **OPT2** (`192.168.100.1/24`) for out-of-band administrative access.
  * **Transit Link Isolation:** Setting **OPT1 (`em3`)** as a dedicated `10.0.1.0/24` subnet keeps DMZ transit traffic completely separate from internal corporate LAN traffic at the hardware level.

---

### End-to-End Transit Link Verification

![ICMP Verification from pfSense to DMZ-VyOS](assets/06-pfsense-to-dmz-vyos-ping.png)

* **Objective:** Confirm basic Layer 3 reachability across the transit link connecting pfSense (`10.0.1.1`) to the `DMZ-VyOS` router (`10.0.1.2`).
* **Key Implementation Details:**
  * **Console Connectivity Test:** Transmitted ICMP echo requests directly from the pfSense menu before configuring web GUI rules to confirm cable and IP bindings were active.
  * **Baseline Link Performance:** Confirmed active bidirectional reachability with **0% packet loss** (~9.6ms average latency), verifying that underlying switch connectivity was solid before testing firewall policies.

---

## Upstream Routing & Gateway Definitions (pfSense Web GUI)

### System Routing Table & Default Gateway Audit

![pfSense System Gateways Overview](assets/07_pfSense_Gateways_Overview.png)

* **Objective:** Review the default edge routing configuration in pfSense before defining downstream routes for internal networks.
* **Key Implementation Details:**
  * **Verifying Base Internet Egress:** Confirmed `WAN_DHCP` (`192.168.42.1`) as the default system gateway with active health monitoring.
  * **Preventing Routing Loops:** Kept default gateway handling on `Automatic` so internet-bound flows default to WAN, rather than accidentally looping back into internal routers.

---

### DMZ Transit Gateway Definition (`DMZ_GW`)

![pfSense DMZ Gateway Configuration](assets/08_pfSense_Add_DMZ_Gateway.png)

* **Objective:** Define `DMZ_GW` pointing to `10.0.1.2` on the `OPT1` interface as the designated next-hop for all DMZ-bound traffic.
* **Key Implementation Details:**
  * **Gateway Object Creation:** Created an explicit gateway entry for `10.0.1.2` on `OPT1` so pfSense knows where to forward packets heading for the `172.16.30.0/24` network behind `DMZ-VyOS`.
  * **Link Monitoring:** Enabled health probes targeting `10.0.1.2` to track transit link latency and packet loss directly from the pfSense dashboard.

---

### Core Router Gateway Definition (`CORE_VYOS_GW`)

![pfSense Core VyOS Gateway Configuration](assets/09_pfSense_Core_VyOS_Gateway_Configuration.png)

* **Objective:** Define `CORE_VYOS_GW` pointing to `10.0.0.2` on the `LAN` interface to handle downstream routing for internal subnets.
* **Key Implementation Details:**
  * **Connecting Edge to Core:** Created an explicit gateway for `10.0.0.2` so pfSense knows how to send return traffic back to internal segments like Management (`192.168.10.0/24`).
  * **Path Separation:** Keeping `CORE_VYOS_GW` on the LAN interface and `DMZ_GW` on the OPT1 interface forces pfSense to maintain strict physical and logical isolation between internal and DMZ traffic paths.

---

### Gateway Table Verification & Health Audit

![pfSense Configured Gateways Overview](assets/10_pfSense_Gateways_List_Configured.png)

* **Objective:** Validate that all internal and external gateways are active, online, and passing continuous health checks.
* **Key Implementation Details:**
  * **Gateway Health Confirmation:** Confirmed green checkmark status for both `DMZ_GW` (`10.0.1.2`) and `CORE_VYOS_GW` (`10.0.0.2`), verifying active ICMP response.
  * **Prerequisite for Static Routes:** Validated gateway health to ensure static routes dependent on these targets would activate cleanly without dropping packets.

---

## Downstream Routing & Egress NAT Configuration (pfSense Web GUI)

### DMZ Subnet Static Route Injection

![pfSense Static Route to DMZ Host Subnet](assets/11_pfSense_Static_Route_DMZ_Subnet.png)

* **Objective:** Map a static route directing `172.16.30.0/24` traffic to `DMZ_GW` (`10.0.1.2`).
* **Key Implementation Details:**
  * **Resolving Downstream Reachability:** Added a static route for `172.16.30.0/24` via `10.0.1.2`, teaching pfSense how to reach the non-locally connected DMZ host network.
  * **Delegating Security Enforcement:** Forwarding this entire subnet to `DMZ-VyOS` delegates local host filtering to the boundary router.

---

### Internal Enterprise Subnet Static Routes

![pfSense Static Route to Management Subnet](assets/12_pfSense_Static_Route_Management_Subnet.png)

![pfSense Static Route to Application Subnet](assets/13_pfSense_Static_Route_Application_Subnet.png)

* **Objective:** Map static routes for internal zones—Management (`192.168.10.0/24`) and Application (`192.168.20.0/24`)—pointing to `CORE_VYOS_GW` (`10.0.0.2`).
* **Key Implementation Details:**
  * **Symmetric Return Paths:** Added static routes directing internal corporate traffic back through `10.0.0.2`, eliminating asymmetric routing issues for internet-bound traffic.
  * **Traffic Isolation:** Ensured internal corporate routes remain cleanly separated from DMZ transit paths.

---

### Ingress Firewall Policy for DMZ Transit Interface

![pfSense Ingress Pass Rule for OPT1 Interface](assets/14_pfSense_Firewall_Rule_OPT1_Pass.png)

* **Objective:** Create an ingress pass rule on interface `OPT1` allowing traffic originating from `172.16.30.0/24` to pass through pfSense.
* **Key Implementation Details:**
  * **Overriding Default Block:** Overrode pfSense's default "deny-all" rule on optional interfaces (`OPT1`) by explicitly permitting source network `172.16.30.0/24`.
  * **Enabling Internet Egress:** Permitted DMZ traffic to reach the edge NAT engine, allowing web servers to download packages and system updates.

---

### Advanced Outbound NAT for DMZ Egress

![pfSense Outbound NAT Mapping for DMZ Subnet](assets/15_pfSense_Outbound_NAT_Rule.png)

* **Objective:** Configure an Outbound NAT rule translating private source IPs from `172.16.30.0/24` to the pfSense `WAN address` when exiting to the internet.
* **Key Implementation Details:**
  * **RFC 1918 Translation:** Applied Port Address Translation (PAT) so private DMZ IPs (`172.16.30.0/24`) can communicate on the public internet.
  * **Internal Topology Hiding:** Masked internal DMZ network addressing behind the edge WAN interface.

---

## DMZ Server Provisioning & End-to-End Egress Verification

### DMZ Server Network & Hostname Initialization

![DMZ Server IP and Default Route Configuration](assets/16_DMZ_Server_IP_Route_Config.png)

* **Objective:** Set up `DMZ-Public-Server` with static IP addressing (`172.16.30.10/24`) and direct default traffic to gateway `172.16.30.1`.
* **Key Implementation Details:**
  * **Hostname Standardization:** Updated hostname from `ubuntu-cloud` to `DMZ-Public-Server` for clean logging and identification across the network.
  * **Static Network Binding:** Assigned `172.16.30.10/24` to interface `ens3` to place the server into the isolated DMZ segment.
  * **Default Route Definition:** Directed default egress (`0.0.0.0/0`) to `172.16.30.1` (`DMZ-VyOS` `eth1`), making the boundary router responsible for inspecting all outbound traffic.

---

### End-to-End DMZ Reachability & Egress Audit

![DMZ Server Step-by-Step ICMP Verification](assets/17_DMZ_Server_Ping_Test_Verification.png)

* **Objective:** Test end-to-end connectivity step-by-step from `DMZ-Public-Server` out to the public internet (`8.8.8.8`).
* **Key Implementation Details:**
  * **Local Gateway Check (`172.16.30.1`):** Reached local gateway with **0% packet loss** (~8.4ms RTT), confirming local switch and link health.
  * **Transit Router Check (`10.0.1.2`):** Reached `DMZ-VyOS` transit interface with **0% packet loss** (~9.1ms RTT), confirming internal router forwarding across interfaces.
  * **Edge Firewall Check (`10.0.1.1`):** Reached pfSense `OPT1` interface with **0% packet loss** (~30.4ms RTT), confirming pfSense's ingress pass rule was working.
  * **Public Internet Check (`8.8.8.8`):** Reached Google DNS with **0% packet loss** (~36.8ms RTT), proving Outbound NAT and internet routing were fully operational.

---

## Security Policy Enforcement & Isolation Verification

### DMZ Outbound Isolation Enforcement (Ping Tests)

![DMZ Server Outbound Ping Timeout to Internal Subnets](assets/18_DMZ_Server_Isolation_Ping_Timeout.png)

* **Objective:** Validate that the DMZ server cannot reach internal subnets by attempting pings toward Management (`192.168.10.1`) and Application (`192.168.20.1`) gateways.
* **Key Implementation Details:**
  * **Zero-Trust Block Verification:** Both ping tests resulted in **100% packet loss**, confirming that DMZ hosts cannot initiate connections into internal corporate segments.
  * **Rule Enforcement Validation:** Verified that rules 20 and 30 on `DMZ-VyOS` actively drop unauthorized traffic at the DMZ boundary.

---

### Packet Capture Inspection on Boundary Router (`tcpdump`)

![VyOS Interface Packet Capture Dropped Packets](assets/19_DMZ_VyOS_tcpdump_Dropped_Packets.png)

* **Objective:** Capture live packets using `tcpdump` on `DMZ-VyOS` interface `eth1` to confirm that unauthorized traffic is dropped by firewall rules.
* **Key Implementation Details:**
  * **Ingress Packet Tracking:** Captured ICMP echo requests arriving from `172.16.30.10` targeting `192.168.10.1` and `192.168.20.1`.
  * **Firewall Drop Confirmation:** Confirmed that while packets arrived on `eth1`, zero packets were forwarded out toward internal networks or pfSense, proving the firewall silently dropped the unauthorized flows.

---

### Inbound Management Access & Service Audit (`nc` / SSH)

![Management Server Inbound Service Access Test](assets/20_Inbound_Service_Access_Test.png)

* **Objective:** Verify that an internal management server (`ubuntu-mgmt-svr01`) can initiate connections to the DMZ server over SSH (port 22).
* **Key Implementation Details:**
  * **Administrative Ping Check:** Confirmed ICMP reachability from `192.168.10.10` to `172.16.30.10` with **0% packet loss**.
  * **SSH Service Audit:** Executed `nc -zv 172.16.30.10 22`, receiving `Connection to 172.16.30.10 22 port [tcp/ssh] succeeded!`.
  * **Stateful Symmetry Validation:** Confirmed that rule 10 (`state established, related`) allows authorized internal admins to connect in, while return traffic passes back out without opening the internal network to DMZ-initiated threats.

---

## Key Takeaways & Engineering Insights

* **Multi-Tier Isolation Strategy:** Placing public services in a dedicated DMZ behind a separate boundary router (`DMZ-VyOS`) establishes a hard security boundary. Even if a public web application is fully compromised, strict inbound filtering prevents attackers from pivoting into internal management or corporate zones.
* **Stateful Flow Dynamics:** Using stateful inspection (`established`, `related`) simplifies security management. Internal hosts can administer DMZ services without requiring wide-open return rules that could compromise internal security.
* **Edge Routing & NAT Alignment:** Routing traffic across a multi-tier network requires coordinated static routes and NAT rules on edge firewalls like pfSense. Without precise static routes for internal subnets and explicit Outbound NAT mappings, DMZ servers lose internet egress and return paths fail.
