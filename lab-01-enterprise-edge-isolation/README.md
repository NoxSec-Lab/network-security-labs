# Lab 1: Enterprise Edge Isolation & Firewall Enforcement

## Overview
This lab implements an enterprise edge network architecture utilizing a **pfSense Edge Firewall**, a **VyOS Core Router**, and dual isolated **Ubuntu Linux subnets**. The project focuses on core networking fundamentals including inter-VLAN routing, Path MTU Discovery (PMTUD) troubleshooting, static route propagation, and protocol header analysis.

## Network Architecture & Topology

![Lab Topology](assets/01-gns3-topology.png)

* **Objective:** Design a secure, multi-zone enterprise network that isolates administrative management systems from core application workloads while maintaining monitored internet connectivity.
* **Architecture Breakdown:**
  * **Edge Defense Layer:** A pfSense perimeter firewall managing external connectivity (WAN) and performing Network Address Translation (NAT) to protect internal hosts.
  * **Core Routing Layer:** A VyOS core router operating at Layer 3 to handle inter-VLAN routing and enforce traffic boundaries between distinct network segments.
  * **Isolated Internal Subnets:**
    * **Management Zone (VLAN 10):** Dedicated network segment (`192.168.10.0/24`) for administrative operations.
    * **Application Zone (VLAN 20):** Isolated network segment (`192.168.20.0/24`) for application workloads, configured with a constrained **1400 MTU** to analyze Path MTU Discovery (PMTUD) behavior.
