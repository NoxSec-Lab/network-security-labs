Lab 2: Enterprise Multi-Tier Architecture & DMZ Isolation
Overview
This lab expands our baseline enterprise architecture into a production-grade 3-tier network topology. By introducing a Demilitarized Zone (DMZ) alongside an Internal LAN and a dedicated Management VLAN, this project enforces strict ingress/egress filtering rules, isolates public-facing web services, and sets up a controlled target environment for multi-stage penetration testing and pivoting exercises.


Core Security Objectives & ACL Matrix
To enforce zero-trust isolation on the VyOS core router, we define four strict firewall traffic flows:
DMZ Isolation Rule: Hosts in the DMZ (172.16.30.0/24) MUST NOT initiate any connection into the Internal LAN (192.168.20.0/24) or Management Zone (192.168.10.0/24).
DMZ Service Access: Management and Internal LAN hosts can access HTTP/HTTPS (TCP 80/443) and SSH (TCP 22) on DMZ hosts.
Stateful Ingress Processing: Return traffic from the DMZ is permitted only if the connection was initiated by internal hosts (established, related).
Edge Gateway Routing: Downstream static routes on pfSense must be updated to route return traffic for 172.16.30.0/24 back to next-hop 10.0.0.2.

----




# Lab 2: Enterprise Multi-Tier Architecture & DMZ Isolation

## Overview
This lab expands our enterprise network architecture into a production-grade 3-tier topology within GNS3, introducing a Demilitarized Zone (DMZ) alongside the internal LAN and Management subnets. By deploying a dedicated DMZ router (`DMZ-VyOS`) connected directly to the `Edge-pfSense-Firewall`, this project enforces strict perimeter isolation, configures stateful ingress/egress filtering rules, and establishes structured routing paths to securely host public-facing services while protecting internal workloads.

---

## Network Architecture & Topology

![Lab Topology](assets/01-gns3-topology.png)

* **Objective:** Design and validate a secure 3-tier enterprise architecture that completely isolates public-facing DMZ servers from internal administrative and application segments.
* **Architecture Breakdown:**
  * **Edge Security Perimeter:** A pfSense firewall managing WAN access (Internet-NAT), internal transit (`10.0.0.1/24`), DMZ transit (`10.0.1.1/24`), and out-of-band management connectivity via a dedicated Windows Admin host (`192.168.100.0/24`).
  * **Core Routing Layer:** A `Core-VyOS` router handling inter-VLAN routing for internal subnets (`192.168.10.0/24` Management and `192.168.20.0/24` Application).
  * **DMZ Routing Layer:** A dedicated `DMZ-VyOS` router servicing the DMZ segment (`172.16.30.0/24`) and enforcing strict ingress/egress firewall boundary policies.
  * **Isolated Workloads:**
    * **Management Zone:** `ubuntu-mgmt-svr01` (`192.168.10.10/24`).
    * **Application Zone:** `ubuntu-app-svr02` (`192.168.20.10/24`).
    * **DMZ Public Host:** `DMZ-Public-Server` (`172.16.30.10/24`).

---
## DMZ Routing & Boundary Firewall Enforcement

### DMZ Router Interface & Gateway Provisioning

![DMZ Router Initial Configuration](assets/02-dmz-vyos-base-config.png)

* **Objective:** Assign Layer 3 addressing across transit and local DMZ interfaces on `DMZ-VyOS` and configure default outbound routing toward the pfSense edge firewall.
* **Key Implementation Details:**
  * **Transit Interface (`eth0`):** Assigned IP `10.0.1.2/24` with description `TRANSIT-TO-PFSENSE-E3` to establish the upstream interconnection.
  * **DMZ Host Interface (`eth1`):** Assigned IP `172.16.30.1/24` with description `DMZ-HOST-SEGMENT` to act as the default gateway for public-facing servers.
  * **Default Gateway:** Applied static default route (`0.0.0.0/0`) pointing to next-hop `10.0.1.1` (pfSense DMZ interface `e3`).

---

### DMZ Router Interface State Verification

![DMZ Router Interface Summary](assets/03-dmz-vyos-interfaces-show.png)

* **Objective:** Verify operational link status (`u/u`), IP address bindings, and descriptions across all physical and logical interfaces on `DMZ-VyOS`.
* **Key Implementation Details:**
  * **Transit Link (`eth0`):** Confirmed active state (`u/u`) with `10.0.1.2/24` bound and standard **1500 MTU**.
  * **DMZ Host Link (`eth1`):** Confirmed active state (`u/u`) with `172.16.30.1/24` bound and standard **1500 MTU**.
  * **Inactive Interfaces:** Verified interfaces `eth2` through `eth9` remain unconfigured in link-down state (`u/D`).

---

### Boundary Firewall Ruleset Construction & Forward Filter Binding

![DMZ Firewall Policy & Forward Filter Hierarchy](assets/04-dmz-firewall-ruleset.png)

* **Objective:** Construct the `DMZ_ISOLATION` IPv4 firewall ruleset to enforce zero-trust isolation against internal networks, and attach it as a forward filter on the DMZ ingress interface.
* **Key Implementation Details:**
  * **Stateful Flow Tracking (Rule 10):** Configured `action accept` for `established` and `related` connection states to ensure internal administrative connections receive return traffic.
  * **Internal Subnet Drops (Rules 20 & 30):** Configured explicit `action drop` rules for traffic destined to Management (`192.168.10.0/24`) and Application (`192.168.20.0/24`) subnets.
  * **Default Policy & Egress Flow:** Maintained `default-action accept` to permit DMZ workloads outbound internet access.
  * **Forward Filter Jump (Rule 10):** Bound inbound interface `eth1` to jump directly into the `DMZ_ISOLATION` inspection chain upon ingress.

---

