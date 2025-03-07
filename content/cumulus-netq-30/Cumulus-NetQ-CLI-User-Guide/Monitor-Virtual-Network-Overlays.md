---
title: Monitor Virtual Network Overlays
author: Cumulus Networks
weight: 560
toc: 3
---

With NetQ, a network administrator can monitor virtual network components in the data center, including VXLAN and EVPN software constructs. NetQ provides the ability to:

- Manage virtual constructs: view the performance and status of VXLANs and EVPN
- Validate overlay communication paths

It helps answer questions such as:

- Is my overlay configured and operating correctly?
- Is my control plane configured correctly?
- Can device A reach device B?

## Monitor Virtual Extensible LANs

Virtual Extensible LANs (VXLANs) provide a way to create a virtual
network on top of layer 2 and layer 3 technologies. It is intended for
organizations, such as data centers, that require larger scale without
additional infrastructure and more flexibility than is available with
existing infrastructure equipment. With NetQ, you can monitor the
current and historical configuration and status of your VXLANs using the
following command:

```
netq [<hostname>] show vxlan [vni <text-vni>] [around <text-time>] [json]
netq show interfaces type vxlan [state <remote-interface-state>] [around <text-time>] [json]
netq <hostname> show interfaces type vxlan [state <remote-interface-state>] [around <text-time>] [count] [json]
netq [<hostname>] show events [level info|level error|level warning|level critical|level debug] type vxlan [between <text-time> and <text-endtime>] [json]
```

{{%notice note%}}
When entering a time value, you must include a numeric value *and* the unit of measure:

- **w**: week(s)
- **d**: day(s)
- **h**: hour(s)
- **m**: minute(s)
- **s**: second(s)
- **now**

For the `between` option, the start (`<text-time>`) and end time (`text-endtime>`) values can be entered as most recent first and least recent second, or vice versa. The values do not have to have the same unit of measure.
{{%/notice%}}

### View All VXLANs in Your Network

You can view a list of configured VXLANs for all devices, including the
VNI (VXLAN network identifier), protocol, address of associated VTEPs
(VXLAN tunnel endpoint), replication list, and the last time it was
changed. You can also view VXLAN information for a given device by
adding a hostname to the `show` command. You can filter the results by
VNI.

This example shows all configured VXLANs across the network. In this
network, there are three VNIs (13, 24, and 104001) associated with three
VLANs (13, 24, 4001), EVPN is the virtual protocol deployed, and the
configuration was last changed around 23 hours ago.

 ```
cumulus@switch:~$ netq show vxlan
Matching vxlan records:
Hostname          VNI        Protoc VTEP IP          VLAN   Replication List                    Last Changed
                                ol
----------------- ---------- ------ ---------------- ------ ----------------------------------- -------------------------
exit01            104001     EVPN   10.0.0.41        4001                                       Fri Feb  8 01:35:49 2019
exit02            104001     EVPN   10.0.0.42        4001                                       Fri Feb  8 01:35:49 2019
leaf01            13         EVPN   10.0.0.112       13     10.0.0.134(leaf04, leaf03)          Fri Feb  8 01:35:49 2019
leaf01            24         EVPN   10.0.0.112       24     10.0.0.134(leaf04, leaf03)          Fri Feb  8 01:35:49 2019
leaf01            104001     EVPN   10.0.0.112       4001                                       Fri Feb  8 01:35:49 2019
leaf02            13         EVPN   10.0.0.112       13     10.0.0.134(leaf04, leaf03)          Fri Feb  8 01:35:49 2019
leaf02            24         EVPN   10.0.0.112       24     10.0.0.134(leaf04, leaf03)          Fri Feb  8 01:35:49 2019
leaf02            104001     EVPN   10.0.0.112       4001                                       Fri Feb  8 01:35:49 2019
leaf03            13         EVPN   10.0.0.134       13     10.0.0.112(leaf02, leaf01)          Fri Feb  8 01:35:49 2019
leaf03            24         EVPN   10.0.0.134       24     10.0.0.112(leaf02, leaf01)          Fri Feb  8 01:35:49 2019
leaf03            104001     EVPN   10.0.0.134       4001                                       Fri Feb  8 01:35:49 2019
leaf04            13         EVPN   10.0.0.134       13     10.0.0.112(leaf02, leaf01)          Fri Feb  8 01:35:49 2019
leaf04            24         EVPN   10.0.0.134       24     10.0.0.112(leaf02, leaf01)          Fri Feb  8 01:35:49 2019
leaf04            104001     EVPN   10.0.0.134       4001                                       Fri Feb  8 01:35:49 2019
```

