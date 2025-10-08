# IXP-BGP-Propagation-config



[TOC]

This repository is related to a security issue identified by DE-CIX.  Currently (as of 27.05.2025) the process of responsible disclosure is still in its early stages.

This document contains suggested configurations to mitigate the problem, and IXPs are encouraged to apply these to their switches and route servers.  These configurations need to be tested in your specific environment.  If you encounter any issue, please let the authors know.

Work on this document is coordinated by the Euro-IX Secretariat, helping DE-CIX in the process.  

if you: 

- Would like to add a vendor configuration;
- Would like to suggest a new vendor;
- Would like to improve the solutions presented.

please reach out to secretariat@euro-ix.net.

# Introduction

The configuration suggestion centers around ACLs.

There are two possible locations in the network where the ACL can be applied. Each scenario has its own advantages and disadvantages.

1. on the egress interface towards the route server
   - advantages:
     - legitimate customer traffic is never touched
     - ACL needs to be applied only on the route server port
   - disadvantages:
     - illegitimate traffic crosses the whole network before being discarded
     - customer routers are still succeptible, only the route server is protected
2. on the ingress interface towards each customer router
   - advantages:
     - illegitimate traffic is discarded as soon as it enters the network
     - customer routers are protected aswell
   - disadvantages:
     - mistakes can lead to legitimate customer traffic being discarded
     - ACL needs to be applied on each customer port

# Effect

Here is a pseudo code implementation of the ACL with explanations for each rule. It assumes that scenario 1 was chosen. The example ACLs are based on this as well.

- IPv4:

  ```
  // allow ping/traceroute within the peering LAN
  10 allow icmp peering_lan -> peering_lan
  
  // allow BGP (179) as the source within the peering LAN
  30 allow tcp peering_lan:bgp -> peering_lan:*
  // allow BGP (179) as the destination within the peering LAN
  31 allow tcp peering_lan:* -> peering_lan:bgp
  
  // allows BFD (3784) as the source within the peering LAN
  40 allow udp peering_lan:bfd -> peering_lan:*
  // allow BFD (3784) as the destination within the peering LAN
  41 allow udp peering_lan:* -> peering_lan:bfd
  
  // drop everything else
  50 drop any any
  ```

- IPv6:

  ```
  // allow ping/traceroute within the peering LAN
  10 allow icmp peering_lan -> peering_lan
  // allow ping/traceroute from the peering LAN to link-local
  11 allow icmp peering_lan -> link-local
  // allow neighbor discovery from the peering LAN
  12 allow icmp peering_lan -> multicast
  
  // allow ping/traceroute from link-local to the peering LAN
  20 allow icmp link-local -> peering_lan
  // allow ping/traceroute within link-local
  21 allow icmp link-local -> link-local
  // allow neighbor discovery from link-local
  22 allow icmp link-local -> multicast
  
  // allow BGP (179) as the source within the peering LAN
  30 allow tcp peering_lan:bgp -> peering_lan:*
  // allow BGP (179) as the destination within the peering LAN
  31 allow tcp peering_lan:* -> peering_lan:bgp
  
  // allows BFD (3784) as the source within the peering LAN
  40 allow udp peering_lan:bfd -> peering_lan:*
  // allow BFD (3784) as the destination within the peering LAN
  41 allow udp peering_lan:* -> peering_lan:bfd
  
  // drop everything else
  50 drop any any
  ```

This implies that BGP/BFD is not used over IPv6 link-local addresses.

# Test Cases

In this setup, scenario 1 was used for testing. The test cases include before and after tests (to make sure that the ACL did actually make a difference). 
From a host on a network outside of the peering LAN...

- the following tests were conducted:
  - open a raw TCP socket on the route server on port 179 (BGP) from a customer router
  - ping the route server from a customer router using ICMP
  - traceroute to the route server from a customer router and make sure that the hops are correct
- the following tests were **not** conducted:
  - customer to customer connectivitiy
    - in scenario 1, the ACL should never come into contact with any legitimate customer traffic
    - therefore it makes no sense to test for this here
  - open a raw UDP socket on the route server on port 3784 (BFD) from a customer router
    - detecting if a UDP port is open or closed is not trivial
    - since the syntax is the same for TCP ports, it is assumed to also work the same
    - therefore it is implicitely tested through the BGP test
  - ARP
    - implicitely tested through general IPv4 connectivity
  - IPv6 neighbor discovery
    - implicitely tested through general IPv6 connectivity

