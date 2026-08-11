# Intelligent Campus Network Automation

**EE8203 – Design and Management of Data Networks**
Faculty of Engineering, University of Ruhuna — Department of Computer Engineering

Design, implementation, automation and monitoring of a hierarchical campus network for the Faculty of Engineering, built in **GNS3** (Cisco IOSv / IOSvL2), automated with **Netmiko** and **Ansible**, and monitored with **Zabbix 7.0 LTS**.

## Overview

A three-tier (core / distribution / access) campus network connecting four departments — **DEIE, DCEE, DMME, DIS** — with:

- **OSPF** (single Area 0) dynamic routing; default route originated at the edge
- **NAT overload (PAT)** with ACL-restricted Internet egress (DEIE + DCEE only)
- **Extended named ACLs** enforcing a 4×4 inter-department access policy
- **Out-of-band management** on VLAN 99 for SSH / SNMP / automation
- **Netmiko** for router configuration, **Ansible (cisco.ios)** for switch configuration
- **Zabbix 7.0 LTS** SNMPv2c monitoring with triggers and a custom dashboard

---

## Topology

![Topology](docs/topology.png)

| Layer | Devices | Role |
|---|---|---|
| Core | R-EDGE, R-CORE, SW-CORE | WAN/NAT, OSPF backbone, inter-VLAN routing, DIS gateway |
| Distribution | SW-D-DEIE, SW-D-DCEE, SW-D-DMME | Per-department SVI gateways + ACL enforcement |
| Access | SW-A-DEIE, SW-A-DCEE, SW-A-DMME, SW-A-DIS | L2 access ports, 802.1Q trunks |
| Services | VM-AUTO, VM-ZABBIX, VM-DHCP/WEB, 8× VPC | Automation host, monitoring, web/reachability tests, end hosts |

---

## Addressing Plan

| VLAN | Department | Subnet | Gateway |
|---|---|---|---|
| 10 | DEIE | 10.10.10.0/24 | 10.10.10.1 (SW-D-DEIE) |
| 20 | DCEE | 10.10.20.0/24 | 10.10.20.1 (SW-D-DCEE) |
| 30 | DMME | 10.10.30.0/24 | 10.10.30.1 (SW-D-DMME) |
| 40 | DIS | 10.10.40.0/24 | 10.10.40.1 (SW-CORE) |
| 99 | Management | 10.99.99.0/24 | 10.99.99.1 (SW-CORE) |
| 100 | Native (trunks) | — | VLAN-hopping mitigation |

Management IPs: switches `.1/.11–.13/.21–.24`, `VM-AUTO .50`, `VM-ZABBIX .51`.
Transit links: `192.168.0.0/30`, `192.168.1.0/30`, `10.255.1.0/30`, `10.255.1.4/30`, `10.255.1.8/30`; loopbacks `1.1.1.1/2.2.2.2/3.3.3.3` as OSPF router IDs.

---

## ACL Policy (verified 4×4 matrix)

| Source ↓ / Dest → | DEIE | DCEE | DMME | DIS |
|---|---|---|---|---|
| **DEIE** | – | DENY | DENY | **PERMIT all** |
| **DCEE** | DENY | – | DENY | HTTP/HTTPS only* |
| **DMME** | DENY | DENY | – | DENY |
| **DIS** | PERMIT | DENY | DENY | – |

\* ICMP is denied by design; TCP 80/443 verified separately.
Management plane: `LAB_SECURITY_POLICY` on SW-CORE restricts **SSH to VLAN 99** and **SNMP to the Zabbix host** only.

---

## Repository Structure

```
.
├── netmiko/
│   ├── automate.py            # backup-first baseline push (entry point)
│   ├── router_config.py       # router IPs / OSPF / NAT / ACL rollout
│   ├── snmp_push.py           # fleet-wide SNMPv2c + trap rollout
│   ├── inventory.json         # device inventory (no hardcoding)
│   ├── devices.json
│   └── routers.json           # router command sets as data
├── ansible/
│   ├── ansible.cfg
│   ├── inventory.ini
│   ├── site.yml               # orchestration: vlans → trunking → access_ports → stp
│   ├── group_vars/all.yml     # connection + credentials
│   ├── host_vars/*.yml        # per-switch design facts
│   ├── roles/{vlans,trunking,access_ports,stp}/tasks/main.yml
│   └── playbooks/rollback/rollback.yml
├── zabbix/
│   ├── export_dashboard.py    # JSON-RPC API dashboard export
│   └── EE8203_4520_4465_ZabbixDashboard.json
└── docs/                      # report, screenshots, MOP
```

---

## Getting Started

**Prerequisites:** GNS3 with IOSv/IOSvL2 images, VMware (VM-AUTO / VM-ZABBIX), Python 3.10+, `ansible-core` + `cisco.ios` collection.

```bash
python3 -m venv venv && source venv/bin/activate
pip install netmiko ansible-core
ansible-galaxy collection install cisco.ios

# Router automation (idempotent, safe to re-run)
python3 netmiko/router_config.py
python3 netmiko/snmp_push.py          # 10/10 [OK]

# Switch automation
cd ansible
ansible-playbook site.yml
ansible-playbook site.yml --check     # idempotency proof → changed=0
```

Idempotency evidence:

```
PLAY RECAP ********************************************************************
SW-A-DCEE : ok=5  changed=2  unreachable=0  failed=0  skipped=2   # first run
SW-A-DCEE : ok=5  changed=0  unreachable=0  failed=0  skipped=2   # --check re-run
```

Rollback a device to its saved baseline in under 5 minutes:

```bash
ansible-playbook playbooks/rollback/rollback.yml -e target=SW-D-DCEE
```

---

## Monitoring (Zabbix 7.0 LTS)

- SNMPv2c community `ZabbixLab` pushed fleet-wide by `snmp_push.py` (injected via the `{$SNMP_COMMUNITY}` macro)
- Hosts onboarded with the **Cisco IOS by SNMP** template (interface / CPU / memory items, unreachable / interface-down / high-CPU triggers)
- Custom **FoE-UoR Network** dashboard (host availability, interface traffic, problems by severity)
- Dashboard exported via the **Zabbix JSON-RPC API** (`user.login` → `dashboard.get`) — the web UI offers no dashboard export

---

## Lessons Learned

- An ACL filters what a VLAN **sends**, not what it receives — inbound SVI placement explains the asymmetric matrix results
- ICMP failure ≠ ACL failure (HTTP/HTTPS-only policies need `curl`-based verification)
- Manual-first, then automate: scripts should encode a design already proven on the CLI
- Never trust interface names across reboots — pin NICs by MAC in netplan
- Keep config backups **outside** the VM they protect

---

## Disclaimer

University group project for EE8203 (2026). Provided for reference and learning; see `docs/` for the full technical report.