This example shows the events and configuration changes that have
occurred on the VXLANs in your network in the last 24 hours. In this
case, the EVPN configuration was added to each of the devices in the
last 24 hours.

 ```
cumulus@switch:~$ netq show events type vxlan between now and 24h
Matching vxlan records:
Hostname          VNI        Protoc VTEP IP          VLAN   Replication List                    DB State   Last Changed
                                ol
----------------- ---------- ------ ---------------- ------ ----------------------------------- ---------- -------------------------
exit02            104001     EVPN   10.0.0.42        4001                                       Add        Fri Feb  8 01:35:49 2019
exit02            104001     EVPN   10.0.0.42        4001                                       Add        Fri Feb  8 01:35:49 2019
exit02            104001     EVPN   10.0.0.42        4001                                       Add        Fri Feb  8 01:35:49 2019
exit02            104001     EVPN   10.0.0.42        4001                                       Add        Fri Feb  8 01:35:49 2019
exit02            104001     EVPN   10.0.0.42        4001                                       Add        Fri Feb  8 01:35:49 2019
exit02            104001     EVPN   10.0.0.42        4001                                       Add        Fri Feb  8 01:35:49 2019
exit02            104001     EVPN   10.0.0.42        4001                                       Add        Fri Feb  8 01:35:49 2019
exit01            104001     EVPN   10.0.0.41        4001                                       Add        Fri Feb  8 01:35:49 2019
exit01            104001     EVPN   10.0.0.41        4001                                       Add        Fri Feb  8 01:35:49 2019
exit01            104001     EVPN   10.0.0.41        4001                                       Add        Fri Feb  8 01:35:49 2019
exit01            104001     EVPN   10.0.0.41        4001                                       Add        Fri Feb  8 01:35:49 2019
exit01            104001     EVPN   10.0.0.41        4001                                       Add        Fri Feb  8 01:35:49 2019
exit01            104001     EVPN   10.0.0.41        4001                                       Add        Fri Feb  8 01:35:49 2019
exit01            104001     EVPN   10.0.0.41        4001                                       Add        Fri Feb  8 01:35:49 2019
exit01            104001     EVPN   10.0.0.41        4001                                       Add        Fri Feb  8 01:35:49 2019
leaf04            104001     EVPN   10.0.0.134       4001                                       Add        Fri Feb  8 01:35:49 2019
leaf04            104001     EVPN   10.0.0.134       4001                                       Add        Fri Feb  8 01:35:49 2019
leaf04            104001     EVPN   10.0.0.134       4001                                       Add        Fri Feb  8 01:35:49 2019
leaf04            104001     EVPN   10.0.0.134       4001                                       Add        Fri Feb  8 01:35:49 2019
leaf04            104001     EVPN   10.0.0.134       4001                                       Add        Fri Feb  8 01:35:49 2019
leaf04            104001     EVPN   10.0.0.134       4001                                       Add        Fri Feb  8 01:35:49 2019
leaf04            104001     EVPN   10.0.0.134       4001                                       Add        Fri Feb  8 01:35:49 2019
leaf04            13         EVPN   10.0.0.134       13     10.0.0.112()                        Add        Fri Feb  8 01:35:49 2019
leaf04            13         EVPN   10.0.0.134       13     10.0.0.112()                        Add        Fri Feb  8 01:35:49 2019
leaf04            13         EVPN   10.0.0.134       13     10.0.0.112()                        Add        Fri Feb  8 01:35:49 2019
leaf04            13         EVPN   10.0.0.134       13     10.0.0.112()                        Add        Fri Feb  8 01:35:49 2019
leaf04            13         EVPN   10.0.0.134       13     10.0.0.112()                        Add        Fri Feb  8 01:35:49 2019
leaf04            13         EVPN   10.0.0.134       13     10.0.0.112()                        Add        Fri Feb  8 01:35:49 2019
leaf04            13         EVPN   10.0.0.134       13     10.0.0.112()                        Add        Fri Feb  8 01:35:49 2019
...
```