# Adjustments

The ACL was originally written for the DE-CIX Frankfurt peering LAN. Therefore you **MUST** adjust the address range to your location accordingly. This also means that the provided examples only serve as a common starting point. It is entirely possible, if not likely, that the ACL does not fit your use case as-is. Depending on what scenario you chose before, you **MAY** and in some cases even **MUST** make some further adjustments.

1. If you chose to apply the ACL on the egress interface towards the route server, you **MAY** use them as-is. You **MAY** also narrow down the source and destination address range to just the route server, because no other legitimate traffic should ever flow through that interface. In that case you also **MUST** duplicate every rule to allow the route server to be both on the sending and the receiving side. You **SHALL NOT** narrow down the address range to just the route server for both the source and destination in the same rule. Otherwise the route server will be isolated. For example, `allow peering_lan -> peering_lan` becomes:
   1. `allow peering_lan -> route_server`
   2. `allow route_server -> peering_lan`
2. If you chose to apply the ACL on the ingress interface towards each customer router, you **MUST** adjust it as follows. You **MAY** additionally narrow down the adress range as described above.
   1. First, remove the `drop any -> any` rule. Otherwise all customer traffic will be discarded.
   2. Then, allow traffic that is both originating from and destined to outside the peering LAN, while discarding traffic that is originating from or destined to the peering LAN. You can choose one of the following options, depending on if your vendor supports negation in ACLs:
      -	1. predefined rules
        2. `drop peering_lan -> any`
        3. `drop any -> peering_lan`
        4. `allow any -> any`
      -	1. predefined rules
        2. `allow !peering_lan -> !peering_lan`
        3. `drop any -> any`

For both scenario, ICMP/ICMPv6 **MAY** be narrowed down to types used in `ping`/`traceroute` for unicast addresses. IPv6 unicast might additionally need duplicate address detection (DAD) and path MTU discovery (PMTUD). For IPv6 multicast, neighbor discovery (ND) is needed aswell, but not on the whole multicast address range.

# Limitations

- In any case, source IP spoofing is still possible if the customer routers do not implement any mitigation techniques, e.g. reverse path forwarding (RPF). Therefore it is also recommended to apply a rate limit with a sensible maximum bandwith. This should happen on a per customer/address basis in order to not affect other customers if one is misbehaving.
- There are also some special cases where, depending on your setup and which option you chose, it is not guaranteed that the ACL is sufficiently effective. These include, but are not limited to:
  - IPv4 mapped IPv6 addresses
  - adressless forwarding on an interface of a different address family
