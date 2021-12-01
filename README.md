# Ziggo-Edgerouter

This page outlines the steps needed to configure the Ziggo native IPv6
connectivity on a Ubiquity EdgeRouter. 

Assumptions:

- The EdgeRouter is directly connected to the cable modem (CM).
- The CM is in `bridge` mode. This can be requested via Customer Service
  (former Ziggo area) or via Mijn Ziggo (former UPC area).
- The CM is connected to `eth0` on the router.
- Internal networks are connected to `eth1`
- You have reset the cable modem before attempting the steps below.

## tl;dr

```
configure
set interfaces ethernet eth0 ipv6 address autoconf
set interfaces ethernet eth0 dhcpv6-pd pd 1 prefix-length 56
set interfaces ethernet eth0 dhcpv6-pd pd 1 interface eth1 host-address ::1
set interfaces ethernet eth0 dhcpv6-pd pd 1 interface eth1 prefix-id :0
set interfaces ethernet eth0 dhcpv6-pd pd 1 interface eth1 service dhcpv6-stateless
```

## Basic setup

Basic IPv6 connectivity from the router can be set up by adding the following
configuration options to `eth0`:

```
set interfaces ethernet eth0 ipv6 address autoconf
commit
```

You will now see that there is an IPv6 address active on interface `eth0`:

```
mtak@er1:~$ show interfaces ethernet     
Codes: S - State, L - Link, u - Up, D - Down, A - Admin Down
Interface    IP Address                        S/L  Description                 
---------    ----------                        ---  -----------                 
eth0         83.85.23.20/23                    u/u  Ziggo                       
             2001:1c00:2500:0:adb2:751c:23b7:8f77/128
```

as well as routes for IPv6:

```
mtak@er1:~$ show ipv6 route
[...]
IP Route Table for VRF "default"
K      ::/0 [0/1024] via fe80::201:5cff:fe70:9446, eth0, 01:36:32
C      ::1/128 via ::, lo, 01w1d05h
C      2001:1c00:2500:0:adb2:751c:23b7:8f77/128 via ::, eth0, 01:16:33
C      fe80::/64 via ::, eth0, 01:43:38
```

You should be able to ping Google DNS over IPv6 now:

```bash
mtak@er1:~$ ping 2001:4860:4860::8888
PING 2001:4860:4860::8888(2001:4860:4860::8888) 56 data bytes
64 bytes from 2001:4860:4860::8888: icmp_seq=1 ttl=119 time=13.4 ms
64 bytes from 2001:4860:4860::8888: icmp_seq=2 ttl=119 time=13.8 ms
^C
--- 2001:4860:4860::8888 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 13.443/13.634/13.826/0.224 ms
```

## Prefix delegation

Prefix delegation is a concept that is new in IPv6. It allows you to
automatically generate subnets on your router based on a delegated prefix from
your ISP. Ziggo will give you a `/56` prefix, and you want to split that up
in `/64`s for one or more subnets at home. You would be able to assign
2^(64-56)=256 subnets that way, each being a full `/64` standard subnet size.
Prefix delegation will automatically do that based on a number of configuration
options. The _delegation_ part of prefix delegation means that you automatically
get assigned a prefix, but it's delegated to your control.

### Single subnet

Let's look at an example:

```
set interfaces ethernet eth0 dhcpv6-pd pd 1 prefix-length 56
set interfaces ethernet eth0 dhcpv6-pd pd 1 interface eth1 host-address ::1
set interfaces ethernet eth0 dhcpv6-pd pd 1 interface eth1 prefix-id :0
set interfaces ethernet eth0 dhcpv6-pd pd 1 interface eth1 service dhcpv6-stateless
```

This will set up eth1 with the `0`th subnet and host address `1`:

```
mtak@er1:~$ show interfaces ethernet
Codes: S - State, L - Link, u - Up, D - Down, A - Admin Down
Interface    IP Address                        S/L  Description                 
---------    ----------                        ---  -----------                 
eth1         10.100.0.1/24                     u/u  Internal                    
             2001:1c00:2509:8200::1/64        
```

In this example, I got assigned the prefix `2001:1c00:2509:82`. I know I have 64
subnets in my prefix, so I should be able to have `/64`s in the range
`2001:1c00:2509:8200::/64` up to `2001:1c00:2509:82ff::/64` (because 0xff == 256).

With the configuration above, the router will automatically start assigning IPv6
addresses to hosts in the subnet via [Stateless Address
Autoconfiguration](https://en.wikipedia.org/wiki/IPv6_address#Stateless_address_autoconfiguration):

```
mtak@ans1:/etc/network$ ip a l eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 2e:0a:c6:da:6a:3f brd ff:ff:ff:ff:ff:ff
    inet 10.100.0.73/24 brd 10.100.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 2001:1c00:2509:8200:2c0a:c6ff:feda:6a3f/64 scope global dynamic mngtmpaddr noprefixroute 
       valid_lft 86388sec preferred_lft 14388sec
    inet6 fe80::2c0a:c6ff:feda:6a3f/64 scope link 
       valid_lft forever preferred_lft forever
```

(A static IP for servers is obviously more preferable. Refer to your operating
systems manual on how to configure that)

### Multiple subnets

The above configuration can be extended for up to 256 subnets:

```
set interfaces ethernet eth0 dhcpv6-pd pd 1 interface eth1.2 host-address ::1
set interfaces ethernet eth0 dhcpv6-pd pd 1 interface eth1.2 prefix-id :2
set interfaces ethernet eth0 dhcpv6-pd pd 1 interface eth1.2 service dhcpv6-stateless

set interfaces ethernet eth0 dhcpv6-pd pd 1 interface eth1.5 host-address ::1
set interfaces ethernet eth0 dhcpv6-pd pd 1 interface eth1.5 prefix-id :5
set interfaces ethernet eth0 dhcpv6-pd pd 1 interface eth1.5 service dhcpv6-stateless
```

With the resulting interface configuration:

```
mtak@er1:~$ show interfaces ethernet
Codes: S - State, L - Link, u - Up, D - Down, A - Admin Down
Interface    IP Address                        S/L  Description                 
---------    ----------                        ---  -----------                 
eth0         83.85.23.20/23                    u/u  Ziggo                       
             2001:1c00:2500:0:adb2:751c:23b7:8f77/128
eth1         10.100.0.1/24                     u/u  Internal                    
             2001:1c00:2509:8200::1/64        
eth1.2       10.100.2.1/24                     u/u  DMZ                         
             2001:1c00:2509:8202::1/64        
eth1.5       10.100.5.1/24                     u/u  Guest                       
             2001:1c00:2509:8205::1/64        
eth1.100     10.100.100.1/24                   u/u  MAAS                        
eth1.316     10.100.3.17/30                    u/u  RIPE Atlas                  
```

