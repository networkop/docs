---
title: Multi-Chassis Link Aggregation - MLAG
author: NVIDIA
weight: 490
toc: 3
---
{{%notice note%}}
**MLAG or CLAG**: The Cumulus Linux implementation of MLAG is referred to by other vendors as CLAG, MC-LAG or VPC. You even see references to CLAG in Cumulus Linux, including the management daemon, named `clagd`, and other options in the code, such as `clag-id`, which exist for historical purposes. The Cumulus Linux implementation is truly a multi-chassis link aggregation protocol, so we call it MLAG.
{{%/notice%}}

Multi-Chassis Link Aggregation (MLAG) enables a server or switch with a two-port bond, such as a link aggregation group (LAG), EtherChannel, port group or trunk, to connect those ports to different switches and operate as if they are connected to a single, logical switch. This provides greater redundancy and greater system throughput.

Dual-connected devices can create LACP bonds that contain links to each physical switch; active-active links from the dual-connected devices are supported even though they are connected to two different physical switches.

## How Does MLAG Work?

A basic MLAG configuration looks like this:

| | |
|---|---|
|<div style="width:400px">{{< img src="/images/cumulus-linux/mlag-basic-setup.png" width="400" >}}|<br> <ul><li>The two switches, leaf01 and leaf02, known as *peer switches*, appear as a single device to the bond on server01.</li><li>server01 distributes traffic between the two links to leaf01 and leaf02 in the way you configure on the host. </li><li>Traffic inbound to server01 can traverse leaf01 or leaf02 and arrive at server01.</li></ul>|

More elaborate configurations are also possible. The number of links between the host and the switches can be greater than two and does not have to be symmetrical. Additionally, because the two peer switches appear as a single switch to other bonding devices, you can also connect pairs of MLAG switches to each other in a switch-to-switch MLAG configuration:

| | |
|---|---|
|<div style="width:400px">{{< img src="/images/cumulus-linux/mlag-two-pair.png" width="350" >}} |<br><br> <ul><li>leaf01 and leaf02 are also MLAG peer switches and present a two-port bond from a single logical system to spine01 and spine02.</li><li>spine01 and spine02 do the same as far as leaf01 and leaf02 are concerned.</li></ul>|

### LACP and Dual-connected Links

{{<link url="Bonding-Link-Aggregation" text="Link Aggregation Control Protocol (LACP)">}}, the IEEE standard protocol for managing bonds, is used for verifying dual-connectedness. LACP runs on the dual-connected devices and on each of the MLAG peer switches. On a dual-connected device, the only configuration requirement is to create a bond that is managed by LACP.

On each of the peer switches, you must place the links that are connected to the dual-connected host or switch in the bond. This is true even if the links are a single port on each peer switch, where each port is placed into a bond, as shown below:

{{< img src = "/images/cumulus-linux/mlag-dual-connect.png" >}}

All of the dual-connected bonds on the peer switches have their system ID set to the MLAG system ID. Therefore, from the point of view of the hosts, each of the links in its bond is connected to the same system and so the host uses both links.

Each peer switch periodically makes a list of the LACP partner MAC addresses for all of their bonds and sends that list to its peer (using the `clagd` service). The LACP partner MAC address is the MAC address of the system at the other end of a bond (server01, server02, and server03 in the figure above). When a switch receives this list from its peer, it compares the list to the LACP partner MAC addresses on its switch. If any matches are found and the `clag-id` for those bonds match, then that bond is a dual-connected bond. You can find the LACP partner MAC address by the running `net show bridge macs` command.

### Requirements

MLAG has these requirements:

- There must be a direct connection between the two peer switches configured with MLAG. This is typically a bond for increased reliability and bandwidth.
- There must be only two peer switches in one MLAG configuration, but you can have multiple configurations in a network for *switch-to-switch MLAG*.
- Both switches in the MLAG pair must be running the same release of Cumulus Linux. See {{<link url="Upgrading-Cumulus-Linux#upgrade-switches-in-an-mlag-pair" text="Upgrading Cumulus Linux">}}.

{{%notice note%}}
- MLAG is *not* supported in a multiple VLAN-aware bridge configuration.
- Both MLAG peers must use the same {{<link url="VXLAN-Devices" text="VXLAN device type">}} (single or traditional).
{{%/notice%}}

## Basic Configuration

To configure MLAG, you need to create a bond that uses LACP on the dual-connected devices and configure the interfaces (including bonds, VLANs, bridges, and peer links) on each peer switch.

Follow these steps on each peer switch in the MLAG pair:

1. On the dual-connected device, such as a host or server that sends traffic to and from the switch, create a bond that uses LACP. The method you use varies with the type of device you are configuring.

   {{%notice note%}}
If you cannot use LACP in your environment, you can configure the bonds in {{<link url="Bonding-Link-Aggregation" text="balance-xor mode">}}.
{{%/notice%}}

2. Place every interface that connects to the MLAG pair from a dual-connected device into a {{<link url="Bonding-Link-Aggregation" text="bond">}}, even if the bond contains only a single link on a single physical switch.

   The following examples place swp1 in bond1 and swp2 in bond2.

    {{< tabs "TabID67 ">}}
{{< tab "NCLU Commands ">}}

The example also adds a description for the bonds (an alias), which is optional.

```
cumulus@leaf01:~$ net add bond bond1 bond slaves swp1
cumulus@leaf01:~$ net add bond bond1 alias bond1 on swp1
cumulus@leaf01:~$ net add bond bond2 bond slaves swp2
cumulus@leaf01:~$ net add bond bond2 alias bond2 on swp2
cumulus@leaf01:~$ net pending
cumulus@leaf01:~$ net commit
```

{{< /tab >}}
{{< tab "NVUE Commands ">}}

```
cumulus@leaf01:~$ nv set interface bond1 bond member swp1
cumulus@leaf01:~$ nv set interface bond2 bond member swp2
cumulus@leaf01:~$ nv config apply
```

{{< /tab >}}
{{< tab "Linux Commands ">}}

Add the following lines to the `/etc/network/interfaces` file. The example also adds a description for the bonds (an alias), which is optional.

```
cumulus@leaf01:~$ sudo nano /etc/network/interfaces
...
auto bond1
iface bond1
    alias bond1 on swp1
    bond-slaves swp1
...

auto bond2
iface bond2
    alias bond2 on swp2
    bond-slaves swp2
...
```

{{< /tab >}}
{{< /tabs >}}

3. Add a unique MLAG ID (clag-id) to each bond.

   You must specify a unique MLAG ID (clag-id) for every dual-connected bond on each peer switch so that switches know which links are dual-connected or are connected to the same host or switch. The value must be between 1 and 65535 and must be the same on both peer switches. A value of 0 disables MLAG on the bond.

   The example commands below add an MLAG ID of 1 to bond1 and 2 to bond2:

    {{< tabs "TabID110 ">}}
{{< tab "NCLU Commands ">}}

```
cumulus@leaf01:~$ net add bond bond1 clag id 1
cumulus@leaf01:~$ net add bond bond2 clag id 2
cumulus@leaf01:~$ net pending
cumulus@leaf01:~$ net commit
```

{{< /tab >}}
{{< tab "NVUE Commands ">}}

```
cumulus@leaf01:~$ nv set interface bond1 bond mlag id 1
cumulus@leaf01:~$ nv set interface bond2 bond mlag id 2 
cumulus@leaf01:~$ nv config apply
```

{{< /tab >}}
{{< tab "Linux Commands ">}}

In the `/etc/network/interfaces` file, add the line `clag-id 1` to the `auto bond1` stanza and `clag-id 2` to `auto bond2` stanza:

```
cumulus@switch:~$ sudo nano /etc/network/interfaces
...
auto bond1
iface bond1
    alias bond1 on swp1
    bond-slaves swp1
    clag-id 1

auto bond2
iface bond2
    alias bond2 on swp2
    bond-slaves swp2
    clag-id 2
...
```

{{< /tab >}}
{{< /tabs >}}

