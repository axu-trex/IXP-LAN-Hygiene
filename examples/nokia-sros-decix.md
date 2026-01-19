## Nokia SR OS

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
