# Lab 11 - Network Configuration Documentation

## Network Topology Overview

| Бүс | Протокол | Төхөөрөмжүүд |
|------|----------|--------------|
| Ногоон зүүн | OSPF | SW1, SW2, VLAN10 PC, VLAN20 PC, Server |
| Шар зүүн | HSRP | R1, R2 (HSRP pair) |
| Ягаан төв | IPsec / GRE | R3 (center hub) |
| Шар баруун | HSRP | R4, R5 (HSRP pair) |
| Ногоон баруун | Random Routing (RIP) | SW3, SW4, PC3, PC4 |

## IP Addressing Scheme

| Segment | Network | Description |
|---------|---------|-------------|
| VLAN 10 | 192.168.10.0/24 | SW1 e0/2 -> VLAN10 PC |
| VLAN 20 | 192.168.20.0/24 | SW1 e0/1 -> VLAN20 PC |
| Server | 192.168.30.0/24 | SW2 e0/3 -> Server |
| SW2 - R1 | 10.0.1.0/30 | SW2 e0/1 - R1 f0/0 |
| SW2 - R2 | 10.0.2.0/30 | SW2 e0/2 - R2 f0/0 |
| R1 - R3 | 10.0.3.0/30 | R1 f0/1 - R3 f0/0 |
| R2 - R3 | 10.0.4.0/30 | R2 f0/1 - R3 f0/1 |
| R3 - R4 | 10.0.5.0/30 | R3 f1/0 - R4 f0/0 |
| R3 - R5 | 10.0.6.0/30 | R3 f2/0 - R5 f0/0 |
| R4 - SW3 | 10.0.7.0/30 | R4 f0/1 - SW3 e0/0 |
| R5 - SW3 | 10.0.8.0/30 | R5 f0/1 - SW3 e0/1 |
| SW3 - SW4 | 10.0.9.0/30 | SW3 e0/2 - SW4 e0/0 |
| PC3 LAN | 192.168.40.0/24 | SW4 e0/1 - PC3 |
| PC4 LAN | 192.168.50.0/24 | SW4 e0/2 - PC4 |
| HSRP Left VIP | 10.0.12.1/24 | R1=10.0.12.2, R2=10.0.12.3 |
| HSRP Right VIP | 10.0.78.1/24 | R4=10.0.78.2, R5=10.0.78.3 |
| GRE Tunnel (R1-R5) | 172.16.1.0/30 | R1=172.16.1.1, R5=172.16.1.2 |
| GRE Tunnel (R1-R4) | 172.16.2.0/30 | R1=172.16.2.1, R4=172.16.2.2 |
| GRE Tunnel (R2-R4) | 172.16.3.0/30 | R2=172.16.3.1, R4=172.16.3.2 |
| GRE Tunnel (R2-R5) | 172.16.4.0/30 | R2=172.16.4.1, R5=172.16.4.2 |

---

## SW1 Configuration

```
enable
configure terminal
hostname SW1

vlan 10
 name VLAN10
vlan 20
 name VLAN20
vlan 30
 name SERVER

interface ethernet 0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 no shutdown

interface ethernet 0/1
 switchport mode access
 switchport access vlan 20
 no shutdown

interface ethernet 0/2
 switchport mode access
 switchport access vlan 10
 no shutdown

interface vlan 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown

interface vlan 20
 ip address 192.168.20.1 255.255.255.0
 no shutdown

router ospf 1
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 0

end
write memory
```

---

## SW2 Configuration

```
enable
configure terminal
hostname SW2

vlan 10
 name VLAN10
vlan 20
 name VLAN20
vlan 30
 name SERVER

interface ethernet 0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 no shutdown

interface ethernet 0/1
 no switchport
 ip address 10.0.1.1 255.255.255.252
 no shutdown

interface ethernet 0/2
 no switchport
 ip address 10.0.2.1 255.255.255.252
 no shutdown

interface ethernet 0/3
 switchport mode access
 switchport access vlan 30
 no shutdown

interface vlan 30
 ip address 192.168.30.1 255.255.255.0
 no shutdown

ip routing

router ospf 1
 network 10.0.1.0 0.0.0.3 area 0
 network 10.0.2.0 0.0.0.3 area 0
 network 192.168.30.0 0.0.0.255 area 0
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 0

end
write memory
```

---

## SW3 Configuration

```
enable
configure terminal
hostname SW3

interface ethernet 0/0
 no switchport
 ip address 10.0.7.2 255.255.255.252
 no shutdown

interface ethernet 0/1
 no switchport
 ip address 10.0.8.2 255.255.255.252
 no shutdown

interface ethernet 0/2
 no switchport
 ip address 10.0.9.1 255.255.255.252
 no shutdown

ip routing

router rip
 version 2
 network 10.0.7.0
 network 10.0.8.0
 network 10.0.9.0
 no auto-summary

end
write memory
```

