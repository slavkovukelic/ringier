# Inter-DC Routing Design Proposal

## Current Topology Snapshot

- **Primary access MLAG pair (ARSW-01/ARSW-02)** operates in AS **65020** with SVI gateway services for VLANs `120-123`, `160`, and the transit **VLAN 9** using addresses `192.168.9.2/24` and `192.168.9.3/24` and a shared VARP of `192.168.9.1`.【F:ARSW-01.cfg†L118-L162】【F:ARSW-02.cfg†L228-L247】
- **DR core MLAG pair (Arista-C-01/Arista-C-02)** operates in AS **65001** with the same transit VLAN using `192.168.9.5/24` and `192.168.9.6/24` and VARP `192.168.9.4`.【F:Arista-C-01.cfg†L208-L222】【F:Arista-C-02.cfg†L226-L244】
- The inter-DC Port-Channel40 is already carrying VLANs `9` and `120-124,160`, providing a 50 Gb/s Layer-2 stretch between the sites.【F:ARSW-01.cfg†L62-L73】【F:Arista-C-01.cfg†L35-L63】

## Routing Objectives

1. Maintain Layer-2 stretch for Nutanix replication VLANs (`120-124`, `160`) while preventing north-south routing loops.
2. Provide deterministic Layer-3 reachability between sites for local-only VLANs and upstream connectivity advertisements.
3. Support automatic failover so that the DR site can advertise stretched VLAN subnets only when its gateway SVIs are brought online.

## Recommended DCI Routing Model

### 1. Dedicated Inter-DC eBGP Adjacency over VLAN 9

- Keep VLAN 9 as the routed transit between sites and form an **eBGP session between AS 65020 (Primary) and AS 65001 (DR)** using the physical SVI addresses.
- Configure both members of each MLAG pair to peer with both members on the opposite site (full mesh of four eBGP sessions) to avoid single-box dependencies.
- Use BFD for BGP (`neighbor x.x.x.x bfd`) to converge with ~1 s detection on the 50 Gb/s link.
- Example configuration on ARSW side:
  ```eos
  router bgp 65020
     neighbor DCI-DR peer group
     neighbor DCI-DR remote-as 65001
     neighbor DCI-DR password <optional>
     neighbor DCI-DR timers 5 15
     neighbor DCI-DR bfd
     neighbor 192.168.9.5 peer group DCI-DR
     neighbor 192.168.9.6 peer group DCI-DR
     address-family ipv4
        neighbor DCI-DR activate
        neighbor DCI-DR route-map PRIMARY-EXPORT out
  ```
  Mirror the configuration on Arista-C pair with their own peer-group pointing to `192.168.9.2/3`.

### 2. Prefix Control and Advertisements

- Reuse the existing prefix-lists on the primary site (e.g., `LIVE_LAN`) to limit exported networks and add any other primary-only VLANs that must be reachable from DR.
- On the DR site define a dedicated prefix-list (e.g., `DR-LIVE`) that initially stays empty or only contains infrastructure routes (loopbacks, management). When the automated gateway activation script removes `shutdown` on VLANs `120-124/160`, the same automation can append those subnets to the prefix-list before pushing `router bgp` updates.
- Apply import route-maps on both sides to reject unexpected subnets and optionally set a lower local preference for routes learned from the opposite site unless in failover mode.

### 3. Default Routing Considerations

- Keep the existing static default towards `172.27.85.1` on the DR core but advertise it into eBGP **only when** DR takes over active services. Use a conditional route-map that matches the presence of a track object tied to the gateway automation (e.g., `neighbor DCI-PRIMARY route-map DR-DEFAULT out`).
- If the primary site should always export a default towards DR (for mgmt/backup traffic), configure a separate route-map that advertises only `0.0.0.0/0` from ARSW once the upstream northbound path is confirmed up (track object on `Ethernet48` or OSPF neighbor state).

### 4. Fast Failover Hooks

- Tie the eBGP advertisement policy to the same health-check logic that controls SVI shutdown on the DR pair. The automation can:
  1. Enable the SVI interfaces.
  2. Update BGP prefix-lists to add the Nutanix VLAN subnets.
  3. Clear the BGP session (or rely on BFD) so the primary side immediately installs DR routes.
- On the primary site, configure `neighbor DCI-DR fall-over bfd` so loss of BFD session immediately withdraws routes, ensuring stretched VLAN traffic is attracted to the active site only.

### 5. Optional OSPF Overlay (If Dynamic Metric Needed)

- If dynamic metric adjustments are preferred, run an additional **OSPF area** across VLAN 9 (e.g., Area 20) between the two sites while keeping BGP as the policy edge. Use OSPF solely to exchange loopbacks and infrastructure routes, and continue to control customer VLAN announcements via BGP to prevent accidental flooding of stretched subnets.
- Ensure `passive-interface default` stays in place and only VLAN 9 is marked `no passive-interface` to maintain deterministic adjacency.

## Operational Checklist

1. **Document ASNs & peer IPs** – 65020 ↔ 65001 over 192.168.9.0/24.
2. **Deploy peer-groups with BFD** – reduces CLI footprint and gives sub-second failure detection.
3. **Implement automation hooks** – integrate BGP route-map updates with the existing gateway activation scripts to keep advertisements consistent with active gateways.
4. **Monitoring** – add SNMP/syslog alerts for BGP neighbor state changes and track BFD events for the DCI sessions.
5. **Testing** – simulate Port-Channel40 failure and full primary-site outage to confirm route withdrawal and DR takeover happen inside the required SLA.

This design keeps the Nutanix VLANs in Layer-2 stretch mode while using policy-driven eBGP to control which site advertises them northbound, allowing deterministic failover without introducing VXLAN.
