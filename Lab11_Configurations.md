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
| GRE Tunnel 1 | 172.16.1.0/30 | R1 - R3 |
| GRE Tunnel 2 | 172.16.2.0/30 | R2 - R3 |

---

## SW1 Configuration

**Үүрэг:** VLAN10, VLAN20 үүсгэх, access port, trunk port тохируулах

```
enable
configure terminal
hostname SW1

! VLANs үүсгэх
vlan 10
 name VLAN10
vlan 20
 name VLAN20
vlan 30
 name SERVER

! Trunk port - SW2 руу
interface ethernet 0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 no shutdown

! Access port - VLAN20 PC
interface ethernet 0/1
 switchport mode access
 switchport access vlan 20
 no shutdown

! Access port - VLAN10 PC
interface ethernet 0/2
 switchport mode access
 switchport access vlan 10
 no shutdown

! VLAN SVI interfaces (Layer 3)
interface vlan 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown

interface vlan 20
 ip address 192.168.20.1 255.255.255.0
 no shutdown

! OSPF тохиргоо
router ospf 1
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 0

end
write memory
```

---

## SW2 Configuration

**Үүрэг:** Trunk холболт SW1 руу, R1/R2 руу Layer 3 холболт, Server access port, OSPF

```
enable
configure terminal
hostname SW2

! VLANs
vlan 10
 name VLAN10
vlan 20
 name VLAN20
vlan 30
 name SERVER

! Trunk port - SW1 руу
interface ethernet 0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 no shutdown

! Layer 3 port - R1 руу
interface ethernet 0/1
 no switchport
 ip address 10.0.1.1 255.255.255.252
 no shutdown

! Layer 3 port - R2 руу
interface ethernet 0/2
 no switchport
 ip address 10.0.2.1 255.255.255.252
 no shutdown

! Access port - Server
interface ethernet 0/3
 switchport mode access
 switchport access vlan 30
 no shutdown

! VLAN SVI
interface vlan 30
 ip address 192.168.30.1 255.255.255.0
 no shutdown

! IP routing идэвхжүүлэх
ip routing

! OSPF тохиргоо
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

**Үүрэг:** R4, R5-аас ирсэн холболтыг хүлээн авч SW4 руу дамжуулах, RIP routing

```
enable
configure terminal
hostname SW3

! Layer 3 port - R4 руу
interface ethernet 0/0
 no switchport
 ip address 10.0.7.2 255.255.255.252
 no shutdown

! Layer 3 port - R5 руу
interface ethernet 0/1
 no switchport
 ip address 10.0.8.2 255.255.255.252
 no shutdown

! Layer 3 port - SW4 руу
interface ethernet 0/2
 no switchport
 ip address 10.0.9.1 255.255.255.252
 no shutdown

! IP routing идэвхжүүлэх
ip routing

! RIP routing (Random Routing Protocol)
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

**Үүрэг:** PC3, PC4-д access port өгөх, SW3-тай холбогдох

```
enable
configure terminal
hostname SW4

! Layer 3 port - SW3 руу
interface ethernet 0/0
 no switchport
 ip address 10.0.9.2 255.255.255.252
 no shutdown

! Access port - PC3
interface ethernet 0/1
 no switchport
 ip address 192.168.40.1 255.255.255.0
 no shutdown

! Access port - PC4
interface ethernet 0/2
 no switchport
 ip address 192.168.50.1 255.255.255.0
 no shutdown

! IP routing идэвхжүүлэх
ip routing

! RIP routing
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

**Үүрэг:** HSRP (Active), OSPF, GRE Tunnel + IPsec R3 руу

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

! HSRP тохиргоо (SW2 талын виртуал gateway)
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

crypto isakmp key LAB11KEY address 10.0.3.2

! IPsec Phase 2
crypto ipsec transform-set MYSET esp-aes 256 esp-sha256-hmac
 mode transport

crypto ipsec profile GRE_PROFILE
 set transform-set MYSET

! GRE Tunnel R3 руу
interface Tunnel 0
 ip address 172.16.1.1 255.255.255.252
 tunnel source FastEthernet 0/1
 tunnel destination 10.0.3.2
 tunnel protection ipsec profile GRE_PROFILE
 no shutdown

! OSPF тохиргоо
router ospf 1
 network 10.0.1.0 0.0.0.3 area 0
 network 10.0.3.0 0.0.0.3 area 0
 network 172.16.1.0 0.0.0.3 area 0

end
write memory
```

---

## R2 Configuration

**Үүрэг:** HSRP (Standby), OSPF, GRE Tunnel + IPsec R3 руу

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

crypto isakmp key LAB11KEY address 10.0.4.2

! IPsec Phase 2
crypto ipsec transform-set MYSET esp-aes 256 esp-sha256-hmac
 mode transport

crypto ipsec profile GRE_PROFILE
 set transform-set MYSET

! GRE Tunnel R3 руу
interface Tunnel 0
 ip address 172.16.2.1 255.255.255.252
 tunnel source FastEthernet 0/1
 tunnel destination 10.0.4.2
 tunnel protection ipsec profile GRE_PROFILE
 no shutdown

