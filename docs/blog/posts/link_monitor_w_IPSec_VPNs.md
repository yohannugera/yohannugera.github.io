---
draft: false
date: 2026-01-23
categories:
  - FortiGate
  - IPSec
slug: Link Monitor with IPSec VPNs
---

# Link Monitor with IPSec VPNs

So, instead of running BGP, what if we run a "link-monitor" for the VPN and if the monitor goes down, we take down the
tunnel and sent the traffic to DR site. Problem is like, what if DR is not ready or it's an issue with probe? but that's
something we've to live with.

## Configuration

Configuration of IPSec VPNs are straightforward. Just to make things easier on the routing-front, I put "0.0.0.0/0" as
my remote/local encryption domains.

```console title="Customer-FW VPN configuration"
Customer-FW # show vpn ipsec phase1-interface
config vpn ipsec phase1-interface
    edit "3rd-Party-DC"
        set interface "port2"
        set ike-version 2
        set peertype any
        set net-device disable
        set proposal aes256gcm-prfsha256
        set dhgrp 14
        set transport auto
        set remote-gw 10.113.4.188
        set psksecret <<removed>>
    next
    edit "3rd-Party-DR"
        set interface "port2"
        set ike-version 2
        set peertype any
        set net-device disable
        set proposal aes256gcm-prfsha256
        set dhgrp 14
        set transport auto
        set remote-gw 10.115.4.38
        set psksecret <<removed>>
    next
end

Customer-FW # show vpn ipsec phase2-interface
config vpn ipsec phase2-interface
    edit "3rd-Party-DC"
        set phase1name "3rd-Party-DC"
        set proposal aes256gcm
        set dhgrp 14
    next
    edit "3rd-Party-DR"
        set phase1name "3rd-Party-DR"
        set proposal aes256gcm
        set dhgrp 14
    next
end

Customer-FW # show router static
config router static
    edit 4
        set dst 10.0.0.135 255.255.255.255
        set device "3rd-Party-DC"
    next
    edit 5
        set dst 10.0.0.135 255.255.255.255
        set distance 20
        set device "3rd-Party-DR"
    next
end
```

Remote-end configuration is somewhat similar, except for the fact that they don't have 2 VPN tunnels.

```console title="3rd Party FW configuration"
3rd-Party-DC # show vpn ipsec phase1-interface
config vpn ipsec phase1-interface
    edit "Customer-FW"
        set interface "port2"
        set ike-version 2
        set peertype any
        set net-device disable
        set proposal aes256gcm-prfsha256
        set dhgrp 14
        set transport auto
        set remote-gw 10.111.4.215
        set psksecret <<removed>>
    next
end

3rd-Party-DC # show vpn ipsec phase2-interface
config vpn ipsec phase2-interface
    edit "Customer-FW"
        set phase1name "Customer-FW"
        set proposal aes256gcm
        set dhgrp 14
    next
end

3rd-Party-DC # show router static
config router static
    edit 3
        set status disable
        set dst 10.0.0.111 255.255.255.255
        set device "Customer-FW"
    next
end
```

Now, I'll configure a link-monitor on my customer-FW to identify if the main tunnel goes down. If it goes down, I want
the VPN to go down and make secondary VPN tunnel take over.

```console title="link-monitor config"
Customer-FW # show system link-monitor
config system link-monitor
    edit "3rd-Party-DC"
        set srcintf "3rd-Party-DC"
        set server "10.0.0.135"
        set source-ip 10.0.0.111
    next
end
```

## Testing & Validation

Super simple, I'll just probe "10.0.0.135". Is it working? let's see.

```console title="Customer-FW routing-check"
Customer-FW # get router info routing-table details
Codes: K - kernel, C - connected, S - static, R - RIP, B - BGP
       O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, L1 - IS-IS level-1, L2 - IS-IS level-2, ia - IS-IS inter area
       V - BGP VPNv4
       * - candidate default

Routing table for VRF=0
S       10.0.0.135/32 [10/0] via 3rd-Party-DC tunnel 10.113.4.188, [1/0]
S       10.0.1.0/30 [5/0] via 3rd-Party-DC tunnel 10.113.4.188, [1/0]
C       10.0.1.1/32 is directly connected, 3rd-Party-DC
S       10.0.2.0/30 [5/0] via 3rd-Party-DR tunnel 10.115.4.38, [1/0]
C       10.0.2.1/32 is directly connected, 3rd-Party-DR
```

