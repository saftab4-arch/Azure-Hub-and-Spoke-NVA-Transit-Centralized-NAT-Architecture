Azure Hub-and-Spoke NVA Transit Architecture

Centralized Egress Routing via Linux NVA & Azure NAT Gateway
Root Cause Analysis & Technical Blueprint

1. Architecture Overview

This lab implements a hub-and-spoke Azure network in which traffic from
a workload VM in the spoke is forced across VNet peering to a Linux
Network Virtual Appliance (NVA) in the hub. The Linux NVA forwards and
NATs the traffic, and the hub subnet's Azure NAT Gateway provides public
internet egress.

End-to-End Traffic Flow

Spoke VM
10.1.1.4
   |
   | UDR: 0.0.0.0/0 -> 10.3.1.4
   v
Spoke VNet (10.1.0.0/16)
   |
   | VNet Peering
   | Allow Forwarded Traffic
   v
Hub VNet (10.3.0.0/16)
   |
   v
Hub NSG
Allow-Spoke-Transit
Source: 10.1.0.0/16
Priority: 100
   |
   v
Linux NVA
10.3.1.4
IP Forwarding: ON
rp_filter: 0
iptables MASQUERADE
   |
   v
Azure NAT Gateway
   |
   v
Internet

2. Technical Network Specifications

Component         Resource Name             IP / Subnet Scope    Critical Configuration

Hub VNet          vnet-hub-southcentral   10.3.0.0/16        Contains snet-hub and
NAT Gateway

Hub NVA VM        vm-firewall-south       10.3.1.4           IP forwarding enabled,
(snet-hub)         rp_filter=0

Spoke VNet        vnet-spoke1-prod        10.1.0.0/16        Peered to Hub with
forwarded traffic allowed

Spoke Test VM     vm-spoke1-test          10.1.1.4           UDR attached:
(snet-workload1)   0.0.0.0/0 -> 10.3.1.4

Spoke

VNet:    vnet-spoke1-prod
CIDR:    10.1.0.0/16
Subnet:  snet-workload1
CIDR:    10.1.1.0/24
VM:      vm-spoke1-test
VM IP:   10.1.1.4

User-Defined Route

Route Table: rt-spoke-to-nva
Destination: 0.0.0.0/0
Next Hop:    10.3.1.4
Status:      Active

Hub

VNet:    vnet-hub-southcentral
CIDR:    10.3.0.0/16
Subnet:  snet-hub
CIDR:    10.3.1.0/24
NVA:     vm-firewall-south
NVA IP:  10.3.1.4

3. Linux NVA Kernel and Routing Configuration

The Linux NVA must be configured to forward transit traffic between the
spoke and the internet.

Enable IPv4 Forwarding

sudo sysctl -w net.ipv4.ip_forward=1

Disable Reverse Path Filtering

Reverse path filtering can drop transit traffic arriving from non-local
subnets, so it was disabled for this lab.

sudo sysctl -w net.ipv4.conf.all.rp_filter=0
sudo sysctl -w net.ipv4.conf.eth0.rp_filter=0
sudo sysctl -w net.ipv4.conf.default.rp_filter=0

Configure iptables NAT Masquerading

sudo iptables -F
sudo iptables -t nat -F
sudo iptables -P FORWARD ACCEPT
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

Persist Kernel Settings

Add the following to /etc/sysctl.conf:

net.ipv4.ip_forward=1
net.ipv4.conf.all.rp_filter=0

4. Troubleshooting and Root Cause Analysis

The Problem

Initial testing produced an interesting result:

ping 10.3.1.4

worked successfully from the spoke VM, while:

ping 8.8.8.8

failed.

At the same time, packet capture on the NVA showed no transit ICMP
packets:

sudo tcpdump -i eth0 icmp

This indicated that the traffic was being dropped before reaching the
Linux networking stack.

Direct Ping: Spoke -> Hub NVA

Source:      10.1.1.4
Destination: 10.3.1.4

Both IP addresses belong to private address spaces inside the peered
VNets.

The Hub VM's NSG allowed the traffic through Azure's default
AllowVnetInBound rule because the source and destination were within
the VirtualNetwork service-tag boundary.

Result:

Spoke VM -> VNet Peering -> Hub NSG -> NVA NIC
                                  PASS

The ping to 10.3.1.4 therefore succeeded.

5. Why Internet-Bound Transit Traffic Failed

The failing traffic looked like this:

Source:      10.1.1.4
Destination: 8.8.8.8

The spoke UDR correctly sent the packet toward the NVA at 10.3.1.4.

However, Azure evaluates the Hub VM's NSG before delivering the frame to
the Linux kernel.

Because the packet's destination was an external address (8.8.8.8), it
did not match the default AllowVnetInBound behavior described above.

The traffic eventually hit:

65500 DenyAllInBound

and Azure silently dropped it before it reached eth0.

This explained why:

sudo tcpdump -i eth0 icmp

showed no packets during the failed test.

6. Final NSG Solution

An explicit higher-priority inbound NSG rule was added to the NVA:

  Priority Name                    Source          Port      Destination   Protocol   Action

       100 `Allow-Spoke-Transit`   `10.1.0.0/16`   `*`       Any (`*`)     Any        Allow

The resulting flow became:

10.1.1.4
   |
   | UDR
   v
10.3.1.4
   |
   | NSG Priority 100: ALLOW
   v
Linux NVA eth0
   |
   | Kernel Forwarding
   | iptables MASQUERADE
   v
Azure NAT Gateway
   |
   v
Internet

7. Final Validation

After applying the NSG rule, packet capture immediately showed the spoke
traffic reaching the NVA:

sudo tcpdump -i eth0 icmp

Internet connectivity was then tested again:

ping -c 4 8.8.8.8

Result:

0% packet loss

The spoke workload now had working internet egress through the complete
path:

Spoke VM
 -> UDR
 -> VNet Peering
 -> Hub NSG
 -> Linux NVA
 -> iptables MASQUERADE
 -> Azure NAT Gateway
 -> Internet

8. Key Lessons

A successful ping to the NVA itself does not prove that transit
forwarding is working.

UDRs determine where the spoke sends traffic, but every security and
forwarding layer along the path still matters.

Azure NSGs can drop a transit packet before the Linux kernel sees
it.

tcpdump is extremely useful for determining whether a packet
actually reaches the NVA.

A Linux NVA requires IPv4 forwarding to operate as a router.

Reverse path filtering can interfere with transit routing and may
need to be disabled for this architecture.

iptables masquerading provides source NAT on the Linux NVA in this
design.

Azure NAT Gateway provides the final public egress path from the hub
subnet.

Hub-and-spoke troubleshooting should follow the packet one layer at
a time: UDR -> Peering -> NSG -> NIC -> Linux Kernel ->
iptables -> NAT Gateway -> Internet.

Project Outcome

This lab successfully demonstrated centralized Azure egress using a
hub-and-spoke topology, VNet peering, UDRs, a Linux NVA, NSG transit
rules, Linux IP forwarding, iptables NAT, and Azure NAT Gateway.

The most valuable part of the lab was troubleshooting the difference
between direct VNet connectivity and forwarded transit traffic,
then identifying the NSG as the point where internet-bound packets were
being dropped.