4. Add the bonds you created above to a bridge. The example commands below add bond1 and bond2 to a VLAN-aware bridge.

   You must add **all** VLANs configured on the MLAG bond to the bridge so that traffic to the downstream device connected in MLAG is redirected successfully over the peerlink in case of an MLAG bond failure.

   {{< tabs "TabID150 ">}}
{{< tab "NCLU Commands ">}}

```
cumulus@leaf01:~$ net add bridge bridge ports bond1,bond2
cumulus@leaf01:~$ net pending
cumulus@leaf01:~$ net commit
```

{{< /tab >}}
{{< tab "NVUE Commands ">}}

```
cumulus@leaf01:~$ nv set interface bond1-2 bridge domain br_default 
cumulus@leaf01:~$ nv config apply
```

{{< /tab >}}
{{< tab "Linux Commands ">}}

Edit the `/etc/network/interfaces` file to add the `bridge-ports bond1 bond2` lines to the `auto bridge` stanza:

```
cumulus@leaf01:~$ sudo nano /etc/network/interfaces
...
auto bridge
iface bridge
    bridge-ports bond1 bond2
    bridge-vlan-aware yes
...
```

{{< /tab >}}
{{< /tabs >}}

5. Create the inter-chassis bond and the peer link VLAN (as a VLAN subinterface). You also need to provide the peer link IP address, the MLAG bond interfaces, the MLAG system MAC address, and the backup interface.
   - By default, Cumulus Linux configures the inter-chassis bond with the name *peerlink* and the peer link VLAN with the name *peerlink.4094*. Use *peerlink.4094* to ensure that the VLAN is independent of the bridge and spanning tree forwarding decisions.
   - The peer link IP address is an unrouteable linklocal address that provides layer 3 connectivity between the peer switches.
   - NVIDIA provides a reserved range of MAC addresses for MLAG (between 44:38:39:ff:00:00 and 44:38:39:ff:ff:ff). Use a MAC address from this range to prevent conflicts with other interfaces in the same bridged network.
      - Do not to use a multicast MAC address.
      - Do not use the same MAC address for different MLAG pairs; make sure you specify a different MAC address for each MLAG pair in the network.  
   - The backup IP address is any layer 3 backup interface for the peer link, which is used in case the peer link goes down. The backup IP address is **required** and **must** be different than the peer link IP address. It must be reachable by a route that does not use the peer link. Use the loopback or management IP address of the switch.
      {{< expand "Loopback or Management IP Address?" >}}
- If your MLAG configuration has **bridged uplinks** (such as a campus network or a large, flat layer 2 network), use the peer switch **eth0** address. When the peer link is down, the secondary switch routes towards the eth0 address using the OOB network (provided you have implemented an OOB network).
- If your MLAG configuration has **routed uplinks** (a modern approach to the data center fabric network), use the peer switch **loopback** address. When the peer link is down, the secondary switch routes towards the loopback address using uplinks (towards the spine layer). If the primary switch is also suffering a more significant problem (for example, `switchd` is unresponsive or stopped), the secondary switch eventually promotes itself to primary and traffic now flows normally. 

   {{%notice note%}}
When using BGP, to ensure IP connectivity between the loopbacks, the MLAG peer switches must use unique BGP ASNs; if they use the same ASN, you must bypass the BGP loop prevention check on the `AS_PATH` attribute.
{{%/notice%}}

{{< /expand >}}

   The following examples show commands for both MLAG peers (leaf01 and leaf02).

   {{< tabs "TabID222 ">}}
{{< tab "NCLU Commands ">}}

The NCLU command is a macro command that:
- Automatically creates the inter-chassis bond (`peerlink`) and the peer link VLAN subinterface (`peerlink.4094`), and adds the `peerlink` bond to the bridge
- Configures the peer link IP address (`primary` is the linklocal address)
- Adds the MLAG system MAC address, the MLAG bond interfaces, and the backup IP address you specify

   {{< tabs "TabID230 ">}}
{{< tab "leaf01 ">}}

```
cumulus@leaf01:~$ net add clag peer sys-mac 44:38:39:BE:EF:AA interface swp49-50 primary backup-ip 10.10.10.2
cumulus@leaf01:~$ net pending
cumulus@leaf01:~$ net commit
```

To configure the backup link to a VRF, include the name of the VRF with the `backup-ip` parameter. The following example configures the backup link to VRF RED:

```
cumulus@leaf01:~$ net add clag peer sys-mac 44:38:39:BE:EF:AA interface swp49-50 primary backup-ip 10.10.10.2 vrf RED
cumulus@leaf01:~$ net pending
cumulus@leaf01:~$ net commit
```

{{< /tab >}}
{{< tab "leaf02 ">}}

```
cumulus@leaf02:~$ net add clag peer sys-mac 44:38:39:BE:EF:AA interface swp49-50 primary backup-ip 10.10.10.1
cumulus@leaf02:~$ net pending
cumulus@leaf02:~$ net commit
```

To configure the backup link to a VRF, include the name of the VRF with the `backup-ip` parameter. The following example configures the backup link to VRF RED:

```
cumulus@leaf02:~$ net add clag peer sys-mac 44:38:39:BE:EF:AA interface swp49-50 primary backup-ip 10.10.10.1 vrf RED
cumulus@leaf02:~$ net pending
cumulus@leaf02:~$ net commit
```

{{< /tab >}}
{{< /tabs >}}

{{< /tab >}}
{{< tab "NVUE Commands ">}}

   {{< tabs "TabID223 ">}}
{{< tab "leaf01 ">}}

```
cumulus@leaf01:~$ nv set interface peerlink bond member swp49-50
cumulus@leaf01:~$ nv set mlag mac-address 44:38:39:BE:EF:AA
cumulus@leaf01:~$ nv set mlag backup 10.10.10.2
cumulus@leaf01:~$ nv set mlag peer-ip linklocal
cumulus@leaf01:~$ nv config apply
```

To configure the backup link to a VRF, include the name of the VRF with the `backup-ip` parameter. The following example configures the backup link to VRF RED:

```
cumulus@leaf01:~$ nv set mlag backup 10.10.10.2 vrf RED
cumulus@leaf01:~$ nv config apply
```

{{< /tab >}}
{{< tab "leaf02 ">}}

```
cumulus@leaf02:~$ nv set interface peerlink bond member swp49-50
cumulus@leaf02:~$ nv set mlag mac-address 44:38:39:BE:EF:AA
cumulus@leaf02:~$ nv set mlag backup 10.10.10.1
cumulus@leaf02:~$ nv set mlag peer-ip linklocal
cumulus@leaf02:~$ nv config apply
```

To configure the backup link to a VRF, include the name of the VRF with the backup-ip parameter. The following example configures the backup link to VRF RED:

```
cumulus@leaf02:~$ nv set mlag backup 10.10.10.1 vrf RED
cumulus@leaf02:~$ nv config apply
```

{{< /tab >}}
{{< /tabs >}}

{{< /tab >}}
{{< tab "Linux Commands ">}}

Edit the `/etc/network/interfaces` file to add the following parameters, then run the `sudo ifreload -a` command.
- The inter-chasis bond (`peerlink`) with two ports in the bond (swp49 and swp50 in the example command below)
- The `peerlink` bond to the bridge
- The peer link VLAN (`peerlink.4094`) with the backup IP address, the peer link IP address (linklocal), and the MLAG system MAC address (from the reserved range of addresses).

   {{< tabs "TabID315 ">}}
{{< tab "leaf01 ">}}

```
cumulus@leaf01:~$ sudo nano /etc/network/interfaces
...
auto br_default
iface br_default
    bridge-ports bond1 bond2 peerlink
    bridge-vlan-aware yes
...
auto peerlink
iface peerlink
    bond-slaves swp49 swp50

auto peerlink.4094
iface peerlink.4094
    clagd-backup-ip 10.10.10.2
    clagd-peer-ip linklocal
    clagd-sys-mac 44:38:39:BE:EF:AA
...
```

To configure the backup link to a VRF, include the name of the VRF with the `clagd-backup-ip` parameter. The following example configures the backup link to VRF RED:

```
cumulus@leaf01:~$ sudo nano /etc/network/interfaces
...
auto peerlink.4094
iface peerlink.4094
    clagd-backup-ip 10.10.10.2 vrf RED
    clagd-peer-ip linklocal
    clagd-sys-mac 44:38:39:BE:EF:AA
...
```