! OSPF тохиргоо
router ospf 1
 network 10.0.2.0 0.0.0.3 area 0
 network 10.0.4.0 0.0.0.3 area 0
 network 172.16.2.0 0.0.0.3 area 0

end
write memory
```

---

## R3 Configuration

**Үүрэг:** Төв router - IPsec/GRE tunnel-үүдийг хүлээн авч R1, R2, R4, R5-тай холбогдоно

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

! GRE Tunnel 0 - R1 руу
interface Tunnel 0
 ip address 172.16.1.2 255.255.255.252
 tunnel source FastEthernet 0/0
 tunnel destination 10.0.3.1
 tunnel protection ipsec profile GRE_PROFILE
 no shutdown

! GRE Tunnel 1 - R2 руу
interface Tunnel 1
 ip address 172.16.2.2 255.255.255.252
 tunnel source FastEthernet 0/1
 tunnel destination 10.0.4.1
 tunnel protection ipsec profile GRE_PROFILE
 no shutdown

! OSPF (Зүүн тал) + Static/Redistribute (Баруун тал)
router ospf 1
 network 10.0.3.0 0.0.0.3 area 0
 network 10.0.4.0 0.0.0.3 area 0
 network 172.16.1.0 0.0.0.3 area 0
 network 172.16.2.0 0.0.0.3 area 0
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

**Үүрэг:** HSRP Active (Баруун тал), R3-тай холбогдох, RIP

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

! HSRP тохиргоо (Active - баруун тал)
interface FastEthernet 0/1
 standby 2 ip 10.0.78.1
 standby 2 priority 110
 standby 2 preempt

! RIP routing
router rip
 version 2
 network 10.0.5.0
 network 10.0.7.0
 no auto-summary

end
write memory
```

---

## R5 Configuration

**Үүрэг:** HSRP Standby (Баруун тал), R3-тай холбогдох, RIP

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

! HSRP тохиргоо (Standby - баруун тал)
interface FastEthernet 0/1
 standby 2 ip 10.0.78.1
 standby 2 priority 90
 standby 2 preempt

! RIP routing
router rip
 version 2
 network 10.0.6.0
 network 10.0.8.0
 no auto-summary

end
write memory
```

---

## VPCS / PC Configurations

### VLAN10 PC (Зүүн доод)
```
ip 192.168.10.10 255.255.255.0 192.168.10.1
```

### VLAN20 PC (Зүүн дээд)
```
ip 192.168.20.10 255.255.255.0 192.168.20.1
```

### Server (SW2 e0/3)
```
ip 192.168.30.10 255.255.255.0 192.168.30.1
```

### PC3 (Баруун дээд - SW4 e0/1)
```
ip 192.168.40.10 255.255.255.0 192.168.40.1
```

### PC4 (Баруун доод - SW4 e0/2)
```
ip 192.168.50.10 255.255.255.0 192.168.50.1
```

---

## Хийгдэх ажлуудын товч тайлбар

| # | Бүс | Ажил | Тайлбар |
|---|------|------|---------|
| 1 | OSPF (Ногоон зүүн) | VLAN үүсгэх | SW1 дээр VLAN 10, 20 үүсгэж PC-үүдийг access port-д холбох |
| 2 | OSPF (Ногоон зүүн) | Trunk тохируулах | SW1-SW2 хооронд trunk link (dot1q) |
| 3 | OSPF (Ногоон зүүн) | OSPF routing | SW1, SW2, R1, R2 дээр OSPF area 0 |
| 4 | HSRP (Шар зүүн) | HSRP тохируулах | R1 = Active (priority 110), R2 = Standby (priority 90), VIP = 10.0.12.1 |
| 5 | IPsec/GRE (Ягаан) | GRE Tunnel | R1-R3, R2-R3 хооронд GRE tunnel үүсгэх |
| 6 | IPsec/GRE (Ягаан) | IPsec шифрлэлт | GRE tunnel дээр IPsec transport mode, AES-256, SHA-256, PSK |
| 7 | HSRP (Шар баруун) | HSRP тохируулах | R4 = Active (priority 110), R5 = Standby (priority 90), VIP = 10.0.78.1 |
| 8 | RIP (Ногоон баруун) | RIP v2 routing | SW3, SW4, R4, R5 дээр RIP version 2 |
| 9 | Redistribute | OSPF <-> RIP | R3 дээр OSPF болон RIP хоорондын redistribute хийх |
| 10 | End-to-End | Connectivity test | VLAN10 PC -> PC3, PC4 хүртэл ping шалгах |

---

## Verification Commands (Шалгах командууд)

```
! OSPF шалгах
show ip ospf neighbor
show ip route ospf

! HSRP шалгах
show standby brief

! GRE Tunnel шалгах
show interface tunnel 0
show crypto ipsec sa
show crypto isakmp sa

! RIP шалгах
show ip route rip
show ip rip database

! Connectivity
ping 192.168.40.10 source 192.168.10.10
traceroute 192.168.50.10
```