Consequently, if you looked for the VXLAN configuration and status for
last week, you would find either another configuration or no
configuration. This example shows that no VXLAN configuration was
present.

```
cumulus@switch:~$ netq show vxlan around 7d
No matching vxlan records found
```

You can filter the list of VXLANs to view only those associated with a
particular VNI. The VNI option lets you specify single VNI (100), a range of VNIs (10-100), or provide a comma-separated list (10,11,12). This example shows the configured VXLANs for *VNI 24*.

 ```
cumulus@switch:~$ netq show vxlan vni 24
Matching vxlan records:
Hostname          VNI        Protoc VTEP IP          VLAN   Replication List                    Last Changed
                                ol
----------------- ---------- ------ ---------------- ------ ----------------------------------- -------------------------
leaf01            24         EVPN   10.0.0.112       24     10.0.0.134(leaf04, leaf03)          Fri Feb  8 01:35:49 2019
leaf02            24         EVPN   10.0.0.112       24     10.0.0.134(leaf04, leaf03)          Fri Feb  8 01:35:49 2019
leaf03            24         EVPN   10.0.0.134       24     10.0.0.112(leaf02, leaf01)          Fri Feb  8 01:35:49 2019
leaf04            24         EVPN   10.0.0.134       24     10.0.0.112(leaf02, leaf01)          Fri Feb  8 01:35:49 2019
```

### View the Interfaces Associated with VXLANs

You can view detailed information about the VXLAN interfaces using the
`netq show interface` command. You can also view this information for a
given device by adding a hostname to the `show` command. This example
shows the detailed VXLAN interface information for the *leaf02* switch.

 ```
cumulus@switch:~$ netq leaf02 show interfaces type vxlan
Matching link records:
Hostname          Interface                 Type             State      VRF             Details                             Last Changed
----------------- ------------------------- ---------------- ---------- --------------- ----------------------------------- -------------------------
leaf02            vni13                     vxlan            up         default         VNI: 13, PVID: 13, Master: bridge,  Fri Feb  8 01:35:49 2019
                                                                                        VTEP: 10.0.0.112, MTU: 9000
leaf02            vni24                     vxlan            up         default         VNI: 24, PVID: 24, Master: bridge,  Fri Feb  8 01:35:49 2019
                                                                                        VTEP: 10.0.0.112, MTU: 9000
leaf02            vxlan4001                 vxlan            up         default         VNI: 104001, PVID: 4001,            Fri Feb  8 01:35:49 2019
                                                                                        Master: bridge, VTEP: 10.0.0.112,
                                                                                        MTU: 1500
```

## Monitor EVPN

EVPN (Ethernet Virtual Private Network) enables network administrators
in the data center to deploy a virtual layer 2 bridge overlay on top of
layer 3 IP networks creating access, or tunnel, between two locations.
This connects devices in different layer 2 domains or sites running
VXLANs and their associated underlays. With NetQ, you can monitor the
configuration and status of the EVPN setup using the `netq show evpn`
command. You can filter the EVPN information by a VNI (VXLAN network
identifier), and view the current information or for a time in the past.
The command also enables visibility into changes that have occurred in
the configuration during a specific time frame. The syntax for the
command is:

 ```
netq [<hostname>] show evpn [vni <text-vni>] [mac-consistency] [around <text-time>] [json]
netq [<hostname>] show events [level info|level error|level warning|level critical|level debug] type evpn [between <text-time> and <text-endtime>] [json]
```

{{%notice note%}}
When entering a time value, you must include a numeric value *and* the unit of measure:

- **w**: week(s)
- **d**: day(s)
- **h**: hour(s)
- **m**: minute(s)
- **s**: second(s)
- **now**

For the `between` option, the start (`<text-time>`) and end time (`text-endtime>`) values can be entered as most recent first and least recent second, or vice versa. The values do not have to have the same unit of measure.
{{%/notice%}}

For more information about and configuration of EVPN in your data center, refer to the [Cumulus Linux EVPN]({{<ref "/cumulus-linux-43/Network-Virtualization/Ethernet-Virtual-Private-Network-EVPN" >}}) topic.

### View the Status of EVPN

