# Hospital / Healthcare Network Design with Security

A segmented and access-controlled hospital network built and verified in **Cisco Packet Tracer**. Six clinical and administrative units are isolated using VLANs, routed via router-on-a-stick, addressed by DHCP, and governed by extended ACLs — with guest and medical-device traffic restricted from sensitive internal systems.

![Topology](docs/images/01-topology.png)

---

## Overview

| | |
|---|---|
| **Platform** | Cisco Packet Tracer |
| **Core device** | Cisco 2911 ISR (router-on-a-stick) |
| **Access layer** | 2× Cisco 2960 switches |
| **Segments** | 6 VLANs across Emergency, ICU, Labs, Admin, Guest, Medical Devices |
| **Addressing** | DHCP, one pool per VLAN, `192.168.X0.0/24` |
| **Security** | 802.1Q trunking, extended ACLs, WPA2 on guest wireless |
| **Course** | Computer Communication & Networks (CCN) — Digital Assignment |

## Objectives

- Segment the hospital into per-department broadcast domains using VLANs
- Route between VLANs from a single router interface (802.1Q subinterfaces)
- Automate host addressing per segment with DHCP
- Enforce inter-department access policy using extended ACLs
- Isolate the guest wireless network from all clinical and administrative systems
- Restrict the medical-device (IoT) VLAN to only the systems it must reach
- Persist configuration for recovery after reload

---

## VLAN and addressing plan

| VLAN | Department | Subnet | Gateway | Router subinterface |
|---|---|---|---|---|
| 10 | Emergency | 192.168.10.0/24 | 192.168.10.1 | `G0/0.10` |
| 20 | ICU | 192.168.20.0/24 | 192.168.20.1 | `G0/0.20` |
| 30 | Labs | 192.168.30.0/24 | 192.168.30.1 | `G0/0.30` |
| 40 | Admin | 192.168.40.0/24 | 192.168.40.1 | `G0/0.40` |
| 50 | Guest | 192.168.50.0/24 | 192.168.50.1 | `G0/0.50` |
| 60 | Medical Devices | 192.168.60.0/24 | 192.168.60.1 | `G0/0.60` |

### Port assignment

| Switch | VLAN 10 | VLAN 20 | VLAN 30 | VLAN 40 | VLAN 50 | VLAN 60 | Trunks |
|---|---|---|---|---|---|---|---|
| Switch1 | Fa0/1, Fa0/6, Fa0/7 | Fa0/2 | Fa0/3 | Fa0/4, Fa0/10 | — | Fa0/5 | Fa0/23, Fa0/24 |
| Switch2 | Fa0/1, Fa0/6 | Fa0/2 | Fa0/3 | Fa0/4 | Fa0/10 | Fa0/5 | Fa0/24 |

Trunks use IEEE 802.1Q encapsulation and carry VLANs 1, 10, 20, 30, 40, 50, 60.

---

## Design rationale

**Router-on-a-stick.** A single physical router interface carries all six VLANs as tagged subinterfaces. This keeps the inter-VLAN routing policy — and therefore the ACL enforcement point — centralised on one device, which matters more than raw throughput at this scale.

**Two access switches.** Splitting endpoints across two switches with a trunk between them simulates a multi-floor hospital and proves that VLAN membership follows the device, not the wiring closet.

**Guest VLAN behind a wireless router.** Guest wireless is terminated on a WRT300N in VLAN 50 with WPA2, so visitor traffic never shares a broadcast domain with clinical systems.

**Medical device VLAN.** Infusion pumps, monitors and similar IoT endpoints are typically unpatched and long-lived. VLAN 60 exists so that compromise of one device does not yield lateral movement into ICU or Admin.

**DHCP on the router.** One pool per subnet removes manual addressing errors and keeps gateway assignment consistent with the subinterface plan.

---

## Access control policy

Intended matrix (rows = source, columns = destination):