Run the `sudo ifreload -a` command to apply all the configuration changes:

```
cumulus@leaf01:~$ sudo ifreload -a
```

{{< /tab >}}
{{< tab "leaf02 ">}}

```
cumulus@leaf02:~$ sudo nano /etc/network/interfaces
...
auto br_default
iface br_default
    bridge-ports bond1 bond2 peerlink
    bridge-vlan-aware yes
...
auto peerlink
iface peerlink
    bond-slaves swp49 swp50

auto peerlink.4094
iface peerlink.4094
    clagd-backup-ip 10.10.10.1
    clagd-peer-ip linklocal
    clagd-sys-mac 44:38:39:BE:EF:AA
...
```

To configure the backup link to a VRF, include the name of the VRF with the `clagd-backup-ip` parameter. The following example configures the backup link to VRF RED:

```
cumulus@leaf02:~$ sudo nano /etc/network/interfaces
...
auto peerlink.4094
iface peerlink.4094
    clagd-backup-ip 10.10.10.1 vrf RED
    clagd-peer-ip linklocal
    clagd-sys-mac 44:38:39:BE:EF:AA
...
```

Run the `sudo ifreload -a` command to apply all the configuration changes:

```
cumulus@leaf02:~$ sudo ifreload -a
```

{{< /tab >}}
{{< /tabs >}}

{{< /tab >}}
{{< /tabs >}}

{{%notice note%}}
- Do *not* add VLAN 4094 to the bridge VLAN list; VLAN 4094 for the peer link subinterface **cannot** be configured as a bridged VLAN with bridge VIDs under the bridge.
- Do not use 169.254.0.1 as the MLAG peer link IP address; Cumulus Linux uses this address exclusively for {{<link url="Border-Gateway-Protocol-BGP#bgp-unnumbered" text="BGP unnumbered">}} interfaces.
- When you configure MLAG manually in the `/etc/network/interfaces` file, the changes take effect when you bring the peer link interface up with the `sudo ifreload -a` command. Do **not** use `systemctl restart clagd.service` to apply the new configuration.
{{%/notice%}}

MLAG synchronizes the dynamic state between the two peer switches but it does not synchronize the switch configurations. After modifying the configuration of one peer switch, you must make the same changes to the configuration on the other peer switch. This applies to all configuration changes, including:
- Port configuration, such as VLAN membership, {{<link url="#mtu-and-mlag" text="MTU">}} and bonding parameters.
- Bridge configuration, such as spanning tree parameters or bridge properties.
- Static address entries, such as static FDB entries and static IGMP entries.
- QoS configuration, such as ACL entries.

## Optional Configuration

This section describes optional configuration procedures.

### Set Roles and Priority

Each MLAG-enabled switch in the pair has a *role*. When the peering relationship is established between the two switches, one switch is put into the *primary* role and the other into the *secondary* role. When an MLAG-enabled switch is in the secondary role, it does not send STP BPDUs on dual-connected links; it only sends BPDUs on single-connected links. The switch in the primary role sends STP BPDUs on all single- and dual-connected links.

By default, the role is determined by comparing the MAC addresses of the two sides of the peering link; the switch with the lower MAC address assumes the primary role. You can override this by setting the `priority` option for the peer link:

{{< tabs "TabID308 ">}}
{{< tab "NCLU Commands ">}}

```
cumulus@leaf01:~$ net add interface peerlink.4094 clag priority 2048
cumulus@leaf01:~$ net pending
cumulus@leaf01:~$ net commit
```

{{< /tab >}}
{{< tab "NVUE Commands ">}}

```
cumulus@leaf01:~$ nv set mlag priority 2084
cumulus@leaf01:~$ nv config apply
```

{{< /tab >}}
{{< tab "Linux Commands ">}}

Edit the `/etc/network/interfaces` file and add the `clagd-priority` option, then run the `ifreload -a` command.

```
cumulus@switch:~$ sudo nano /etc/network/interfaces
...
auto peerlink.4094
iface peerlink.4094
    clagd-peer-ip linklocal
    clagd-backup-ip 10.10.10.2
    clagd-sys-mac 44:38:39:BE:EF:AA
    clagd-priority 2048
...
```

```
cumulus@switch:~$ sudo ifreload -a
```

{{< /tab >}}
{{< /tabs >}}

The switch with the lower priority value is given the primary role; the default value is 32768 and the range is between 0 and 65535.

When the MLAG service exits during switch reboot or if you stop the service on the primary switch, the peer switch that is in the secondary role becomes the primary.

However, if the primary switch goes down without stopping the MLAG service for any reason, or if the peer link goes down, the secondary switch does **not** change its role. If the peer switch is determined to not be alive, the switch in the secondary role rolls back the LACP system ID to be the bond interface MAC address instead of the MLAG system MAC address (`clagd-sys-mac`) and the switch in primary role uses the MLAG system MAC address as the LACP system ID on the bonds.

### Set clagctl Timers

The `clagd` service has a number of timers that you can tune for enhanced performance:

| <div style="width:250px">Timer | Description |
| ----- | ----------- |
| `--reloadTimer <seconds>` | The number of seconds to wait for the peer switch to become active. If the peer switch does not become active after the timer expires, the MLAG bonds leave the initialization ({{<link url="#peer-link-interfaces-and-the-protodown-state" text="protodown">}}) state and become active. This provides `clagd` with sufficient time to determine whether the peer switch is coming up or if it is permanently unreachable. <br>The default is 300 seconds.|
| `--peerTimeout <seconds>`<br> | The number of seconds `clagd` waits without receiving any messages from the peer switch before it determines that the peer is no longer active. At this point, the switch reverts all configuration changes so that it operates as a standard non-MLAG switch. This includes removing all statically assigned MAC addresses, clearing the egress forwarding mask, and allowing addresses to move from any port to the peer port. After a message is again received from the peer, MLAG operation restarts. If this parameter is not specified, `clagd` uses ten times the local `lacpPoll` value. |
| `--initDelay <seconds>` | The number of seconds `clagd` delays bringing up MLAG bonds and anycast IP addresses. <br>The default is 180 seconds.<br>This timer is set to 0 automatically under the following conditions:<ul><li>When the peer is not alive and the backup link is not active after a reload timeout</li><li>When the peer sends a goodbye (through the peerlink or the backup link)</li><li>When both MLAG sessions come up at the same time</li></ul>|
| `--sendTimeout <seconds>` | The number of seconds `clagd` waits until the sending socket times out. If it takes longer than the `sendTimeout` value to send data to the peer, `clagd` generates an exception. <br>The default is 30 seconds. |
| `--lacpPoll <seconds>` | The number of seconds `clagd` waits before obtaining local LACP information. <br>The default is 2 seconds.|

{{< tabs "TabID363 ">}}
{{< tab "NCLU Commands ">}}

Run the `net add interface peerlink.4094 clag args <timer> <value>` command. The following example command sets the peerlink timer to 900 seconds:

```
cumulus@leaf01:~$ net add interface peerlink.4094 clag args --initDelay 100
cumulus@leaf01:~$ net pending
cumulus@leaf01:~$ net commit
```

{{< /tab >}}
{{< tab "NVUE Commands ">}}

Currently, the only timer you can set with NVUE is the initial delay timer. The following example NVUE Command sets the initial delay to 100 seconds:

```
cumulus@leaf01:~$ nv set mlag init-delay 100
cumulus@leaf01:~$ nv config apply
```

{{< /tab >}}
{{< tab "Linux Commands ">}}

To set the the `clagd` timers, edit the `/etc/network/interfaces` file to add the `clagd-args --<timer>` line to the peerlink.4094 stanza, then run the `ifreload -a` command.

The following example command sets the initial delay timer to 100 seconds:

```
cumulus@leaf01:~$ sudo nano /etc/network/interfaces
...
auto peerlink.4094
iface peerlink.4094
    clagd-args --initDelay 100
    clagd-peer-ip linklocal
    clagd-backup-ip 10.10.10.2
    clagd-sys-mac 44:38:39:BE:EF:AA
    clagd-priority 2048
...
```