---

## SW4 Configuration

```
enable
configure terminal
hostname SW4

interface ethernet 0/0
 no switchport
 ip address 10.0.9.2 255.255.255.252
 no shutdown

interface ethernet 0/1
 no switchport
 ip address 192.168.40.1 255.255.255.0
 no shutdown

interface ethernet 0/2
 no switchport
 ip address 192.168.50.1 255.255.255.0
 no shutdown

ip routing

router rip
 version 2
 network 10.0.9.0
 network 192.168.40.0
 network 192.168.50.0
 no auto-summary

end
write memory
```

---

## R1 Configuration

**Үүрэг:** HSRP (Active), OSPF, IPsec/GRE VPN Tunnel R5 руу (Tunnel 0) + R4 руу (Tunnel 1)

```
enable
configure terminal
hostname R1

! Physical Interfaces
interface FastEthernet 0/0
 ip address 10.0.1.2 255.255.255.252
 no shutdown

interface FastEthernet 0/1
 ip address 10.0.3.1 255.255.255.252
 no shutdown

! HSRP тохиргоо (Active)
interface FastEthernet 0/0
 standby 1 ip 10.0.12.1
 standby 1 priority 110
 standby 1 preempt

! IPsec Phase 1
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key LAB11KEY address 10.0.6.2
crypto isakmp key LAB11KEY address 10.0.5.2

! IPsec Phase 2
crypto ipsec transform-set MYSET esp-aes 256 esp-sha256-hmac
 mode transport

crypto ipsec profile GRE_PROFILE
 set transform-set MYSET

! GRE VPN Tunnel 0 - R5 руу (R1 <-> R5)
interface Tunnel 0
 ip address 172.16.1.1 255.255.255.252
 tunnel source FastEthernet 0/1
 tunnel destination 10.0.6.2
 tunnel protection ipsec profile GRE_PROFILE
 no shutdown

! GRE VPN Tunnel 1 - R4 руу (R1 <-> R4)
interface Tunnel 1
 ip address 172.16.2.1 255.255.255.252
 tunnel source FastEthernet 0/1
 tunnel destination 10.0.5.2
 tunnel protection ipsec profile GRE_PROFILE
 no shutdown

! OSPF тохиргоо
router ospf 1
 network 10.0.1.0 0.0.0.3 area 0
 network 10.0.3.0 0.0.0.3 area 0
 network 172.16.1.0 0.0.0.3 area 0
 network 172.16.2.0 0.0.0.3 area 0

! Static routes for tunnel destinations (R3-р дамжина)
ip route 10.0.6.2 255.255.255.255 10.0.3.2
ip route 10.0.5.2 255.255.255.255 10.0.3.2

end
write memory
```

---

## R2 Configuration

**Үүрэг:** HSRP (Standby), OSPF, IPsec/GRE VPN Tunnel R4 руу (Tunnel 0) + R5 руу (Tunnel 1)

```
enable
configure terminal
hostname R2

! Physical Interfaces
interface FastEthernet 0/0
 ip address 10.0.2.2 255.255.255.252
 no shutdown

interface FastEthernet 0/1
 ip address 10.0.4.1 255.255.255.252
 no shutdown

! HSRP тохиргоо (Standby)
interface FastEthernet 0/0
 standby 1 ip 10.0.12.1
 standby 1 priority 90
 standby 1 preempt

! IPsec Phase 1
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key LAB11KEY address 10.0.5.2
crypto isakmp key LAB11KEY address 10.0.6.2

! IPsec Phase 2
crypto ipsec transform-set MYSET esp-aes 256 esp-sha256-hmac
 mode transport

crypto ipsec profile GRE_PROFILE
 set transform-set MYSET

! GRE VPN Tunnel 0 - R4 руу (R2 <-> R4)
interface Tunnel 0
 ip address 172.16.3.1 255.255.255.252
 tunnel source FastEthernet 0/1
 tunnel destination 10.0.5.2
 tunnel protection ipsec profile GRE_PROFILE
 no shutdown

! GRE VPN Tunnel 1 - R5 руу (R2 <-> R5)
interface Tunnel 1
 ip address 172.16.4.1 255.255.255.252
 tunnel source FastEthernet 0/1
 tunnel destination 10.0.6.2
 tunnel protection ipsec profile GRE_PROFILE
 no shutdown

! OSPF тохиргоо
router ospf 1
 network 10.0.2.0 0.0.0.3 area 0
 network 10.0.4.0 0.0.0.3 area 0
 network 172.16.3.0 0.0.0.3 area 0
 network 172.16.4.0 0.0.0.3 area 0

! Static routes for tunnel destinations (R3-р дамжина)
ip route 10.0.5.2 255.255.255.255 10.0.4.2
ip route 10.0.6.2 255.255.255.255 10.0.4.2

end
write memory
```