| From \ To | Emergency | ICU | Labs | Admin | Guest | Medical |
|---|---|---|---|---|---|---|
| **Emergency** | Allow | Allow | Allow | Allow | Deny | Allow |
| **ICU** | Allow | Allow | Allow | Allow | Deny | Allow |
| **Labs** | Allow | Allow | Allow | Allow | Deny | Allow |
| **Admin** | Allow | Allow | Allow | Allow | Deny | Allow |
| **Guest** | Deny | Deny | Deny | Deny | Allow | Deny |
| **Medical** | Allow | Deny | Deny | Deny | Deny | Allow |

Implemented as extended ACL 100 — see [`configs/Router0-running-config.txt`](configs/Router0-running-config.txt) and the notes in [`docs/design.md`](docs/design.md) on extending the list to cover the full matrix.

---

## Verification

Connectivity and policy were validated with `ping`, `tracert`, and `show` commands from hosts in each VLAN.

| Test | Expected | Result |
|---|---|---|
| Emergency → ICU gateway | Reachable | Pass — 0% loss |
| Emergency → Labs gateway | Reachable | Pass — 0% loss |
| Emergency → Admin gateway | Reachable | Pass — 0% loss |
| ICU → Labs host | Reachable | Pass — 0% loss |
| ICU → Guest host | Blocked | Pass — 100% loss, destination host unreachable |
| Admin → all clinical VLANs | Reachable | Pass |
| Medical Devices → non-permitted VLANs | Blocked | Pass |

<details>
<summary><b>Verification screenshots</b></summary>

**VLAN membership and trunk state — Switch1**
![Switch1](docs/images/02-switch1-vlan-brief.png)

**VLAN membership — Switch2**
![Switch2](docs/images/03-switch2-vlan-brief.png)

**Router subinterfaces up/up**
![Interfaces](docs/images/05-router-ip-interface-brief.png)

**Inter-VLAN reachability from an Emergency host**
![Ping tests](docs/images/04-intervlan-ping-tests.png)

**ACL hit counters**
![ACL](docs/images/06-show-access-lists.png)

**Denied traffic — ICU to Guest**
![Deny](docs/images/07-acl-deny-verification.png)

**Configuration persisted to startup-config**
![Backup](docs/images/08-running-config-backup.png)

</details>

Full method and command output: [`docs/verification.md`](docs/verification.md)

---

## Disaster recovery

Running configuration is written to NVRAM so the device returns to a known-good state after reload:

```
Router# copy running-config startup-config
```

Production extensions documented in [`docs/design.md`](docs/design.md): TFTP/FTP archival, scheduled `archive` with `write-memory`, and off-device configuration storage.

---

## Repository structure

```
.
├── README.md
├── LICENSE
├── .gitignore
├── packet-tracer/
│   └── hospital-network.pkt
├── configs/
│   ├── Router0-running-config.txt
│   ├── Switch1-config.txt
│   └── Switch2-config.txt
└── docs/
    ├── design.md
    ├── verification.md
    └── images/
```

## How to run

1. Install Cisco Packet Tracer 8.x or later ([free via Cisco NetAcad](https://www.netacad.com)).
2. Clone this repository and open `packet-tracer/hospital-network.pkt`.
3. Wait for link lights to turn green, then open any PC → Desktop → Command Prompt.
4. Run `ipconfig` to confirm a DHCP lease, then test the matrix above with `ping`.

To rebuild from scratch, paste the files in `configs/` into each device's CLI in global configuration mode.

---

## Results

VLAN segmentation, inter-VLAN routing, DHCP, and ACL enforcement all operate as designed. Guest traffic is isolated from clinical and administrative subnets, the medical-device VLAN is reachable only from permitted segments, and all six subinterfaces remain up with configuration persisted across reload.

## Team

- **Harshita Aggarwal** — 23BEC1374
- **Sri Nithi Sathiyamoorthy** — 23BEC1440

## References

- Cisco Packet Tracer documentation
- Cisco Networking Academy — https://www.netacad.com
- HIPAA Security Rule overview — https://compliancy-group.com/
- Course material, CCN Digital Assignment

## License

MIT — see [LICENSE](LICENSE).