```
cumulus@leaf01:~$ sudo ifreload -a
```

The following example command sets the peer timeout to 900 seconds:

```
cumulus@leaf01:~$ sudo nano /etc/network/interfaces
...
auto peerlink.4094
iface peerlink.4094
    clagd-args --peerTimeout 900
    clagd-peer-ip linklocal
    clagd-backup-ip 10.10.10.2
    clagd-sys-mac 44:38:39:BE:EF:AA
    clagd-priority 2048
...
```

```
cumulus@leaf01:~$ sudo ifreload -a
```

{{< /tab >}}
{{< /tabs >}}

### Configure MLAG with a Traditional Mode Bridge

To configure MLAG with a traditional mode bridge instead of a {{<link url="VLAN-aware-Bridge-Mode" text="VLAN-aware mode bridge">}}, you must configure the peer link and all dual-connected links as {{<link url="Traditional-Bridge-Mode" text="untagged (native)">}} ports on a bridge (note the absence of any VLANs in the `bridge-ports` line and the lack of the `bridge-vlan-aware` parameter below):

```
...
auto br0
iface br0
    bridge-ports peerlink bond1 bond2
...
```

The following example shows you how to allow VLAN 10 across the peer link:

```
...
auto br0.10
iface br0.10
    bridge-ports peerlink.10 bond1.10 bond2.10
    bridge-stp on
...
```

### Configure a Backup UDP Port

By default, Cumulus Linux uses UDP port 5342 with the backup IP address. To change the backup UDP port, edit the `/etc/network/interfaces` file to add `clagd-args --backupPort <port>` to the `auto peerlink.4094` stanza. For example:

{{< tabs "TabID582 ">}}
{{< tab "NCLU Commands ">}}

```
cumulus@leaf01:~$ net add interface peerlink.4094 clag args --backupPort 5400
cumulus@leaf01:~$ net pending
cumulus@leaf01:~$ net commit
```

{{< /tab >}}
{{< tab "Linux Commands ">}}

```
...
auto peerlink.4094
iface peerlink.4094
    clagd-args --backupPort 5400
    clagd-backup-ip 10.10.10.2
    clagd-peer-ip linklocal
    clagd-sys-mac 44:38:39:BE:EF:AA
...
```

Run the `sudo ifreload -a` command to apply all the configuration changes:

```
cumulus@leaf01:~$ sudo ifreload -a
```

{{< /tab >}}
{{< /tabs >}}

## Best Practices

Follow these best practices when configuring MLAG on your switches.

### MTU and MLAG

The {{<link url="Switch-Port-Attributes#mtu" text="MTU">}} in MLAG traffic is determined by the bridge MTU. Bridge MTU is determined by the lowest MTU setting of an interface that is a member of the bridge. If you want to set an MTU other than the default of 9216 bytes, you must configure the MTU on each physical interface and bond interface that is a member of every MLAG bridge in the entire bridged domain.

The following example commands set an MTU of 1500 for each of the bond interfaces (peerlink, uplink, bond1, bond2), which are members of bridge *bridge*:

{{< tabs "TabID498 ">}}
{{< tab "NCLU Commands ">}}

```
cumulus@leaf01:~$ net add bond peerlink mtu 1500
cumulus@leaf01:~$ net add bond uplink mtu 1500
cumulus@leaf01:~$ net add bond bond1 mtu 1500
cumulus@leaf01:~$ net add bond bond2 mtu 1500
cumulus@leaf01:~$ net pending
cumulus@leaf01:~$ net commit
```

{{< /tab >}}
{{< tab "NVUE Commands ">}}

```
cumulus@leaf01:~$ nv set interface peerlink.4094 link mtu 1500
cumulus@leaf01:~$ nv set interface uplink link mtu 1500
cumulus@leaf01:~$ nv set interface bond1 link mtu 1500
cumulus@leaf01:~$ nv set interface bond2 link mtu 1500
cumulus@leaf01:~$ nv config apply
```

{{< /tab >}}
{{< tab "Linux Commands ">}}

Edit the `/etc/network/interfaces` file, then run the `ifreload -a` command. For example:

```
cumulus@leaf01:~$ sudo nano /etc/network/interfaces
...
auto br_default
iface br_default
    bridge-ports peerlink uplink bond1 bond2

auto peerlink
iface peerlink
    mtu 1500

auto bond1
iface bond1
    mtu 1500

auto bond2
iface bond2
    mtu 1500

auto uplink
iface uplink
    mtu 1500
...
```

```
cumulus@leaf01:~$ sudo ifreload -a
```

{{< /tab >}}
{{< /tabs >}}

### STP and MLAG

Always enable STP in your layer 2 network and {{<link title="Spanning Tree and Rapid Spanning Tree - STP#bpdu-guard" text="BPDU Guard">}} on the host-facing bond interfaces.

- The STP global configuration must be the same on both peer switches.
- The STP configuration for dual-connected ports must be the same on both peer switches.
- The STP priority must be the same on both peer switches.
- To minimize convergence times when a link transitions to the forwarding state, configure the edge ports (for tagged and untagged frames) with PortAdminEdge and BPDU guard enabled.

### Peer Link Sizing

The peer link carries very little traffic when compared to the bandwidth consumed by dataplane traffic. In a typical MLAG configuration, most every connection between the two switches in the MLAG pair is dual-connected so the only traffic going across the peer link is traffic from the `clagd` process and some LLDP or LACP traffic; the traffic received on the peer link is not forwarded out of the dual-connected bonds.

However, there are some instances where a host is connected to only one switch in the MLAG pair; for example:

- You have a hardware limitation on the host where there is only one PCIE slot, and therefore, one NIC on the system, so the host is only single-connected across that interface.
- The host does not support 802.3ad and you cannot create a bond on it.
- You are accounting for a link failure, where the host becomes single connected until the failure is resolved.

Determine how much bandwidth is traveling across the single-connected interfaces and allocate half of that bandwidth to the peer link. On average, one half of the traffic destined to the single-connected host arrives on the switch directly connected to the single-connected host and the other half arrives on the switch that is not directly connected to the single-connected host. When this happens, only the traffic that arrives on the switch that is not directly connected to the single-connected host needs to traverse the peer link.

In addition, you might want to add extra links to the peer link bond to handle link failures in the peer link bond itself.

| | |
|---|---|
|<div style="width:600px">{{< img src="/images/cumulus-linux/mlag-peerlink-sizing.png" width="800" >}}| <br><ul><li>Each host has two 10G links, with each 10G link going to each switch in the MLAG pair. </li><li>Each host has 20G of dual-connected bandwidth; all three hosts have a total of 60G of dual-connected bandwidth. </li><li>Allocate at least 15G of bandwidth to each peer link bond, which represents half of the single-connected bandwidth.</li></ul>

When planning for link failures for a full rack, you need only allocate enough bandwidth to meet your site strategy for handling failure scenarios. For example, for a full rack with 40 servers and two switches, you might plan for four to six servers to lose connectivity to a single switch and become single connected before you respond to the event. Therefore, if you have 40 hosts each with 20G of bandwidth dual-connected to the MLAG pair, you might allocate between 20G and 30G of bandwidth to the peer link, which accounts for half of the single-connected bandwidth for four to six hosts.

### Peer Link Routing

When enabling a routing protocol in an MLAG environment, it is also necessary to manage the uplinks; by default MLAG is not aware of layer 3 uplink interfaces. If there is a peer link failure, MLAG does not remove static routes or bring down a BGP or OSPF adjacency unless you use a separate link state daemon such as `ifplugd`.

When you use MLAG with VRR, set up a routed adjacency across the peerlink.4094 interface. If a routed connection is not built across the peer link, during an uplink failure on one of the switches in the MLAG pair, egress traffic might not be forwarded if the destination is on the switch whose uplinks are down.

To set up the adjacency, configure a {{<link url="Border-Gateway-Protocol-BGP#bgp-unnumbered" text="BGP">}} or {{<link url="Open-Shortest-Path-First-OSPF" text="OSPF">}} unnumbered peering, as appropriate for your network.