You can view the configuration and status of your EVPN overlay across
your network or for a particular device. This example shows the
configuration and status for all devices, including the associated VNI,
VTEP address, the import and export route (showing the BGP ASN and VNI
path), and the last time a change was made for each device running EVPN.
Use the `hostname` option to view the configuration and status for a
single device.

 ```
cumulus@switch:~$ netq show evpn
Matching evpn records:
Hostname          VNI        VTEP IP          In Kernel Export RT        Import RT        Last Changed
----------------- ---------- ---------------- --------- ---------------- ---------------- -------------------------
leaf01            33         27.0.0.22        yes       197:33           197:33           Fri Feb  8 01:48:27 2019
leaf01            34         27.0.0.22        yes       197:34           197:34           Fri Feb  8 01:48:27 2019
leaf01            35         27.0.0.22        yes       197:35           197:35           Fri Feb  8 01:48:27 2019
leaf01            36         27.0.0.22        yes       197:36           197:36           Fri Feb  8 01:48:27 2019
leaf01            37         27.0.0.22        yes       197:37           197:37           Fri Feb  8 01:48:27 2019
leaf01            38         27.0.0.22        yes       197:38           197:38           Fri Feb  8 01:48:27 2019
leaf01            39         27.0.0.22        yes       197:39           197:39           Fri Feb  8 01:48:27 2019
leaf01            40         27.0.0.22        yes       197:40           197:40           Fri Feb  8 01:48:27 2019
leaf01            41         27.0.0.22        yes       197:41           197:41           Fri Feb  8 01:48:27 2019
leaf01            42         27.0.0.22        yes       197:42           197:42           Fri Feb  8 01:48:27 2019
leaf02            33         27.0.0.23        yes       198:33           198:33           Thu Feb  7 18:31:41 2019
leaf02            34         27.0.0.23        yes       198:34           198:34           Thu Feb  7 18:31:41 2019
leaf02            35         27.0.0.23        yes       198:35           198:35           Thu Feb  7 18:31:41 2019
leaf02            36         27.0.0.23        yes       198:36           198:36           Thu Feb  7 18:31:41 2019
leaf02            37         27.0.0.23        yes       198:37           198:37
...
```

### View the Status of EVPN for a Given VNI

You can filter the full device view to focus on a single VNI. This
example only shows the EVPN configuration and status for VNI *42*.

 ```
cumulus@switch:~$ netq show evpn vni 42
Matching evpn records:
Hostname          VNI        VTEP IP          In Kernel Export RT        Import RT        Last Changed
----------------- ---------- ---------------- --------- ---------------- ---------------- -------------------------
leaf01            42         27.0.0.22        yes       197:42           197:42           Thu Feb 14 00:48:24 2019
leaf02            42         27.0.0.23        yes       198:42           198:42           Wed Feb 13 18:14:49 2019
leaf11            42         36.0.0.24        yes       199:42           199:42           Wed Feb 13 18:14:22 2019
leaf12            42         36.0.0.24        yes       200:42           200:42           Wed Feb 13 18:14:27 2019
leaf21            42         36.0.0.26        yes       201:42           201:42           Wed Feb 13 18:14:33 2019
leaf22            42         36.0.0.26        yes       202:42           202:42           Wed Feb 13 18:14:37 2019
```

### View EVPN Events

You can view status and configuration change events for the EVPN
protocol service using the `netq show events` command. This example
shows the events that have occurred in the last 48 hours.

```
cumulus@switch:/$ netq show events type evpn between now and 48h
Matching events records:
Hostname          Message Type Severity Message                             Timestamp
----------------- ------------ -------- ----------------------------------- -------------------------
torc-21           evpn         info     VNI 33 state changed from down to u 1d:8h:16m:29s
                                        p
torc-12           evpn         info     VNI 41 state changed from down to u 1d:8h:16m:35s
                                        p
torc-11           evpn         info     VNI 39 state changed from down to u 1d:8h:16m:41s
                                        p
tor-1             evpn         info     VNI 37 state changed from down to u 1d:8h:16m:47s
                                        p
tor-2             evpn         info     VNI 42 state changed from down to u 1d:8h:16m:51s
                                        p
torc-22           evpn         info     VNI 39 state changed from down to u 1d:8h:17m:40s
                                        p
...
```
