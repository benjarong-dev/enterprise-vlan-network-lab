# Enterprise Office & Server Infrastructure — VLAN Segmentation Lab

Logical network architecture and configuration lab implementing **Router-on-a-Stick (IEEE 802.1Q)**, **Stateful Inter-VLAN Firewalling (`nftables`)**, and **Omada SDN Centralized Wireless Management** with **RSTP (802.1w)** loop protection.

## Network Topology

<img src="topology/enterprise-network-topology.png" alt="Enterprise Network Topology" width="100%">


## Architecture Highlights

- **802.1Q VLAN Segmentation:** Isolates infrastructure management, virtualization/core servers, corporate staff Wi-Fi, and untrusted IoT/CCTV endpoints into four distinct broadcast domains.
- **Stateful Firewall Policy (`nftables`):** Enforces strict zone-based access control on the Linux Edge Router/Firewall. Blocks untrusted IoT/CCTV devices (VLAN 40) from initiating connections to RFC1918 internal subnets while allowing stateful return traffic for server-initiated monitoring and RTSP streams.
- **Centralized SDN Provisioning:** Places the TP-Link Omada SDN Controller (`192.168.10.5`) in the dedicated Management VLAN, provisioning L2 switch profiles and multi-SSID Enterprise APs over untagged Native VLAN 10.
- **Hypervisor Trunking:** Delivers tagged VLAN 10 (Proxmox host management) and VLAN 20 (production VMs) over a 10G SFP+ uplink (`Te0/1`) to the Linux virtual bridge (`vmbr0`).

---

## IP Addressing & Switch Port Allocation

| VLAN | Zone | Subnet (CIDR) | Gateway | Switch Port Assignment | Mode / Tagging |
|:---|:---|:---|:---|:---|:---|
| **VLAN 10** | Management | `192.168.10.0/24` | `192.168.10.1` | `Gi0/1 - Gi0/4` | Access (Untagged VLAN 10) |
| **VLAN 20** | Core Servers | `192.168.20.0/24` | `192.168.20.1` | `Te0/1`, `Gi0/5 - Gi0/12` | 802.1Q Trunk (VLAN 10, 20) |
| **VLAN 30** | Staff & Wi-Fi | `192.168.30.0/24` | `192.168.30.1` | `Gi0/13 - Gi0/20` | Trunk (Native 10, Tagged 30, 40) |
| **VLAN 40** | IoT & CCTV | `192.168.40.0/24` | `192.168.40.1` | `Gi0/21 - Gi0/23` | Access (Untagged VLAN 40) |
| **Uplink** | Edge Firewall | `802.1Q Trunk` | — | `Gi0/24 <-> FW Gi0/1` | 802.1Q Trunk (VLAN 10, 20, 30, 40) |
| **VPN** | Remote Admin | `10.8.0.0/24` | — | IPSec / SSL Tunnel | Encrypted WAN Termination |

### Static Infrastructure Inventory
- **`192.168.10.5`** — TP-Link Omada SDN Controller (Switch & AP Provisioning)
- **`192.168.10.10`** — Proxmox / KVM Hypervisor Host Management Interface
- **`192.168.10.50`** — Admin Workstation (SSH, Proxmox Web UI, Omada UI)
- **`192.168.20.10`** — Windows Server VM (Active Directory DS, DNS, DHCP)
- **`192.168.20.20`** — Ubuntu Linux Server VM (Python API, Webhooks, RTSP Ingestion)
- **`192.168.20.30`** — Monitoring Server VM (Grafana Telemetry)

---

## Inter-VLAN Security Policy Matrix

| Source Zone | Destination Zone | Protocol / Port | Action | Purpose |
|:---|:---|:---|:---|:---|
| **VLAN 10 / VPN** | All Internal VLANs | Any (SSH / HTTPS) | `ACCEPT` | Full infrastructure administration |
| **VLAN 20 (`.20.20`, `.20.30`)** | VLAN 40 (IoT/CCTV) | TCP `554`, `9100` | `ACCEPT` | Unidirectional RTSP stream & telemetry pull |
| **VLAN 30 (Staff)** | VLAN 20 (`.20.10`) | TCP/UDP `53, 88, 389, 445` | `ACCEPT` | Active Directory authentication & DNS |
| **VLAN 30 (Staff)** | VLAN 20 (`.20.20`, `.20.30`) | TCP `80, 443, 3000` | `ACCEPT` | Internal web services & Grafana dashboards |
| **VLAN 30 (Staff)** | VLAN 10, VLAN 40 | Any | `DROP` | Prevent unauthorized management or camera access |
| **VLAN 40 (IoT/CCTV)** | RFC1918 Internal Subnets | Any | `DROP + LOG` | Block lateral movement from untrusted devices |
| **VLAN 40 (IoT/CCTV)** | WAN (Internet) | UDP `123` (NTP) | `ACCEPT` | Clock synchronization only; block cloud exfiltration |

---

## Repository Structure

- `topology/` — Exported topology diagram (`.png`) and editable source file (`.drawio`).
- `configs/01-router-interfaces.cfg` — Linux 802.1Q sub-interface definitions (`eth1.10` – `eth1.40`).
- `configs/02-firewall-stateful-acl.nft` — Production-style `nftables` stateful ruleset and NAT masquerading.
- `configs/03-switch-8021q-ports.cfg` — L2 Managed Switch CLI configuration (RSTP, BPDU Guard, Access/Trunk PVIDs).
