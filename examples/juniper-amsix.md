## Juniper

 - Suggested by: Stavros Konstantaras - AMS-IX
 - Verified by: 

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
