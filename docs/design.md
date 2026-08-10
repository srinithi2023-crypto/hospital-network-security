# Design Notes

## 1. Why segmentation first

A hospital network carries three classes of traffic with incompatible risk profiles: clinical systems handling patient data, unmanaged medical IoT devices, and untrusted visitor devices. Running these in one broadcast domain means a single compromised guest laptop can ARP-spoof its way into an ICU workstation. VLANs give each class its own domain and force all cross-class traffic through a routed hop where policy can be applied.

## 2. Router-on-a-stick

All six VLANs are trunked to one physical router interface (`G0/0`) and terminated on 802.1Q subinterfaces.

**Trade-off.** Every inter-VLAN packet crosses the same physical link twice, so the uplink is a bandwidth ceiling and a single point of failure. In a real deployment this would be a layer-3 switch with SVIs, and the uplink would be an EtherChannel or a redundant pair with HSRP.

**Why it is right here.** With one routing device, the ACL enforcement point is unambiguous and the whole policy is auditable from one `show running-config`.

## 3. Access layer

Two 2960 switches connected by an 802.1Q trunk. Endpoints for the same department exist on both switches, which demonstrates that VLAN membership is a logical property, not a physical one — an Emergency PC on Switch2 lands in the same subnet as one on Switch1.

Trunks explicitly allow VLANs 1, 10, 20, 30, 40, 50, 60. Pruning unused VLANs from trunks limits the blast radius of a VLAN-hopping attempt.

## 4. Addressing

One `/24` per VLAN, numbered so that the third octet matches the VLAN ID (VLAN 10 → 192.168.10.0/24). The gateway is always `.1` and equals the router subinterface address.

DHCP runs on the router with a pool per subnet. Addresses `.1` through `.10` are excluded from each pool so servers and infrastructure can be statically addressed without collisions.

## 5. Access control

The verified deployment applies extended ACL 100 inbound on the ICU subinterface:

```
access-list 100 deny ip 192.168.20.0 0.0.0.255 192.168.50.0 0.0.0.255
access-list 100 permit ip any any
```

Hit counters in `show access-lists` confirm both matches and denials.

### Known gap

The access matrix describes a broader policy than ACL 100 alone implements:

- **Guest isolation is one-directional.** ACL 100 blocks ICU → Guest. It does not block Guest → ICU or Guest → any other internal VLAN. Because the deny is stateless, return traffic for a guest-initiated flow is not automatically dropped.
- **Medical device egress is unrestricted.** The matrix specifies Medical → ICU/Labs/Admin/Guest as denied; no ACL currently enforces this.

Both gaps are addressed by the `GUEST_ISOLATION` and `MEDICAL_EGRESS` lists included (commented) in `configs/Router0-running-config.txt`. Applying them inbound on `G0/0.50` and `G0/0.60` respectively brings the running configuration in line with the documented matrix.

### ACL placement principle

Extended ACLs are applied as close to the source as possible so denied traffic is dropped before it consumes router resources or crosses the trunk. Each list ends with an explicit `permit ip any any` because the implicit deny at the end of every ACL would otherwise black-hole all remaining traffic on that interface.

## 6. Wireless

Guest wireless terminates on a WRT300N in VLAN 50 using WPA2-PSK. WPA2 protects the air interface; VLAN 50 plus the guest ACL protects the wired side. Neither alone is sufficient.

## 7. Disaster recovery

```
Router# copy running-config startup-config
```

On reload, `startup-config` is copied into `running-config` and the network resumes without manual intervention.

| Method | Command | Benefit |
|---|---|---|
| TFTP archival | `copy running-config tftp:` | Off-device copy survives hardware failure |
| Scheduled archive | `archive` → `path`, `write-memory` | Automatic versioned snapshots on every write |
| Cloud/config-management | Ansible, RANCID, Oxidized | Diffable history, change audit trail |

## 8. What a production version would add

- Layer-3 switching with SVIs and HSRP for gateway redundancy
- Spanning-tree hardening: PortFast plus BPDU Guard on all access ports
- Port security limiting MAC addresses per access port
- DHCP snooping and Dynamic ARP Inspection, particularly on the guest and medical VLANs
- A dedicated management VLAN with SSH-only access, replacing the current `line vty` password login
- Syslog and SNMP to a central collector for the audit trail HIPAA-style compliance expects
- Stateful firewalling (ZBF or a dedicated appliance) instead of stateless ACLs
