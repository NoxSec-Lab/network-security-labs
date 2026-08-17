Lab 2: Enterprise Multi-Tier Architecture & DMZ Isolation
Overview
This lab expands our baseline enterprise architecture into a production-grade 3-tier network topology. By introducing a Demilitarized Zone (DMZ) alongside an Internal LAN and a dedicated Management VLAN, this project enforces strict ingress/egress filtering rules, isolates public-facing web services, and sets up a controlled target environment for multi-stage penetration testing and pivoting exercises.


Core Security Objectives & ACL Matrix
To enforce zero-trust isolation on the VyOS core router, we define four strict firewall traffic flows:
DMZ Isolation Rule: Hosts in the DMZ (172.16.30.0/24) MUST NOT initiate any connection into the Internal LAN (192.168.20.0/24) or Management Zone (192.168.10.0/24).
DMZ Service Access: Management and Internal LAN hosts can access HTTP/HTTPS (TCP 80/443) and SSH (TCP 22) on DMZ hosts.
Stateful Ingress Processing: Return traffic from the DMZ is permitted only if the connection was initiated by internal hosts (established, related).
Edge Gateway Routing: Downstream static routes on pfSense must be updated to route return traffic for 172.16.30.0/24 back to next-hop 10.0.0.2.