For BGP, use a configuration like this:

{{< tabs "TabID704 ">}}
{{< tab "NCLU Commands ">}}

```
cumulus@leaf01:~$ net add bgp neighbor peerlink.4094 interface remote-as internal
cumulus@leaf01:~$ net commit
```

If you are using {{<link url="Ethernet-Virtual-Private-Network-EVPN" text="EVPN">}} and MLAG, you need to enable the EVPN address family across the peerlink.4094 interface as well:

```
cumulus@leaf01:~$ net add bgp neighbor peerlink.4094 interface remote-as internal
cumulus@leaf01:~$ net add bgp l2vpn evpn neighbor peerlink.4094 activate
cumulus@leaf01:~$ net commit
```

{{%notice note%}}
If you use NCLU to create an iBGP peering across the peer link, the `net add bgp l2vpn evpn neighbor peerlink.4094 activate` command creates a new eBGP neighborship when one is already configured for iBGP. The existing iBGP configuration is still valid.
{{%/notice%}}

{{< /tab >}}
{{< tab "NVUE Commands ">}}

```
cumulus@leaf01:~$ nv set vrf default router bgp peer peerlink remote-as internal
cumulus@leaf01:~$ nv config apply
```

{{< /tab >}}
{{< tab "vtysh Commands ">}}

```
cumulus@leaf01:~$ sudo vtysh
leaf01# configure terminal
leaf01(config)# router bgp 65101
leaf01(config-router)# bgp router-id 10.10.10.1
leaf01(config-router)# neighbor peerlink remote-as external
leaf01(config-router)# end
leaf01# write memory
leaf01# exit
cumulus@leaf01:~$
```

If you are using {{<link url="Ethernet-Virtual-Private-Network-EVPN" text="EVPN">}} and MLAG, you need to enable the EVPN address family across the peerlink.4094 interface as well:

```
cumulus@leaf01:~$ sudo vtysh
leaf01# configure terminal
leaf01(config)# router bgp 65101
leaf01(config-router)# bgp router-id 10.10.10.1
leaf01(config-router)# neighbor peerlink remote-as external
leaf01(config-router)# address-family l2vpn evpn
leaf01(config-router-af)# neighbor peerlink activate
leaf01(config-router-af)# end
leaf01# write memory
leaf01# exit
cumulus@leaf01:~$
```

{{< /tab >}}
{{< /tabs >}}

For OSPF, use a configuration like this:

{{< tabs "TabID787 ">}}
{{< tab "NCLU Commands ">}}

```
cumulus@leaf01:~$ net add interface peerlink.4094 ospf area 0.0.0.1
cumulus@v:~$ net commit
```

{{< /tab >}}
{{< tab "NVUE Commands ">}}

```
cumulus@leaf01:~$ nv set interface peerlink.4094 router ospf area 0.0.0.1
cumulus@leaf01:~$ nv config apply
```

{{< /tab >}}
{{< /tabs >}}

## Configuration Example

The example below shows a basic MLAG configuration, where:
- leaf01 and leaf02 are MLAG peers
- Three bonds are configured for MLAG, each with a single port, a peer link that is a bond with two member ports, and three VLANs on each port

{{< figure src="/images/cumulus-linux/mlag-config.png" width="450" >}}

For an example configuration with MLAG and BGP, see the {{<link title="Configuration Example" text="BGP configuration example">}}.

### NCLU Commands

{{< tabs "TabID803 ">}}
{{< tab "NCLU ">}}

{{< tabs "TabID821 ">}}
{{< tab "leaf01 ">}}

```
cumulus@leaf01:~$ net add loopback lo ip address 10.10.10.1/32
cumulus@leaf01:~$ net add bond bond1 bond slaves swp1
cumulus@leaf01:~$ net add bond bond2 bond slaves swp2
cumulus@leaf01:~$ net add bond bond3 bond slaves swp3
cumulus@leaf01:~$ net add bond bond1 clag id 1
cumulus@leaf01:~$ net add bond bond2 clag id 2
cumulus@leaf01:~$ net add bond bond3 clag id 3
cumulus@leaf01:~$ net add bridge bridge ports bond1,bond2,bond3
cumulus@leaf01:~$ net add vlan 10 vlan-id 10
cumulus@leaf01:~$ net add vlan 20 vlan-id 20
cumulus@leaf01:~$ net add vlan 30 vlan-id 30
cumulus@leaf01:~$ net add vlan 10 ip address 10.1.10.2/24
cumulus@leaf01:~$ net add vlan 20 ip address 10.1.20.2/24
cumulus@leaf01:~$ net add vlan 30 ip address 10.1.30.2/24
cumulus@leaf01:~$ net add bridge bridge vids 10,20,30
cumulus@leaf01:~$ net add clag peer sys-mac 44:38:39:BE:EF:AA interface swp49-50 primary backup-ip 10.10.10.2
cumulus@leaf01:~$ net add interface peerlink.4094 clag args --initDelay 100
cumulus@leaf01:~$ net pending
cumulus@leaf01:~$ net commit
```

{{< /tab >}}
{{< tab "leaf02 ">}}

```
cumulus@leaf02:~$ net add loopback lo ip address 10.10.10.2/32
cumulus@leaf02:~$ net add bond bond1 bond slaves swp1
cumulus@leaf02:~$ net add bond bond2 bond slaves swp2
cumulus@leaf02:~$ net add bond bond3 bond slaves swp3
cumulus@leaf02:~$ net add bond bond1 clag id 1
cumulus@leaf02:~$ net add bond bond2 clag id 2
cumulus@leaf02:~$ net add bond bond3 clag id 3
cumulus@leaf02:~$ net add bridge bridge ports bond1,bond2,bond3
cumulus@leaf02:~$ net add vlan 10 vlan-id 10
cumulus@leaf02:~$ net add vlan 20 vlan-id 20
cumulus@leaf02:~$ net add vlan 30 vlan-id 30
cumulus@leaf02:~$ net add vlan 10 ip address 10.1.10.3/24
cumulus@leaf02:~$ net add vlan 20 ip address 10.1.20.3/24
cumulus@leaf02:~$ net add vlan 30 ip address 10.1.30.3/24
cumulus@leaf02:~$ net add bridge bridge vids 10,20,30
cumulus@leaf02:~$ net add clag peer sys-mac 44:38:39:BE:EF:AA interface swp49-50 primary backup-ip 10.10.10.1
cumulus@leaf02:~$ net add interface peerlink.4094 clag args --initDelay 100
cumulus@leaf02:~$ net pending
cumulus@leaf02:~$ net commit
```

{{< /tab >}}
{{< tab "spine01 ">}}

```
cumulus@spine01:~$ net add loopback lo ip address 10.10.10.101/32
cumulus@spine01:~$ net add interface swp1
cumulus@spine01:~$ net pending
cumulus@spine01:~$ net commit
```

{{< /tab >}}
{{< /tabs >}}

{{< /tab >}}
{{< tab "/etc/network/interfaces ">}}

{{< tabs "TabID860 ">}}
{{< tab "leaf01 ">}}

```
cumulus@leaf01:~$ cat /etc/network/interfaces

auto lo
iface lo inet loopback
    address 10.10.10.1/32

auto swp1
iface swp1

auto swp2
iface swp2

auto swp3
iface swp3

auto swp49
iface swp49

auto swp50
iface swp50

auto bond1
iface bond1
    bond-slaves swp1
    clag-id 1

auto bond2
iface bond2
    bond-slaves swp2
    clag-id 2

auto bond3
iface bond3
    bond-slaves swp3
    clag-id 3

auto bridge
iface bridge
    bridge-ports bond1 bond2 bond3 peerlink
    bridge-vids 10 20 30
    bridge-vlan-aware yes

auto mgmt
iface mgmt
    vrf-table auto
    address 127.0.0.1/8
    address ::1/128

auto eth0
iface eth0 inet dhcp
    vrf mgmt

auto peerlink
iface peerlink
    bond-slaves swp49 swp50

auto peerlink.4094
iface peerlink.4094
    clagd-args --initDelay 100
    clagd-backup-ip 10.10.10.2
    clagd-peer-ip linklocal
    clagd-priority 1000
    clagd-sys-mac 44:38:39:BE:EF:AA

auto vlan10
iface vlan10
    address 10.1.10.2/24
    vlan-id 10
    vlan-raw-device bridge

auto vlan20
iface vlan20
    address 10.1.20.2/24
    vlan-id 20
    vlan-raw-device bridge

auto vlan30
iface vlan30
    address 10.1.30.2/24
    vlan-id 30
    vlan-raw-device bridge
```

