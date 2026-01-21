---
draft: true
date: 2026-01-21
categories:
  - FortiGate
  - IPSec
slug: Site-to-MultiSite VPN Setup with BGP
---

## Requirement

My customer wanted to configure two VPNs with a 3rd party, one with their primary site and another with their DR. They
wanted to use BGP to advertise their subnet to us dynamically, so that we don't have to configure multiple static
routes (with different admin-distances), plus, static-routing makes the setup a bit rigid.

## Test Setup

I will configure three FortiGates, one will be my customer's FW and other 2 will be our 3rd party devices. 

![img.png](images/pub_ip_underlay.png)

So I have this 3 FWs (actually, to facilitate routing between these FWs, I have another FW in middle) and I have
configured 3 IPs on their public-facing interface. With routing established, I'll move on and configure the VPNs.

On Customer FW, I have configured two VPNs and policies to match traffic.

```bash
```