- Be careful to not apply the ACL on layer 2, as this might result in all frames getting dropped by default.
- If you use the peering LAN for your management traffic (you really shouldn't), this will cause further issues.



# Vendors

## Arista

We still do not have configuration suggestions for Arista.

Suggested by: DE-CIX

```
ip access-list rs_protection_v4
   10 permit icmp 80.81.192.0/21 80.81.192.0/21
   30 permit tcp 80.81.192.0/21 eq bgp 80.81.192.0/21
   31 permit tcp 80.81.192.0/21 80.81.192.0/21 eq bgp
   40 permit udp 80.81.192.0/21 eq bfd 80.81.192.0/21
   41 permit udp 80.81.192.0/21 80.81.192.0/21 eq bfd
   50 deny ip any any

ipv6 access-list rs_protection_v6
   10 permit icmpv6 2001:7f8::/64 2001:7f8::/64
   11 permit icmpv6 2001:7f8::/64 fe80::/10
   12 permit icmpv6 2001:7f8::/64 ff00::/8
   20 permit icmpv6 fe80::/10 2001:7f8::/64
   21 permit icmpv6 fe80::/10 fe80::/10
   22 permit icmpv6 fe80::/10 ff00::/8
   30 permit tcp 2001:7f8::/64 eq bgp 2001:7f8::/64
   31 permit tcp 2001:7f8::/64 2001:7f8::/64 eq bgp
   40 permit udp 2001:7f8::/64 eq bfd 2001:7f8::/64
   41 permit udp 2001:7f8::/64 2001:7f8::/64 eq bfd
   50 deny ipv6 any any

interface *
   ip access-group rs_protection_v4 out
   ipv6 access-group rs_protection_v6 out
```



## Cisco

Suggested by: DE-CIX

```
ipv4 access-list rs_protection_v4
 10 permit icmp 80.81.192.0/21 80.81.192.0/21
 30 permit tcp 80.81.192.0/21 eq bgp 80.81.192.0/21
 31 permit tcp 80.81.192.0/21 80.81.192.0/21 eq bgp
 40 permit udp 80.81.192.0/21 eq bfd 80.81.192.0/21
 41 permit udp 80.81.192.0/21 80.81.192.0/21 eq bfd
 50 deny ipv4 any any

ipv6 access-list rs_protection_v6
 10 permit icmpv6 2001:7f8::/64 2001:7f8::/64
 11 permit icmpv6 2001:7f8::/64 fe80::/10
 12 permit icmpv6 2001:7f8::/64 ff00::/8
 20 permit icmpv6 fe80::/10 2001:7f8::/64
 21 permit icmpv6 fe80::/10 fe80::/10
 22 permit icmpv6 fe80::/10 ff00::/8
 30 permit tcp 2001:7f8::/64 eq bgp 2001:7f8::/64
 31 permit tcp 2001:7f8::/64 2001:7f8::/64 eq bgp
 40 permit udp 2001:7f8::/64 eq bfd 2001:7f8::/64
 41 permit udp 2001:7f8::/64 2001:7f8::/64 eq bfd
 50 deny ipv6 any any

interface *
 ipv4 access-group rs_protection_v4 egress
 ipv6 access-group rs_protection_v6 egress
```

## Extreme

We still have no suggestion for Extreme.

## Juniper

Suggested by: Stavros Konstantaras - AMS-IX
Verified by: 

### STEP 1 - Define the prefix list that contains the Peering LAN IP address

```
set policy-options prefix-list PL-PEERING-LAN 80.249.208.0/21
```

### STEP 2 - Define the firewall filter that controls the incoming traffic 

#### STEP 2.1 Accept IPv4 ARP

```
set firewall family vpls filter IP-ERL-OUT-501 term accept-ipv4-arp from ether-type arp
set firewall family vpls filter IP-ERL-OUT-501 term accept-ipv4-arp then accept
```

#### STEP 2.2 Increase the CoS queue for the IPv4 BGP Traffic

```
set firewall family vpls filter IP-ERL-OUT-501 term increase-ipv4-bgp-priority from ether-type ipv4
set firewall family vpls filter IP-ERL-OUT-501 term increase-ipv4-bgp-priority from ip-protocol tcp
set firewall family vpls filter IP-ERL-OUT-501 term increase-ipv4-bgp-priority from port bgp
set firewall family vpls filter IP-ERL-OUT-501 term increase-ipv4-bgp-priority then forwarding-class network-control
set firewall family vpls filter IP-ERL-OUT-501 term increase-ipv4-bgp-priority then next term
```

#### STEP 2.3 - Rate limit all IPv4 traffic to 1Gbps

```
set firewall family vpls filter IP-ERL-OUT-501 term rate-limit-valid-ipv4-traffic from ether-type ipv4
set firewall family vpls filter IP-ERL-OUT-501 term rate-limit-valid-ipv4-traffic from ip-destination-address 80.249.208.255/32
set firewall family vpls filter IP-ERL-OUT-501 term rate-limit-valid-ipv4-traffic from ip-protocol tcp
set firewall family vpls filter IP-ERL-OUT-501 term rate-limit-valid-ipv4-traffic from ip-protocol udp
set firewall family vpls filter IP-ERL-OUT-501 term rate-limit-valid-ipv4-traffic from ip-protocol icmp
set firewall family vpls filter IP-ERL-OUT-501 term rate-limit-valid-ipv4-traffic from source-prefix-list PL-PEERING-LAN
set firewall family vpls filter IP-ERL-OUT-501 term rate-limit-valid-ipv4-traffic then policer PC-1G
set firewall family vpls filter IP-ERL-OUT-501 term rate-limit-valid-ipv4-traffic then accept
```

#### STEP 2.4 - Accept IPv6-ND except IPv6-RAs

```
set firewall family vpls filter IP-ERL-OUT-501 term accept-ipv6-icmpnd from ether-type ipv6
set firewall family vpls filter IP-ERL-OUT-501 term accept-ipv6-icmpnd from icmp-type-except router-advertisement
set firewall family vpls filter IP-ERL-OUT-501 term accept-ipv6-icmpnd from ipv6-next-header icmp6
set firewall family vpls filter IP-ERL-OUT-501 term accept-ipv6-icmpnd then accept
```

#### STEP 2.5 Increase the CoS queue for the IPv6 BGP Traffic

```
set firewall family vpls filter IP-ERL-OUT-501 term increase-ipv6-bgp-priority from port bgp
set firewall family vpls filter IP-ERL-OUT-501 term increase-ipv6-bgp-priority from ipv6-next-header tcp
set firewall family vpls filter IP-ERL-OUT-501 term increase-ipv6-bgp-priority then forwarding-class network-control
set firewall family vpls filter IP-ERL-OUT-501 term increase-ipv6-bgp-priority then next term
```

#### STEP 2.6 Rate limit all IPv6 Traffic to 1Gbps

```
set firewall family vpls filter IP-ERL-OUT-501 term rate-limit-valid-ipv6-traffic from ether-type ipv6
set firewall family vpls filter IP-ERL-OUT-501 term  rate-limit-valid-ipv6-traffic from ipv6-destination-address  2001:7f8:1::a500:6777:1/128
set firewall family vpls filter IP-ERL-OUT-501 term rate-limit-valid-ipv6-traffic from ipv6-source-prefix-list PL-PEERING-LAN-V6
set firewall family vpls filter IP-ERL-OUT-501 term rate-limit-valid-ipv6-traffic then policer PC-1G
set firewall family vpls filter IP-ERL-OUT-501 term rate-limit-valid-ipv6-traffic then accept

```

#### STEP 2.7 Discard all other traffic

```
set firewall family vpls filter IP-ERL-OUT-501 term discard-all then discard
```

### STEP 3 Apply the filter to the interface

```
set interfaces xe-X/X/X:X unit YYY family vpls filter output IP-ERL-OUT-501
```



## Mikrotik

We still do not have configuration suggestions for Mikrotik.

## Nokia

### Nokia SR OS

Suggested by: Matthias Wichtlhuber - DE-CIX
Verified by: Greg Hankins - Nokia

```
qos
        sap-egress <egress id> name "rs_protection" create
            description "Egress RS Protection VPLS <VPLS>"
            queue 1 create
            exit
            queue 2 create
                rate 5000
            exit
            queue 4 create
                mbs 0 kilobytes
            exit
            fc af create
                queue 4
            exit
            fc be create
                queue 1
            exit
            fc l2 create
                queue 2
            exit
            ip-criteria
                entry 10 create
                    description "Allow ICMP between RS/Peers"
                    match protocol "icmp"
                        src-ip <Peering LAN Prefix IPv4>
                        dst-ip <Peering LAN Prefix IPv4>
                    exit
                    action fc "be"
                exit
                entry 40 create
                    description "Allow BGP between RS/Peers"
                    match protocol tcp
                        src-ip <Peering LAN Prefix IPv4>
                        dst-ip <Peering LAN Prefix IPv4>
                        src-port eq 179
                    exit
                    action fc "be"
                exit
                entry 41 create
                    description "Allow BGP between RS/Peers"
                    match protocol tcp
                        src-ip <Peering LAN Prefix IPv4>
                        dst-ip <Peering LAN Prefix IPv4>
                        dst-port eq 179
                    exit
                    action fc "be"
                exit
                entry 50 create
                    description "Allow BFD between RS/Peers"
                    match protocol udp
                        src-ip <Peering LAN Prefix IPv4>
                        dst-ip <Peering LAN Prefix IPv4>
                        src-port eq 3784
                    exit
                    action fc "be"
                exit
                entry 51 create
                    description "Allow BFD between RS/Peers"
                    match protocol udp
                        src-ip <Peering LAN Prefix IPv4>
                        dst-ip <Peering LAN Prefix IPv4>
                        dst-port eq 3784
                    exit
                    action fc "be"
                exit
                entry 1000 create
                    description "Catch-All and drop"
                    match
                    exit
                    action fc "af"
                exit
            exit
            ipv6-criteria
                entry 10 create
                    description "Allow link-local ICMPv6 NS/NA between RS/Peers"
                    match next-header ipv6-icmp
                        src-ip fe80::/10
                    exit
                    action fc "be"
                exit
                entry 11 create
                    description "Allow GUA sourced ICMPv6 NS/NA between RS/Peers"
                    match next-header ipv6-icmp
                        src-ip <Peering LAN Prefix IPv6>
                        dst-ip ff00::/8
                    exit
                    action fc "be"
                exit
                entry 12 create
                    description "Allow GUA ICMPv6 between RS/Peers"
                    match next-header ipv6-icmp
                        src-ip <Peering LAN Prefix IPv6>
                        dst-ip <Peering LAN Prefix IPv6>
                    exit
                    action fc "be"
                exit
                entry 40 create
                    description "Allow BGP between RS/Peers"
                    match next-header tcp
                        src-ip <Peering LAN Prefix IPv6>
                        dst-ip <Peering LAN Prefix IPv6>
                        src-port eq 179
                    exit
                    action fc "be"
                exit
                entry 41 create
                    description "Allow BGP between RS/Peers"
                    match next-header tcp
                        src-ip <Peering LAN Prefix IPv6>
                        dst-ip <Peering LAN Prefix IPv6>
                        dst-port eq 179
                    exit
                    action fc "be"
                exit
                entry 50 create
                    description "Allow BFD between RS/Peers"
                    match next-header udp
                        src-ip <Peering LAN Prefix IPv6>
                        dst-ip <Peering LAN Prefix IPv6>
                        src-port eq 3784
                    exit
                    action fc "be"
                exit
                entry 51 create
                    description "Allow BFD between RS/Peers"
                    match next-header udp
                        src-ip <Peering LAN Prefix IPv6>
                        dst-ip <Peering LAN Prefix IPv6>
                        dst-port eq 3784
                    exit
                    action fc "be"
                exit
                entry 1000 create
                    description "Catch-All and drop"
                    match
                    exit
                    action fc "af"
                exit
            exit
        exit
    exit
...
    service
        vpls <VPLS> customer 1 create
            sap <sap> create
                egress
                    qos <egress id>
                    queue-override
                        queue 1 create
                            rate <rate limit in kbps>
                        exit
                    exit
                    filter mac <mac filter id>
                exit
            exit
        exit
    exit

```

### Nokia SR Linux

Suggested by: DE-CIX

```
acl {
    acl-filter rs_protection_v4 type ipv4 {
        entry 10 {
            match {
                ipv4 {
                    source-ip {
                        prefix 80.81.192.0/21
                    }
                    destination-ip {
                        prefix 80.81.192.0/21
                    }
                    protocol icmp
                }
            }
            action {
                accept {
                }
            }
        }
        entry 30 {
            match {
                ipv4 {
                    source-ip {
                        prefix 80.81.192.0/21
                    }
                    destination-ip {
                        prefix 80.81.192.0/21
                    }
                    protocol tcp
                }
                transport {
                    source-port {
                        value bgp
                    }
                }
            }
            action {
                accept {
                }
            }
        }
        entry 31 {
            match {
                ipv4 {
                    source-ip {
                        prefix 80.81.192.0/21
                    }
                    destination-ip {
                        prefix 80.81.192.0/21
                    }
                    protocol tcp
                }
                transport {
                    destination-port {
                        value bgp
                    }
                }
            }
            action {
                accept {
                }
            }
        }
        entry 40 {
            match {
                ipv4 {
                    source-ip {
                        prefix 80.81.192.0/21
                    }
                    destination-ip {
                        prefix 80.81.192.0/21
                    }
                    protocol udp
                }
                transport {
                    source-port {
                        value bfd
                    }
                }
            }
            action {
                accept {
                }
            }
        }
        entry 41 {
            match {
                ipv4 {
                    source-ip {
                        prefix 80.81.192.0/21
                    }
                    destination-ip {
                        prefix 80.81.192.0/21
                    }
                    protocol udp
                }
                transport {
                    destination-port {
                        value bfd
                    }
                }
            }
            action {
                accept {
                }
            }
        }
        entry 50 {
            action {
                drop {
                }
            }
        }
    }
    acl-filter rs_protection_v6 type ipv6 {
        entry 10 {
            match {
                ipv6 {
                    source-ip {
                        prefix 2001:7f8::/64
                    }
                    destination-ip {
                        prefix 2001:7f8::/64
                    }
                    next-header icmp6
                }
            }
            action {
                accept {
                }
            }
        }
        entry 11 {
            match {
                ipv6 {
                    source-ip {
                        prefix 2001:7f8::/64
                    }
                    destination-ip {
                        prefix fe80::/10
                    }
                    next-header icmp6
                }
            }
            action {
                accept {
                }
            }
        }
        entry 12 {
            match {
                ipv6 {
                    source-ip {
                        prefix 2001:7f8::/64
                    }
                    destination-ip {
                        prefix ff00::/8
                    }
                }
                next-header icmp6
            }
            action {
                accept {
                }
            }
        }
        entry 20 {
            match {
                ipv6 {
                    source-ip {
                        prefix fe80::/10
                    }
                    destination-ip {
                        prefix 2001:7f8::/64
                    }
                    next-header icmp6
                }
            }
            action {
                accept {
                }
            }
        }
        entry 21 {
            match {
                ipv6 {
                    source-ip {
                        prefix fe80::/10
                    }
                    destination-ip {
                        prefix fe80::/10
                    }
                    next-header icmp6
                }
            }
            action {
                accept {
                }
            }
        }
        entry 22 {
            match {
                ipv6 {
                    source-ip {
                        prefix fe80::/10
                    }
                    destination-ip {
                        prefix ff00::/8
                    }
                }
                next-header icmp6
            }
            action {
                accept {
                }
            }
        }
        entry 30 {
            match {
                ipv6 {
                    source-ip {
                        prefix 2001:7f8::/64
                    }
                    destination-ip {
                        prefix 2001:7f8::/64
                    }
                    next-header tcp
                }
                transport {
                    source-port {
                        value bgp
                    }
                }
            }
            action {
                accept {
                }
            }
        }
        entry 31 {
            match {
                ipv6 {
                    source-ip {
                        prefix 2001:7f8::/64
                    }
                    destination-ip {
                        prefix 2001:7f8::/64
                    }
                    next-header tcp
                }
                transport {
                    destination-port {
                        value bgp
                    }
                }
            }
            action {
                accept {
                }
            }
        }
        entry 40 {
            match {
                ipv6 {
                    source-ip {
                        prefix 2001:7f8::/64
                    }
                    destination-ip {
                        prefix 2001:7f8::/64
                    }
                    next-header udp
                }
                transport {
                    source-port {
                        value bfd
                    }
                }
            }
            action {
                accept {
                }
            }
        }
        entry 41 {
            match {
                ipv6 {
                    source-ip {
                        prefix 2001:7f8::/64
                    }
                    destination-ip {
                        prefix 2001:7f8::/64
                    }
                    protocol udp
                }
                transport {
                    destination-port {
                        value bfd
                    }
                }
            }
            action {
                accept {
                }
            }
        }
        entry 50 {
            action {
                drop {
                }
            }
        }
    }
    interface * {
        output {
            acl-filter rs_protection_v4 type ipv4 {
            }
            acl-filter rs_protection_v6 type ipv6 {
            }
        }
    }
}
```



# Notes

- General:
  1. Using `traceroute` in ICMP mode (`-I`) for testing fails for some unkown reason, but only for IPv6 and only on the first hop. This happens regardless of the ACL.
  2. When using `traceroute`'s default UDP mode, replies to different traceroutes that are running simultaniously might get mixed up. This is because Python subprocesses are not network isolated against each other by default. This happens regardless of the ACL.
  3. Due to limitations of some vendors, the IPv4 and IPv6 ACLs cannot be tested at the same time automatically. Change [main.py#L55-L56](main.py#L55-L56), [main.py#L142](main.py#L142) and [main.py#L163](main.py#L163) inbetween tests accordingly.
- Cisco IOS XR:
  1. In order to test Cisco IOS XR, the XRD image is used. As this is a control plane only image, the data plane is handled by the underlying virtualization host, which the image has no control over anymore. This means that an ACL applied on an egress is not effective. To test the ACL, it is therefore applied on the ingress, since this is the only place where the traffic actually hits the image.
- Juniper Junos OS:
  1. Depending on what type of interface (`ethernet-switching`, `inet`/`inet6`, `mpls`, `vpls` etc.) you apply the ACL to, its syntax might vary a lot or not be compatible at all.
  2. Deployment of the config fails silently if the `inet` and `inet6` address families are configured for a BGP peer simultaneously (MP-BGP) and the peer address is of type `inet6`. Either disable the `inet` family or configure `bgp family inet unicast local-ipv4-address`. On the other hand, if the peer address is of type `inet`, the IPv6 next hop will be the IPv4 mapped address and not the address of the `inet6` interface.
- Nokia SROS:
  1. Deployment of the config fails silently if using MD CLI and `filter md-auto-id filter-id-range` is not configured.