---

## R3 Configuration

**Үүрэг:** Төв router - tunnel traffic дамжуулах, OSPF ↔ RIP redistribute

```
enable
configure terminal
hostname R3

! Physical Interfaces
interface FastEthernet 0/0
 ip address 10.0.3.2 255.255.255.252
 description To R1
 no shutdown

interface FastEthernet 0/1
 ip address 10.0.4.2 255.255.255.252
 description To R2
 no shutdown

interface FastEthernet 1/0
 ip address 10.0.5.1 255.255.255.252
 description To R4
 no shutdown

interface FastEthernet 2/0
 ip address 10.0.6.1 255.255.255.252
 description To R5
 no shutdown

! OSPF (Зүүн тал) + Redistribute
router ospf 1
 network 10.0.3.0 0.0.0.3 area 0
 network 10.0.4.0 0.0.0.3 area 0
 redistribute rip subnets

! RIP (Баруун тал руу)
router rip
 version 2
 network 10.0.5.0
 network 10.0.6.0
 redistribute ospf 1 metric 5
 no auto-summary

end
write memory
```

---

## R4 Configuration

**Үүрэг:** HSRP Active (Баруун), IPsec/GRE VPN Tunnel R1 руу (Tunnel 0) + R2 руу (Tunnel 1), RIP

```
enable
configure terminal
hostname R4

! Physical Interfaces
interface FastEthernet 0/0
 ip address 10.0.5.2 255.255.255.252
 description To R3
 no shutdown

interface FastEthernet 0/1
 ip address 10.0.7.1 255.255.255.252
 description To SW3
 no shutdown

! HSRP тохиргоо (Active)
interface FastEthernet 0/1
 standby 2 ip 10.0.78.1
 standby 2 priority 110
 standby 2 preempt

! IPsec Phase 1
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key LAB11KEY address 10.0.3.1
crypto isakmp key LAB11KEY address 10.0.4.1

! IPsec Phase 2
crypto ipsec transform-set MYSET esp-aes 256 esp-sha256-hmac
 mode transport

crypto ipsec profile GRE_PROFILE
 set transform-set MYSET

! GRE VPN Tunnel 0 - R1 руу (R4 <-> R1)
interface Tunnel 0
 ip address 172.16.2.2 255.255.255.252
 tunnel source FastEthernet 0/0
 tunnel destination 10.0.3.1
 tunnel protection ipsec profile GRE_PROFILE
 no shutdown

! GRE VPN Tunnel 1 - R2 руу (R4 <-> R2)
interface Tunnel 1
 ip address 172.16.3.2 255.255.255.252
 tunnel source FastEthernet 0/0
 tunnel destination 10.0.4.1
 tunnel protection ipsec profile GRE_PROFILE
 no shutdown

! RIP routing
router rip
 version 2
 network 10.0.5.0
 network 10.0.7.0
 network 172.16.2.0
 network 172.16.3.0
 no auto-summary

! Static routes for tunnel destinations (R3-р дамжина)
ip route 10.0.3.1 255.255.255.255 10.0.5.1
ip route 10.0.4.1 255.255.255.255 10.0.5.1

end
write memory
```

---

## R5 Configuration

**Үүрэг:** HSRP Standby (Баруун), IPsec/GRE VPN Tunnel R1 руу (Tunnel 0) + R2 руу (Tunnel 1), RIP