OK, "10.0.0.135" route is towards DC. Let's see if it's really sending probes.

```console
3rd-Party-DC # diag sniff packet any "host 10.0.0.111 and icmp" 4
Using Original Sniffing Mode
interfaces=[any]
filters=[host 10.0.0.111 and icmp]
0.270224 Customer-FW in 10.0.0.111 -> 10.0.0.135: icmp: echo request
0.270260 Customer-FW out 10.0.0.135 -> 10.0.0.111: icmp: echo reply
```

yes it is. Cool. Now I'll remove the static route from 3rd Party DC FW so that the probes will go unanswered.

```console
Customer-FW # get router info routing-table details
Codes: K - kernel, C - connected, S - static, R - RIP, B - BGP
       O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, L1 - IS-IS level-1, L2 - IS-IS level-2, ia - IS-IS inter area
       V - BGP VPNv4
       * - candidate default

Routing table for VRF=0
S       10.0.0.135/32 [20/0] via 3rd-Party-DR tunnel 10.115.4.38, [1/0]
S       10.0.1.0/30 [5/0] via 3rd-Party-DC tunnel 10.113.4.188, [1/0]
C       10.0.1.1/32 is directly connected, 3rd-Party-DC
S       10.0.2.0/30 [5/0] via 3rd-Party-DR tunnel 10.115.4.38, [1/0]
C       10.0.2.1/32 is directly connected, 3rd-Party-DC
```

OK. Routing got changed too. Let's see the link status now.

```console
Customer-FW # diag sys link-monitor status

Link Monitor: 3rd-Party-DC, Status: dead, Server num(1), cfg_version=0 HA state: local(dead), shared(dead)
Flags=0x9 init log_downgateway, Create time: Thu Jan 22 23:42:12 2026
Source interface: cameron-kvm52 (23)
VRF: 0
Source IP: 10.0.0.111
Interval: 500 ms
Service-detect: disable
Diffservcode: 000000
Class-ID: 0
Transport-Group: 0
Class-ID: 0
  Peer: 10.0.0.135(10.0.0.135)
        Source IP(10.0.0.111)
        Route: 10.0.0.111->10.0.0.135/32, gwy(10.113.4.188)
        protocol: ping, state: dead
                Packet lost: 100.000%
                MOS: 4.350
                Number of out-of-sequence packets: 0
                Recovery times(0/5) Fail Times(2/5)
                Packet sent: 4266, received: 800, Sequence(sent/rcvd/exp): 4267/3999/4000
```

I see that the **Status: dead**. If I enable it now, how it'll show & does the setup auto-heal?

```console
Link Monitor: 3rd-Party-DC, Status: alive, Server num(1), cfg_version=0 HA state: local(alive), shared(alive)
Flags=0x1 init, Create time: Thu Jan 22 23:42:12 2026
Source interface: cameron-kvm52 (23)
VRF: 0
Source IP: 10.0.0.111
Interval: 500 ms
Service-detect: disable
Diffservcode: 000000
Class-ID: 0
Transport-Group: 0
Class-ID: 0
  Peer: 10.0.0.135(10.0.0.135)
        Source IP(10.0.0.111)
        Route: 10.0.0.111->10.0.0.135/32, gwy(10.113.4.188)
        protocol: ping, state: alive
                Latency(Min/Max/Avg): 0.502/0.729/0.610 ms
                Jitter(Min/Max/Avg): 0.000/0.144/0.072 ms
                Packet lost: 89.000%
                MOS: 4.356
                Number of out-of-sequence packets: 0
                Fail Times(0/5)
                Packet sent: 4410, received: 811, Sequence(sent/rcvd/exp): 4411/4411/4412

Customer-FW # get router info routing-table details
Codes: K - kernel, C - connected, S - static, R - RIP, B - BGP
       O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, L1 - IS-IS level-1, L2 - IS-IS level-2, ia - IS-IS inter area
       V - BGP VPNv4
       * - candidate default

Routing table for VRF=0
S       10.0.0.135/32 [10/0] via 3rd-Party-DC tunnel 10.113.4.188, [1/0]
S       10.0.1.0/30 [5/0] via 3rd-Party-DC tunnel 10.113.4.188, [1/0]
C       10.0.1.1/32 is directly connected, 3rd-Party-DC
S       10.0.2.0/30 [5/0] via 3rd-Party-DR tunnel 10.115.4.38, [1/0]
C       10.0.2.1/32 is directly connected, 3rd-Party-DR
```

That's it with my testing.