{{< /tab >}}
{{< tab "leaf02 ">}}

```
cumulus@leaf02:~$ cat /etc/network/interfaces

auto lo
iface lo inet loopback
    address 10.10.10.2/32

auto swp1
iface swp1

auto swp2
iface swp2

auto swp3
iface swp3

auto swp49
iface swp49

auto swp50
iface swp50

auto bond1
iface bond1
    bond-slaves swp1
    clag-id 1

auto bond2
iface bond2
    bond-slaves swp2
    clag-id 2

auto bond3
iface bond3
    bond-slaves swp3
    clag-id 3

auto bridge
iface bridge
    bridge-ports bond1 bond2 bond3 peerlink
    bridge-vids 10 20 30
    bridge-vlan-aware yes

auto mgmt
iface mgmt
    vrf-table auto
    address 127.0.0.1/8
    address ::1/128

auto eth0
iface eth0 inet dhcp
    vrf mgmt

auto peerlink
iface peerlink
    bond-slaves swp49 swp50

auto peerlink.4094
iface peerlink.4094
    clagd-args --initDelay 100
    clagd-backup-ip 10.10.10.1
    clagd-peer-ip linklocal
    clagd-priority 32768
    clagd-sys-mac 44:38:39:BE:EF:AA

auto vlan10
iface vlan10
    address 10.1.10.3/24
    vlan-id 10
    vlan-raw-device bridge

auto vlan20
iface vlan20
    address 10.1.20.3/24
    vlan-id 20
    vlan-raw-device bridge

auto vlan30
iface vlan30
    address 10.1.30.3/24
    vlan-id 30
    vlan-raw-device bridge
```

{{< /tab >}}
{{< tab "spine01 ">}}

```
cumulus@spine01:~$ cat /etc/network/interfaces

auto lo
iface lo inet loopback
    address 10.10.10.101/32

auto swp1
iface swp1

auto mgmt
iface mgmt
    vrf-table auto
    address 127.0.0.1/8
    address ::1/128

auto eth0
iface eth0 inet dhcp
    vrf mgmt
```

{{< /tab >}}
{{< /tabs >}}

{{< /tab >}}
{{< /tabs >}}

### NVUE Commands

{{< tabs "TabID1087 ">}}
{{< tab "NVUE ">}}

{{< tabs "TabID1090 ">}}
{{< tab "leaf01 ">}}

```
cumulus@leaf01:~$ nv set interface lo ip address 10.10.10.1/32
cumulus@leaf01:~$ nv set interface bond1 bond member swp1
cumulus@leaf01:~$ nv set interface bond2 bond member swp2
cumulus@leaf01:~$ nv set interface bond3 bond member swp3
cumulus@leaf01:~$ nv set interface bond1 bond mlag id 1
cumulus@leaf01:~$ nv set interface bond2 bond mlag id 2
cumulus@leaf01:~$ nv set interface bond3 bond mlag id 3
cumulus@leaf01:~$ nv set interface vlan10 ip address 10.1.10.2/24
cumulus@leaf01:~$ nv set interface vlan20 ip address 10.1.20.2/24
cumulus@leaf01:~$ nv set interface vlan30 ip address 10.1.30.2/24
cumulus@leaf01:~$ nv set bridge domain br_default vlan 10,20,30
cumulus@leaf01:~$ nv set interface bond1-3 bridge domain br_default 
cumulus@leaf01:~$ nv set interface peerlink bond member swp49-50
cumulus@leaf01:~$ nv set mlag mac-address 44:38:39:BE:EF:AA
cumulus@leaf01:~$ nv set mlag backup 10.10.10.2
cumulus@leaf01:~$ nv set mlag peer-ip linklocal
cumulus@leaf01:~$ nv set mlag init-delay 100
cumulus@leaf01:~$ nv config apply
```

{{< /tab >}}
{{< tab "leaf02 ">}}

```
cumulus@leaf02:~$ nv set interface lo ip address 10.10.10.2/32
cumulus@leaf02:~$ nv set interface bond1 bond member swp1
cumulus@leaf02:~$ nv set interface bond2 bond member swp2
cumulus@leaf02:~$ nv set interface bond3 bond member swp3
cumulus@leaf02:~$ nv set interface bond1 bond mlag id 1
cumulus@leaf02:~$ nv set interface bond2 bond mlag id 2
cumulus@leaf02:~$ nv set interface bond3 bond mlag id 3
cumulus@leaf02:~$ nv set interface vlan10 ip address 10.1.10.3/24
cumulus@leaf02:~$ nv set interface vlan20 ip address 10.1.20.3/24
cumulus@leaf02:~$ nv set interface vlan30 ip address 10.1.30.3/24
cumulus@leaf02:~$ nv set bridge domain br_default vlan 10,20,30
cumulus@leaf02:~$ nv set interface bond1-3 bridge domain br_default
cumulus@leaf02:~$ nv set interface peerlink bond member swp49-50
cumulus@leaf02:~$ nv set mlag mac-address 44:38:39:BE:EF:AA
cumulus@leaf02:~$ nv set mlag backup 10.10.10.1
cumulus@leaf02:~$ nv set mlag peer-ip linklocal
cumulus@leaf02:~$ nv set mlag init-delay 100
cumulus@leaf02:~$ nv config apply
```

{{< /tab >}}
{{< tab "spine01 ">}}

```
cumulus@spine01:~$ nv set interface lo ip address 10.10.10.101/32
cumulus@spine01:~$ nv set interface swp1 link state up
cumulus@spine01:~$ nv config apply
```

{{< /tab >}}
{{< /tabs >}}

{{< /tab >}}
{{< tab "/etc/nvue.d/startup.yaml ">}}

{{< tabs "TabID1132 ">}}
{{< tab "leaf01 ">}}

```
- set:
    interface:
      bond1:
        bond:
          member:
            swp1: {}
          mlag:
            id: 1
        type: bond
        bridge:
          domain:
            br_default: {}
      bond2:
        bond:
          member:
            swp2: {}
          mlag:
            id: 2
        type: bond
        bridge:
          domain:
            br_default: {}
      bond3:
        bond:
          member:
            swp3: {}
          mlag:
            id: 3
        type: bond
        bridge:
          domain:
            br_default: {}
      vlan10:
        ip:
          address:
            10.1.10.2/24: {}
        type: svi
        vlan: 10
      vlan20:
        ip:
          address:
            10.1.20.2/24: {}
        type: svi
        vlan: 20
      vlan30:
        ip:
          address:
            10.1.30.2/24: {}
        type: svi
        vlan: 30
      peerlink:
        bond:
          member:
            swp49: {}
            swp50: {}
        type: peerlink
      peerlink.4094:
        type: sub
        base-interface: peerlink
        vlan: 4094
      lo:
        ip:
          address:
            10.10.10.1/32: {}
        type: loopback
    bridge:
      domain:
        br_default:
          vlan:
            '10': {}
            '20': {}
            '30': {}
    mlag:
      mac-address: 44:38:39:BE:EF:AA
      backup:
        10.10.10.2: {}
      peer-ip: linklocal
      init-delay: 100
```

{{< /tab >}}
{{< tab "leaf02 ">}}