```
enable
configure terminal
hostname R5

! Physical Interfaces
interface FastEthernet 0/0
 ip address 10.0.6.2 255.255.255.252
 description To R3
 no shutdown

interface FastEthernet 0/1
 ip address 10.0.8.1 255.255.255.252
 description To SW3
 no shutdown

! HSRP тохиргоо (Standby)
interface FastEthernet 0/1
 standby 2 ip 10.0.78.1
 standby 2 priority 90
 standby 2 preempt

! IPsec Phase 1
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key LAB11KEY address 10.0.3.1
crypto isakmp key LAB11KEY address 10.0.4.1

! IPsec Phase 2
crypto ipsec transform-set MYSET esp-aes 256 esp-sha256-hmac
 mode transport

crypto ipsec profile GRE_PROFILE
 set transform-set MYSET

! GRE VPN Tunnel 0 - R1 руу (R5 <-> R1)
interface Tunnel 0
 ip address 172.16.1.2 255.255.255.252
 tunnel source FastEthernet 0/0
 tunnel destination 10.0.3.1
 tunnel protection ipsec profile GRE_PROFILE
 no shutdown

! GRE VPN Tunnel 1 - R2 руу (R5 <-> R2)
interface Tunnel 1
 ip address 172.16.4.2 255.255.255.252
 tunnel source FastEthernet 0/0
 tunnel destination 10.0.4.1
 tunnel protection ipsec profile GRE_PROFILE
 no shutdown

! RIP routing
router rip
 version 2
 network 10.0.6.0
 network 10.0.8.0
 network 172.16.1.0
 network 172.16.4.0
 no auto-summary

! Static routes for tunnel destinations (R3-р дамжина)
ip route 10.0.3.1 255.255.255.255 10.0.6.1
ip route 10.0.4.1 255.255.255.255 10.0.6.1

end
write memory
```

---

## VPCS / PC Configurations

### VLAN10 PC
```
ip 192.168.10.10 255.255.255.0 192.168.10.1
```

### VLAN20 PC
```
ip 192.168.20.10 255.255.255.0 192.168.20.1
```

### Server (SW2 e0/3)
```
ip 192.168.30.10 255.255.255.0 192.168.30.1
```

### PC3 (SW4 e0/1)
```
ip 192.168.40.10 255.255.255.0 192.168.40.1
```

### PC4 (SW4 e0/2)
```
ip 192.168.50.10 255.255.255.0 192.168.50.1
```

---

## Full Mesh VPN Tunnel Redundancy

### 4 Tunnel-ийн бүрэн схем

| # | Tunnel | Endpoint A | Endpoint B | Tunnel IP A | Tunnel IP B |
|---|--------|-----------|-----------|-------------|-------------|
| 1 | R1↔R5 | R1 Tunnel0 (src:f0/1, dst:10.0.6.2) | R5 Tunnel0 (src:f0/0, dst:10.0.3.1) | 172.16.1.1/30 | 172.16.1.2/30 |
| 2 | R1↔R4 | R1 Tunnel1 (src:f0/1, dst:10.0.5.2) | R4 Tunnel0 (src:f0/0, dst:10.0.3.1) | 172.16.2.1/30 | 172.16.2.2/30 |
| 3 | R2↔R4 | R2 Tunnel0 (src:f0/1, dst:10.0.5.2) | R4 Tunnel1 (src:f0/0, dst:10.0.4.1) | 172.16.3.1/30 | 172.16.3.2/30 |
| 4 | R2↔R5 | R2 Tunnel1 (src:f0/1, dst:10.0.6.2) | R5 Tunnel1 (src:f0/0, dst:10.0.4.1) | 172.16.4.1/30 | 172.16.4.2/30 |

### Redundancy Scenarios (PC3 → Server)

| Scenario | Боломжтой зам |
|----------|--------------|
| Бүгд ажиллаж байгаа | R4→Tunnel→R1, R4→Tunnel→R2, R5→Tunnel→R1, R5→Tunnel→R2 |
| R1 унасан | R4→Tunnel1→R2, R5→Tunnel1→R2 |
| R2 унасан | R4→Tunnel0→R1, R5→Tunnel0→R1 |
| R4 унасан | HSRP→R5→Tunnel0→R1, R5→Tunnel1→R2 |
| R5 унасан | R4→Tunnel0→R1, R4→Tunnel1→R2 |
| R1+R5 унасан | R4→Tunnel1→R2 |
| R2+R4 унасан | R5→Tunnel0→R1 |

### HSRP Summary

| Бүс | Group | VIP | Active | Standby | Priority |
|-----|-------|-----|--------|---------|----------|
| Зүүн (R1, R2) | 1 | 10.0.12.1 | R1 (110) | R2 (90) | preempt |
| Баруун (R4, R5) | 2 | 10.0.78.1 | R4 (110) | R5 (90) | preempt |

---

## Verification Commands

```
! OSPF шалгах
show ip ospf neighbor
show ip route ospf

! HSRP шалгах
show standby brief

! GRE VPN Tunnel шалгах
show interface tunnel 0
show interface tunnel 1
show crypto ipsec sa
show crypto isakmp sa

! RIP шалгах
show ip route rip

! Connectivity (PC3 -> Server)
ping 192.168.30.10 source 192.168.40.10
traceroute 192.168.30.10 source 192.168.40.10

! All tunnels verify
show crypto session
show ip route | include Tunnel
```
