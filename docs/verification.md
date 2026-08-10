# Verification

Method and recorded output for validating the topology, routing, addressing, and access policy.

---

## 1. VLAN membership

**Command:** `show vlan brief` on each switch

![Switch1 VLAN brief](images/02-switch1-vlan-brief.png)

![Switch2 VLAN brief](images/03-switch2-vlan-brief.png)

**Result:** All six VLANs active on both switches. Port assignments match the plan in the README.

---

## 2. Trunk state

**Command:** `show interfaces trunk` on Switch1

| Port | Mode | Encapsulation | Status | Native VLAN | Allowed & active |
|---|---|---|---|---|---|
| Fa0/23 | on | 802.1q | trunking | 1 | 1,10,20,30,40,50,60 |
| Fa0/24 | on | 802.1q | trunking | 1 | 1,10,20,30,40,50,60 |

**Result:** Both trunks up, all department VLANs forwarding and not pruned by spanning tree.

---

## 3. Router subinterfaces

**Command:** `show ip interface brief`

![Router interfaces](images/05-router-ip-interface-brief.png)

| Interface | IP address | Status | Protocol |
|---|---|---|---|
| G0/0 | unassigned | up | up |
| G0/0.10 | 192.168.10.1 | up | up |
| G0/0.20 | 192.168.20.1 | up | up |
| G0/0.30 | 192.168.30.1 | up | up |
| G0/0.40 | 192.168.40.1 | up | up |
| G0/0.50 | 192.168.50.1 | up | up |
| G0/0.60 | 192.168.60.1 | up | up |

**Result:** All six gateways up/up. `G0/1` and `G0/2` remain administratively down as intended.

---

## 4. DHCP

**Command:** `ipconfig` on a host in each VLAN

**Result:** Each host receives an address from its own subnet with the correct default gateway, confirming the per-VLAN pool binding.

---

## 5. Inter-VLAN reachability

**Command:** `ping` from an Emergency host to each gateway

![Ping tests](images/04-intervlan-ping-tests.png)

| Target | Sent | Received | Loss | Result |
|---|---|---|---|---|
| 192.168.20.1 (ICU) | 4 | 4 | 0% | Pass |
| 192.168.30.1 (Labs) | 4 | 4 | 0% | Pass |
| 192.168.40.1 (Admin) | 4 | 4 | 0% | Pass |

---

## 6. ACL enforcement

**Command:** `ping` from an ICU host, then `show access-lists` on the router

![Deny verification](images/07-acl-deny-verification.png)

| Source | Target | Expected | Sent | Received | Loss |
|---|---|---|---|---|---|
| ICU host | 192.168.30.2 (Labs) | Permit | 4 | 4 | 0% |
| ICU host | 192.168.50.2 (Guest) | Deny | 4 | 0 | 100% |

The denied attempt returns *Destination host unreachable* from 192.168.20.1 — the router itself, confirming the drop occurs at the ACL rather than from a missing route.

![ACL counters](images/06-show-access-lists.png)

```
Extended IP access list 100
    10 deny ip 192.168.20.0 0.0.0.255 192.168.50.0 0.0.0.255 (4 match(es))
    20 permit ip any any (6 match(es))
```

---

## 7. Configuration persistence

**Command:** `copy running-config startup-config`

![Running config and backup](images/08-running-config-backup.png)

**Result:** Configuration written to NVRAM, `[OK]` returned.

---

## Summary

| Area | Status |
|---|---|
| VLAN segmentation | Verified |
| 802.1Q trunking | Verified |
| Inter-VLAN routing | Verified |
| DHCP addressing | Verified |
| ACL enforcement (ICU → Guest) | Verified |
| Full access matrix | Partially enforced — see `design.md` § Known gap |
| Configuration backup | Verified |