```
- set:
    interface:
      lo:
        ip:
          address:
            10.10.10.2/32: {}
        type: loopback
      bond1:
        bond:
          member:
            swp1: {}
          mlag:
            id: 1
        type: bond
        bridge:
          domain:
            br_default: {}
      bond2:
        bond:
          member:
            swp2: {}
          mlag:
            id: 2
        type: bond
        bridge:
          domain:
            br_default: {}
      bond3:
        bond:
          member:
            swp3: {}
          mlag:
            id: 3
        type: bond
        bridge:
          domain:
            br_default: {}
      vlan10:
        ip:
          address:
            10.1.10.3/24: {}
        type: svi
        vlan: 10
      vlan20:
        ip:
          address:
            10.1.20.3/24: {}
        type: svi
        vlan: 20
      vlan30:
        ip:
          address:
            10.1.30.3/24: {}
        type: svi
        vlan: 30
      peerlink:
        bond:
          member:
            swp49: {}
            swp50: {}
        type: peerlink
      peerlink.4094:
        type: sub
        base-interface: peerlink
        vlan: 4094
    bridge:
      domain:
        br_default:
          vlan:
            '10': {}
            '20': {}
            '30': {}
    mlag:
      mac-address: 44:38:39:BE:EF:AA
      backup:
        10.10.10.1: {}
      peer-ip: linklocal
      init-delay: 100
```

{{< /tab >}}
{{< tab "spine01 ">}}

```
- set:
    interface:
      lo:
        ip:
          address:
            10.10.10.101/32: {}
        type: loopback
      swp1:
        link:
          state:
            up: {}
        type: swp
```

{{< /tab >}}
{{< /tabs >}}

{{< /tab >}}
{{< tab "/etc/network/interfaces">}}

{{< tabs "TabID1150 ">}}
{{< tab "leaf01 ">}}

```
auto lo
iface lo inet loopback
    address 10.10.10.1/32

auto mgmt
iface mgmt
    address 127.0.0.1/8
    address ::1/128
    vrf-table auto

auto eth0
iface eth0 inet dhcp
    ip-forward off
    ip6-forward off
    vrf mgmt

auto bond1
iface bond1
    bond-slaves swp1
    bond-mode 802.3ad
    bond-lacp-bypass-allow no
    clag-id 1

auto bond2
iface bond2
    bond-slaves swp2
    bond-mode 802.3ad
    bond-lacp-bypass-allow no
    clag-id 2

auto bond3
iface bond3
    bond-slaves swp3
    bond-mode 802.3ad
    bond-lacp-bypass-allow no
    clag-id 3

auto vlan10
iface vlan10
    address 10.1.10.2/24
    hwaddress 44:38:39:22:01:bb
    vlan-raw-device br_default
    vlan-id 10

auto vlan20
iface vlan20
    address 10.1.20.2/24
    hwaddress 44:38:39:22:01:bb
    vlan-raw-device br_default
    vlan-id 20

auto vlan30
iface vlan30
    address 10.1.30.2/24
    hwaddress 44:38:39:22:01:bb
    vlan-raw-device br_default
    vlan-id 30

auto peerlink
iface peerlink
    bond-slaves swp49 swp50
    bond-mode 802.3ad
    bond-lacp-bypass-allow no

auto peerlink.4094
iface peerlink.4094
    clagd-peer-ip linklocal
    clagd-backup-ip 10.10.10.2
    clagd-sys-mac 44:38:39:BE:EF:AA
    clagd-args --initDelay 100

auto br_default
iface br_default
    bridge-ports bond1 bond2 bond3 peerlink
    hwaddress 44:38:39:22:01:bb
    bridge-vlan-aware yes
    bridge-vids 10 20 30
    bridge-pvid 1
```

{{< /tab >}}
{{< tab "leaf02 ">}}

```
auto lo
iface lo inet loopback
    address 10.10.10.2/32

auto mgmt
iface mgmt
    address 127.0.0.1/8
    address ::1/128
    vrf-table auto

auto eth0
iface eth0 inet dhcp
    ip-forward off
    ip6-forward off
    vrf mgmt

auto bond1
iface bond1
    bond-slaves swp1
    bond-mode 802.3ad
    bond-lacp-bypass-allow no
    clag-id 1

auto bond2
iface bond2
    bond-slaves swp2
    bond-mode 802.3ad
    bond-lacp-bypass-allow no
    clag-id 2

auto bond3
iface bond3
    bond-slaves swp3
    bond-mode 802.3ad
    bond-lacp-bypass-allow no
    clag-id 3

auto vlan10
iface vlan10
    address 10.1.10.3/24
    hwaddress 44:38:39:22:01:bb
    vlan-raw-device br_default
    vlan-id 10

auto vlan20
iface vlan20
    address 10.1.20.3/24
    hwaddress 44:38:39:22:01:bb
    vlan-raw-device br_default
    vlan-id 20

auto vlan30
iface vlan30
    address 10.1.30.3/24
    hwaddress 44:38:39:22:01:bb
    vlan-raw-device br_default
    vlan-id 30

auto peerlink
iface peerlink
    bond-slaves swp49 swp50
    bond-mode 802.3ad
    bond-lacp-bypass-allow no

auto peerlink.4094
iface peerlink.4094
    clagd-peer-ip linklocal
    clagd-backup-ip 10.10.10.1
    clagd-sys-mac 44:38:39:BE:EF:AA
    clagd-args --initDelay 100

auto br_default
iface br_default
    bridge-ports bond1 bond2 bond3 peerlink
    hwaddress 44:38:39:22:01:bb
    bridge-vlan-aware yes
    bridge-vids 10 20 30
    bridge-pvid 1
```

{{< /tab >}}
{{< tab "spine01 ">}}

```
auto lo
iface lo inet loopback
    address 10.10.10.101/32

auto mgmt
iface mgmt
    address 127.0.0.1/8
    address ::1/128
    vrf-table auto

auto eth0
iface eth0 inet dhcp
    ip-forward off
    ip6-forward off
    vrf mgmt

auto swp1
iface swp1
```

{{< /tab >}}
{{< /tabs >}}

{{< /tab >}}
{{< /tabs >}}

## Troubleshooting

Use the following troubleshooting tips to check that MLAG is configured and working correctly.

### Check MLAG Status

To check the status of your MLAG configuration, run the NCLU `net show clag` command or the Linux `clagctl` command. For example:

```
cumulus@leaf01:~$ net show clag
The peer is alive
     Our Priority, ID, and Role: 32768 44:38:39:00:00:59 primary
    Peer Priority, ID, and Role: 32768 44:38:39:00:00:5a secondary
          Peer Interface and IP: peerlink.4094 fe80::4638:39ff:fe00:5a (linklocal)
                      Backup IP: 10.10.10.2 (inactive)
                     System MAC: 44:38:39:be:ef:aa

CLAG Interfaces
Our Interface      Peer Interface     CLAG Id   Conflicts              Proto-Down Reason
----------------   ----------------   -------   --------------------   -----------------
           bond1   -                  1         -                      -              
           bond2   -                  2         -                      -              
           bond3   -                  3         -                      -              
```

### Show All MLAG Settings

To see all MLAG settings, run the `clagctl params` command:

```
cumulus@leaf01:~$ clagctl params
clagVersion = 1.4.0
clagDataVersion = 1.4.0
clagCmdVersion = 1.1.0
peerIp = linklocal
peerIf = peerlink.4094
sysMac = 44:38:39:be:ef:aa
lacpPoll = 2
currLacpPoll = 2
peerConnect = 1
cmdConnect = 1
peerLinkPoll = 1
switchdReadyTimeout = 120
reloadTimer = 300
periodicRun = 4
priority = 32768
quiet = False
debug = 0x0
verbose = False
log = syslog
vm = True
peerPort = 5342
peerTimeout = 20
initDelay = 100
sendTimeout = 30
sendBufSize = 65536
forceDynamic = False
dormantDisable = False
redirectEnable = False
redirect2Enable = True
backupIp = 10.10.10.2
backupVrf = None
backupPort = 5342
vxlanAnycast = None
neighSync = True
permanentMacSync = True
cmdLine = /usr/sbin/clagd --daemon linklocal peerlink.4094 44:38:39:BE:EF:AA --priority 32768 --backupIp 10.10.10.2 --initDelay 100
peerlinkLearnEnable = False
```

### View the MLAG Log File

By default, when running, the `clagd` service logs status messages to the `/var/log/clagd.log` file and to `syslog`. Example log file output is shown below:

```
cumulus@spine01:~$ sudo tail /var/log/clagd.log
2016-10-03T20:31:50.471400+00:00 spine01 clagd[1235]: Initial config loaded
2016-10-03T20:31:52.479769+00:00 spine01 clagd[1235]: The peer switch is active.
2016-10-03T20:31:52.496490+00:00 spine01 clagd[1235]: Initial data sync to peer done.
2016-10-03T20:31:52.540186+00:00 spine01 clagd[1235]: Role is now primary; elected
2016-10-03T20:31:54.250572+00:00 spine01 clagd[1235]: HealthCheck: role via backup is primary
2016-10-03T20:31:54.252642+00:00 spine01 clagd[1235]: HealthCheck: backup active
2016-10-03T20:31:54.537967+00:00 spine01 clagd[1235]: Initial data sync from peer done.
2016-10-03T20:31:54.538435+00:00 spine01 clagd[1235]: Initial handshake done.
2016-10-03T22:47:35.255317+00:00 spine01 clagd[1235]: leaf01-02 is now dual connected.
```

### Monitor the clagd Service

Due to the critical nature of the `clagd` service, `systemd` continuously monitors its status by receiving notify messages every 30 seconds. If the `clagd` service terminates or becomes unresponsive for any reason and `systemd` receives no messages after 60 seconds, `systemd` restarts the `clagd` service. `systemd` logs these failures in the `/var/log/syslog` file and, on the first failure, also generates a `cl-support`file.

Monitoring is configured and enabled automatically as long as the `clagd` service is enabled (the peer IP address (`clagd-peer-ip`), the MLAG system MAC address (`clagd-sys-mac`), and the backup IP address (`clagd-backup-ip`) are configured for an interface) and the `clagd` service is running. If you stop `clagd` with the `systemctl stop clagd.service` command, `clagd` monitoring also stops.

You can check if `clagd` is enabled and running with the `cl-service-summary` or the `systemctl status` command:

```
cumulus@leaf01:~$ cl-service-summary
Service cron               enabled    active
Service ssh                enabled    active
Service syslog             enabled    active
Service asic-monitor       enabled    inactive
Service clagd              enabled    active
...
```

```
cumulus@leaf01:~$ systemctl status clagd.service
 ● clagd.service - Cumulus Linux Multi-Chassis LACP Bonding Daemon
    Loaded: loaded (/lib/systemd/system/clagd.service; enabled)
    Active: active (running) since Fri 2021-06-11 16:17:19 UTC; 12min ago
        Docs: man:clagd(8)
    Main PID: 27078 (clagd)
    CGroup: /system.slice/clagd.service
            └─27078 /usr/bin/python3 /usr/sbin/clagd --daemon linklocal peerlink.4094 44:38:39:BE:EF:AA --priority 32768
```

### Large Packet Drops on the Peer Link Interface

A large volume of packet drops across one of the peer link interfaces can be expected. These drops serve to prevent looping of BUM (broadcast, unknown unicast, multicast) packets. When a packet is received across the peer link, if the destination lookup results in an egress interface that is a dual-connected bond, the switch does not forward the packet (to prevent loops). This results in a drop being recorded on the peer link.

To check packet drops across peer link interfaces, run the following command:

{{< tabs "TabID1547 ">}}
{{< tab "NCLU Commands ">}}

Run the `net show counters` command. The number of dropped packets is displayed in the `RX_DRP` column.

```
cumulus@leaf01:~$ net show counters

Kernel Interface table
Iface            MTU    RX_OK    RX_ERR    RX_DRP    RX_OVR    TX_OK    TX_ERR    TX_DRP    TX_OVR  Flg
-------------  -----  -------  --------  --------  --------  -------  --------  --------  --------  -----
bond1           9216        0         0         0         0      542         0         0         0  BMmU
bond2           9216        0         0         0         0      542         0         0         0  BMmU
bridge          9216        0         0         0         0       17         0         0         0  BMRU
eth0            1500     5497         0         0         0      933         0         0         0  BMRU
lo             65536     1328         0         0         0     1328         0         0         0  LRU
mgmt           65536      790         0         0         0        0         0        33         0  OmRU
peerlink        9216    23626         0       520         0    23665         0         0         0  BMmRU
peerlink.4094   9216     8013         0         0         0     8017         0         0         0  BMRU
swp1            9216        5         0         0         0      553         0         0         0  BMsRU
swp2            9216        3         0         0         0      552         0         0         0  BMsRU
swp49           9216    11822         0         0         0    11852         0         0         0  BMsRU
swp50           9216    11804         0         0         0    11841         0         0         0  BMsRU
swp51           9216        0         0         0         0      292         0         0         0  BMRU
```

{{< /tab >}}
{{< tab "Linux Commands ">}}

Run the `ethtool -S <interface>` command:

```
cumulus@leaf01:mgmt:~$ ethtool -S swp49
NIC statistics:
     rx_queue_0_packets: 136
     rx_queue_0_bytes: 36318
     rx_queue_0_drops: 0
     rx_queue_0_xdp_packets: 0
     rx_queue_0_xdp_tx: 0
     rx_queue_0_xdp_redirects: 0
     rx_queue_0_xdp_drops: 0
     rx_queue_0_kicks: 1
     tx_queue_0_packets: 200
     tx_queue_0_bytes: 44244
     tx_queue_0_xdp_tx: 0
     tx_queue_0_xdp_tx_drops: 0
     tx_queue_0_kicks: 195
```

{{< /tab >}}
{{< /tabs >}}

### Peer Link Interfaces and the protodown State

In addition to the standard UP and DOWN administrative states, an interface that is a member of an MLAG bond can also be in a `protodown` state. When MLAG detects a problem that might result in connectivity issues, it can put that interface into `protodown` state. Such connectivity issues include:

- When the peer link goes down but the peer switch is up (the backup link is active).
- When the bond is configured with an MLAG ID but the `clagd` service is not running (either deliberately stopped or crashes).
- When an MLAG-enabled node is booted or rebooted, the MLAG bonds are placed in a `protodown` state until the node establishes a connection to its peer switch, or five minutes have elapsed.

When an interface goes into a `protodown` state, it results in a local OPER DOWN (carrier down) on the interface.

To show an interface in `protodown` state, run the NCLU `net show bridge link` command or the Linux `ip link show` command. For example:

```
cumulus@leaf01:~$ net show bridge link
3: swp1 state DOWN: <NO-CARRIER,BROADCAST,MULTICAST,MASTER,UP> mtu 9216 master pfifo_fast master host-bond1 state DOWN mode DEFAULT qlen 500 protodown on
    link/ether 44:38:39:00:69:84 brd ff:ff:ff:ff:ff:ff
```

### LACP Partner MAC Address Duplicate or Mismatch

Cumulus Linux puts interfaces in a protodown state under the following conditions:

- When there is an LACP partner MAC address mismatch. For example if a bond comes up with a `clag-id` and the peer is using a bond with the same `clag-id` but a different LACP partner MAC address. The `clagctl` command output shows the protodown reason as a `partner-mac-mismatch`.

- When there is a duplicate LACP partner MAC address. For example, when there are multiple LACP bonds between the same two LACP endpoints. The `clagctl` command output shows the protodown reason as a `duplicate-partner-mac`.

  To prevent a bond from coming up when an MLAG bond with an LACP partner MAC address already in use comes up, use the `--clag-args --allowPartnerMacDup False` option. This option puts the slaves of that bond interface in a protodown state and the `clagctl` output shows the protodown reason as a `duplicate-partner-mac`.

After you make the necessary cable or configuration changes to avoid the protodown state and you want MLAG to reevaluate the LACP partners, use the `clagctl clearconflictstate` command to remove `duplicate-partner-mac` or `partner-mac-mismatch` from the protodowned bonds, allowing them to come back up.

## Related Information

- [MLAG Redundancy Scenarios]({{<ref "/knowledge-base/Configuration-and-Usage/Network-Interfaces/MLAG-Redundancy-Scenarios" >}})
- [Compare Traditional Bridge Mode to VLAN-aware Bridge Mode]({{<ref "/knowledge-base/Configuration-and-Usage/Network-Interfaces/Compare-Traditional-Bridge-Mode-to-VLAN-aware-Bridge-Mode" >}})
