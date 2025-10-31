# Updated Inter-DC Routing Analysis

## Topology Snapshot After Latest Changes

- **Primary access MLAG (ARSW-01/ARSW-02)** continues to operate in AS `65020` with VLAN `9` as the inter-DC transit and VLANs `120-124` and `160` stretched across the Port-Channel40 LAG. The SVIs remain active with VARP gateways and a shared virtual MAC `00:00:0c:07:ac:01`.【F:ARSW-01.cfg†L62-L119】【F:ARSW-01.cfg†L213-L266】【F:ARSW-01.cfg†L300-L348】
- **DR core MLAG (Arista-C-01/Arista-C-02)** runs AS `65001`, also carries VLANs `9` and `120-124/160`, and keeps VARP `00:00:0c:07:ac:02`. However, only VLAN `7/71` SVIs are active and the BGP neighbor statements on the MLAG peer-link still point to AS `65020` instead of the local AS, preventing a working iBGP relationship between the two DR members.【F:Arista-C-01.cfg†L20-L113】【F:Arista-C-01.cfg†L214-L263】【F:Arista-C-02.cfg†L17-L133】【F:Arista-C-02.cfg†L230-L286】
- **Huawei edge switch (HUAWEI-DC)** is now connected to the primary Port-Channel40, stretches the Nutanix VLANs, and terminates VLAN `9` with IP `192.168.9.10/24`. It forms eBGP peering (AS `65030`) with both ARSW members and gates Nutanix route advertisements behind a track object bound to the uplink state.【F:Huawei-DC.cfg†L1-L107】

## Routing State Observations

1. **Primary ↔ DR connectivity**
   - The primary pair still lacks any eBGP or OSPF adjacency towards the DR site; only the MLAG peer (10.0.0.1/2) is configured in BGP, so the sites rely purely on Layer-2 stretch without Layer-3 policy control.【F:ARSW-01.cfg†L313-L336】【F:ARSW-02.cfg†L312-L337】
   - DR core switches reference each other with `remote-as 65020` instead of `65001`, so their iBGP never establishes. The EVPN address-family also attempts to activate the local loopback instead of the peer’s loopback, leaving EVPN and any planned inter-site BGP sessions down.【F:Arista-C-01.cfg†L236-L258】【F:Arista-C-02.cfg†L250-L281】

2. **Primary ↔ Huawei edge**
   - Huawei advertises Nutanix subnets only when the uplink track is up, but the primary Aristas do not yet consume those routes. Without matching eBGP neighbors and import policies on ARSW-01/02, the automation cannot complete the failover loop.【F:Huawei-DC.cfg†L78-L107】
   - VLANs `120-124/160` on the Huawei remain L2-only (SVIs shutdown), ensuring the Arista VARP gateways stay authoritative during normal operation.【F:Huawei-DC.cfg†L59-L76】

3. **Failover automation hooks**
   - The primary site still relies on local scripts/automation to withdraw Nutanix routes; no `track` or conditional route-map is tied to the existing prefix-lists on ARSW-01/02, so outbound advertisements remain manual.【F:ARSW-01.cfg†L300-L336】【F:ARSW-02.cfg†L299-L337】

## Recommended Routing Improvements

1. **Establish eBGP full-mesh across VLAN 9**
   - On ARSW-01/02, add a DCI peer-group (remote AS `65001`) pointing at `192.168.9.5/6` with BFD and reuse the `LIVE_LAN` prefix-list for outbound filtering. Tie route-map activation to the existing gateway automation so Nutanix subnets are advertised only when the primary site is active.【F:ARSW-01.cfg†L300-L336】
   - On Arista-C-01/02, correct the MLAG peer BGP `remote-as` to `65001`, fix EVPN neighbor activation to target the opposite loopback, and create an eBGP peer-group towards the primary (`192.168.9.2/3`). Apply route-maps that insert a lower local preference when learning stretched VLANs from the primary, allowing deterministic failback.【F:Arista-C-01.cfg†L236-L258】【F:Arista-C-02.cfg†L250-L281】
   - Ensure Huawei peers (`192.168.9.10`) are added to the primary peer-group so the automation-protected advertisements can be consumed. If the Huawei device should also see DR routes, add a guarded neighbor on the DR pair with strict prefix-lists to avoid accidental transit.

2. **Integrate track objects with BGP policy**
   - Extend the primary automation so that when SVI shutdown is triggered (failover to DR), the `LIVE_LAN` prefix-list removes Nutanix entries. Conversely, when Nutanix SVIs come up on DR, inject the same subnets into a `DR-LIVE` prefix-list before enabling the DR eBGP peer-group, mirroring the Huawei approach.【F:ARSW-01.cfg†L300-L336】【F:Huawei-DC.cfg†L95-L107】
   - On Huawei, keep the `EXPORT-NUTANIX` route-policy tied to `track 1` and add syslog/SNMP notifications for faster visibility of state transitions.【F:Huawei-DC.cfg†L93-L107】

3. **Cleanup EVPN and gateway symmetry on the DR site**
   - Activate SVIs (even in shutdown state) for VLANs `120-124/160` on both Arista-C switches so that VARP configuration is symmetric and ready for orchestration. Use automation to toggle `shutdown` during DR activation to align with gateway state. Right now only VLAN120 has a virtual IP on Arista-C-02 and none on Arista-C-01.【F:Arista-C-01.cfg†L222-L244】【F:Arista-C-02.cfg†L230-L262】
   - Once EVPN neighbors reference the opposite loopback (`neighbor 10.1.1.2 activate` on C-01, `neighbor 10.1.1.1 activate` on C-02), consider enabling Type-5 route exchange for future VXLAN expansion, even if VXLAN is currently disabled, so the configuration stays future-proof.【F:Arista-C-01.cfg†L248-L258】【F:Arista-C-02.cfg†L268-L281】

4. **Operational safeguards**
   - Disable Telnet on all platforms and rely on SSH/NETCONF/eAPI for management to reduce attack surface.【F:ARSW-01.cfg†L341-L345】【F:ARSW-02.cfg†L338-L342】【F:Arista-C-01.cfg†L260-L263】【F:Arista-C-02.cfg†L282-L285】
   - Standardize MSTP root roles: keep the primary Aristas as root primary, configure DR as root secondary, and set the Huawei edge to guard against root role to prevent unintended STP changes when failover occurs.【F:ARSW-01.cfg†L31-L35】【F:Arista-C-01.cfg†L28-L32】【F:Huawei-DC.cfg†L18-L41】

Implementing these changes delivers deterministic routing control between the primary and DR sites while leveraging the Huawei edge as an automation-aware participant for Nutanix VLAN advertisements.
