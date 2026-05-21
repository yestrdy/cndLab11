# Lab 11 - VPN Network Configuration (GRE over IPsec)

## Лабораторийн зорилго

PC3 (192.168.40.10) нь Server (192.168.30.10) руу **GRE over IPsec VPN tunnel**-ээр дамжин хандана. Сүлжээний аюулгүй байдлыг IPsec-ээр хангаж, GRE tunnel нь routing protocol-ийг дамжуулна.

**Траффикийн зам:** PC3 → SW4 → SW3 → R4/R5 (HSRP) → R3 → **[GRE+IPsec Tunnel]** → R1/R2 (HSRP) → SW2 → Server

---

## Network Topology Overview

| Бүс | Өнгө | Протокол | Төхөөрөмжүүд | Үүрэг |
|------|-------|----------|--------------|-------|
| Зүүн LAN | Ногоон | OSPF | SW1, SW2, VLAN10 PC, VLAN20 PC, Server | Дотоод сүлжээ, Server байрлал |
| Зүүн WAN | Шар | HSRP | R1 (Active), R2 (Standby) | Gateway redundancy, VPN endpoint |
| Төв | Ягаан | IPsec + GRE | R3 | VPN tunnel hub, протокол redistribute |
| Баруун WAN | Шар | HSRP | R4 (Active), R5 (Standby) | Gateway redundancy |
| Баруун LAN | Ногоон | BGP (EGP) | SW3, SW4, PC3, PC4 | Алсын сүлжээ, PC3 байрлал |

---

## IP Addressing Scheme

| Segment | Network | Interface Details |
|---------|---------|-------------------|
| VLAN 10 | 192.168.10.0/24 | SW1 e0/2 → VLAN10 PC |
| VLAN 20 | 192.168.20.0/24 | SW1 e0/1 → VLAN20 PC |
| Server LAN | 192.168.30.0/24 | SW2 e0/3 → Server |
| SW2 ↔ R1 | 10.0.1.0/30 | SW2 e0/1 (.1) ↔ R1 f0/0 (.2) |
| SW2 ↔ R2 | 10.0.2.0/30 | SW2 e0/2 (.1) ↔ R2 f0/0 (.2) |
| R1 ↔ R3 | 10.0.3.0/30 | R1 f0/1 (.1) ↔ R3 f0/0 (.2) |
| R2 ↔ R3 | 10.0.4.0/30 | R2 f0/1 (.1) ↔ R3 f0/1 (.2) |
| R3 ↔ R4 | 10.0.5.0/30 | R3 f1/0 (.1) ↔ R4 f0/0 (.2) |
| R3 ↔ R5 | 10.0.6.0/30 | R3 f2/0 (.1) ↔ R5 f0/0 (.2) |
| R4 ↔ SW3 | 10.0.7.0/30 | R4 f0/1 (.1) ↔ SW3 e0/0 (.2) |
| R5 ↔ SW3 | 10.0.8.0/30 | R5 f0/1 (.1) ↔ SW3 e0/1 (.2) |
| SW3 ↔ SW4 | 10.0.9.0/30 | SW3 e0/2 (.1) ↔ SW4 e0/0 (.2) |
| PC3 LAN | 192.168.40.0/24 | SW4 e0/1 → PC3 |
| PC4 LAN | 192.168.50.0/24 | SW4 e0/2 → PC4 |
| HSRP Left VIP | 10.0.12.1 | R1(.2) Active, R2(.3) Standby |
| HSRP Right VIP | 10.0.78.1 | R4(.2) Active, R5(.3) Standby |
| GRE Tunnel 0 | 172.16.1.0/30 | R1 Tunnel0 (.1) ↔ R3 Tunnel0 (.2) |
| GRE Tunnel 1 | 172.16.2.0/30 | R2 Tunnel0 (.1) ↔ R3 Tunnel1 (.2) |

---

## SW1 Configuration

**Үүрэг:** VLAN 10, 20 үүсгэх, access/trunk port тохируулах, inter-VLAN routing (OSPF)

```
enable
configure terminal
hostname SW1

! VLANs үүсгэх
vlan 10
 name VLAN10
vlan 20
 name VLAN20

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

! VLAN SVI interfaces
interface vlan 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown

interface vlan 20
 ip address 192.168.20.1 255.255.255.0
 no shutdown

! IP routing идэвхжүүлэх
ip routing

! OSPF тохиргоо
router ospf 1
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 0

end
write memory
```

---

## SW2 Configuration

**Үүрэг:** Trunk (SW1), Layer 3 uplink (R1, R2), Server access port, OSPF – Server-ийн gateway

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

## R1 Configuration

**Үүрэг:** HSRP Active (зүүн тал), OSPF, GRE Tunnel + IPsec → R3, VPN endpoint

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

! HSRP тохиргоо - Active router
interface FastEthernet 0/0
 standby 1 ip 10.0.12.1
 standby 1 priority 110
 standby 1 preempt

! ============ IPsec тохиргоо ============
! Phase 1 - ISAKMP Policy
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key LAB11KEY address 10.0.3.2

! Phase 2 - Transform Set
crypto ipsec transform-set MYSET esp-aes 256 esp-sha256-hmac
 mode transport

! IPsec Profile (GRE дээр хэрэглэнэ)
crypto ipsec profile GRE_PROFILE
 set transform-set MYSET

! ============ GRE Tunnel ============
! VPN Tunnel - R3 руу (Server LAN-д хандах зам)
interface Tunnel 0
 ip address 172.16.1.1 255.255.255.252
 tunnel source FastEthernet 0/1
 tunnel destination 10.0.3.2
 tunnel protection ipsec profile GRE_PROFILE
 no shutdown

! OSPF тохиргоо - Tunnel network-ийг зарлана
router ospf 1
 network 10.0.1.0 0.0.0.3 area 0
 network 10.0.3.0 0.0.0.3 area 0
 network 172.16.1.0 0.0.0.3 area 0

end
write memory
```

---

## R2 Configuration

**Үүрэг:** HSRP Standby (зүүн тал), OSPF, GRE Tunnel + IPsec → R3, Нөөц VPN endpoint

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

! HSRP тохиргоо - Standby router
interface FastEthernet 0/0
 standby 1 ip 10.0.12.1
 standby 1 priority 90
 standby 1 preempt

! ============ IPsec тохиргоо ============
! Phase 1 - ISAKMP Policy
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key LAB11KEY address 10.0.4.2

! Phase 2 - Transform Set
crypto ipsec transform-set MYSET esp-aes 256 esp-sha256-hmac
 mode transport

! IPsec Profile
crypto ipsec profile GRE_PROFILE
 set transform-set MYSET

! ============ GRE Tunnel ============
! Нөөц VPN Tunnel - R3 руу
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

**Үүрэг:** VPN Hub - IPsec/GRE tunnel-үүдийг R1, R2-оос хүлээн авч, OSPF ↔ BGP redistribute хийнэ. Зүүн (OSPF) болон Баруун (BGP) сүлжээг холбоно.

```
enable
configure terminal
hostname R3

! ============ Physical Interfaces ============
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

! ============ IPsec тохиргоо ============
! Phase 1
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key LAB11KEY address 10.0.3.1
crypto isakmp key LAB11KEY address 10.0.4.1

! Phase 2
crypto ipsec transform-set MYSET esp-aes 256 esp-sha256-hmac
 mode transport

crypto ipsec profile GRE_PROFILE
 set transform-set MYSET

! ============ GRE Tunnels ============
! Tunnel 0 - R1 руу (Primary VPN path)
interface Tunnel 0
 ip address 172.16.1.2 255.255.255.252
 tunnel source FastEthernet 0/0
 tunnel destination 10.0.3.1
 tunnel protection ipsec profile GRE_PROFILE
 no shutdown

! Tunnel 1 - R2 руу (Backup VPN path)
interface Tunnel 1
 ip address 172.16.2.2 255.255.255.252
 tunnel source FastEthernet 0/1
 tunnel destination 10.0.4.1
 tunnel protection ipsec profile GRE_PROFILE
 no shutdown

! ============ OSPF (Зүүн тал - Server рүү) ============
router ospf 1
 router-id 3.3.3.3
 network 10.0.3.0 0.0.0.3 area 0
 network 10.0.4.0 0.0.0.3 area 0
 network 172.16.1.0 0.0.0.3 area 0
 network 172.16.2.0 0.0.0.3 area 0
 redistribute bgp 65001 subnets

! ============ BGP (Баруун тал - PC3 руу, EGP) ============
router bgp 65001
 bgp log-neighbor-changes
 neighbor 10.0.5.2 remote-as 65002
 neighbor 10.0.6.2 remote-as 65002
 !
 address-family ipv4
  neighbor 10.0.5.2 activate
  neighbor 10.0.6.2 activate
  network 10.0.3.0 mask 255.255.255.252
  network 10.0.4.0 mask 255.255.255.252
  network 172.16.1.0 mask 255.255.255.252
  network 172.16.2.0 mask 255.255.255.252
  redistribute ospf 1
 exit-address-family

end
write memory
```

---

## R4 Configuration

**Үүрэг:** HSRP Active (баруун тал), BGP (AS 65002), R3-тай eBGP peering

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

! HSRP тохиргоо - Active (баруун тал)
interface FastEthernet 0/1
 standby 2 ip 10.0.78.1
 standby 2 priority 110
 standby 2 preempt

! ============ BGP тохиргоо (EGP - AS 65002) ============
router bgp 65002
 bgp log-neighbor-changes
 neighbor 10.0.5.1 remote-as 65001
 neighbor 10.0.7.2 remote-as 65002
 !
 address-family ipv4
  neighbor 10.0.5.1 activate
  neighbor 10.0.7.2 activate
  network 10.0.5.0 mask 255.255.255.252
  network 10.0.7.0 mask 255.255.255.252
 exit-address-family

end
write memory
```

---

## R5 Configuration

**Үүрэг:** HSRP Standby (баруун тал), BGP (AS 65002), R3-тай eBGP peering

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

! HSRP тохиргоо - Standby (баруун тал)
interface FastEthernet 0/1
 standby 2 ip 10.0.78.1
 standby 2 priority 90
 standby 2 preempt

! ============ BGP тохиргоо (EGP - AS 65002) ============
router bgp 65002
 bgp log-neighbor-changes
 neighbor 10.0.6.1 remote-as 65001
 neighbor 10.0.8.2 remote-as 65002
 !
 address-family ipv4
  neighbor 10.0.6.1 activate
  neighbor 10.0.8.2 activate
  network 10.0.6.0 mask 255.255.255.252
  network 10.0.8.0 mask 255.255.255.252
 exit-address-family

end
write memory
```

---

## SW3 Configuration

**Үүрэг:** R4, R5-аас ирсэн холболтыг хүлээн авч SW4 руу дамжуулах, iBGP (AS 65002)

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

! ============ BGP тохиргоо (AS 65002 - iBGP) ============
router bgp 65002
 bgp log-neighbor-changes
 neighbor 10.0.7.1 remote-as 65002
 neighbor 10.0.8.1 remote-as 65002
 neighbor 10.0.9.2 remote-as 65002
 !
 address-family ipv4
  neighbor 10.0.7.1 activate
  neighbor 10.0.8.1 activate
  neighbor 10.0.9.2 activate
  network 10.0.7.0 mask 255.255.255.252
  network 10.0.8.0 mask 255.255.255.252
  network 10.0.9.0 mask 255.255.255.252
 exit-address-family

end
write memory
```

---

## SW4 Configuration

**Үүрэг:** PC3, PC4-д gateway болох, SW3-тай iBGP (AS 65002)

```
enable
configure terminal
hostname SW4

! Layer 3 port - SW3 руу
interface ethernet 0/0
 no switchport
 ip address 10.0.9.2 255.255.255.252
 no shutdown

! PC3 LAN gateway
interface ethernet 0/1
 no switchport
 ip address 192.168.40.1 255.255.255.0
 no shutdown

! PC4 LAN gateway
interface ethernet 0/2
 no switchport
 ip address 192.168.50.1 255.255.255.0
 no shutdown

! IP routing идэвхжүүлэх
ip routing

! ============ BGP тохиргоо (AS 65002 - iBGP) ============
router bgp 65002
 bgp log-neighbor-changes
 neighbor 10.0.9.1 remote-as 65002
 !
 address-family ipv4
  neighbor 10.0.9.1 activate
  network 192.168.40.0 mask 255.255.255.0
  network 192.168.50.0 mask 255.255.255.0
  network 10.0.9.0 mask 255.255.255.252
 exit-address-family

end
write memory
```

---

## VPCS / PC Configurations

### VLAN10 PC (Зүүн доод)
```
ip 192.168.10.10/24 192.168.10.1
```

### VLAN20 PC (Зүүн дээд)
```
ip 192.168.20.10/24 192.168.20.1
```

### Server (SW2 e0/3 - VPN-ийн зорилтот хост)
```
ip 192.168.30.10/24 192.168.30.1
```

### PC3 (Баруун дээд - VPN-ээр Server рүү хандана)
```
ip 192.168.40.10/24 192.168.40.1
```

### PC4 (Баруун доод)
```
ip 192.168.50.10/24 192.168.50.1
```

---

## Хийгдэх ажлуудын дараалал

| # | Бүс | Ажил | Тайлбар |
|---|------|------|---------|
| 1 | OSPF (Зүүн) | VLAN үүсгэх | SW1: VLAN 10, 20 үүсгэж PC-үүдийг access port-д холбох |
| 2 | OSPF (Зүүн) | Trunk тохируулах | SW1 ↔ SW2 trunk link (dot1q) |
| 3 | OSPF (Зүүн) | OSPF routing | SW1, SW2, R1, R2 дээр OSPF area 0 тохируулах |
| 4 | HSRP (Зүүн) | HSRP тохируулах | R1 = Active (priority 110), R2 = Standby (priority 90), VIP = 10.0.12.1 |
| 5 | VPN (Төв) | GRE Tunnel үүсгэх | R1↔R3, R2↔R3 хооронд GRE tunnel |
| 6 | VPN (Төв) | IPsec шифрлэлт | GRE дээр IPsec (AES-256, SHA-256, PSK="LAB11KEY", transport mode) |
| 7 | HSRP (Баруун) | HSRP тохируулах | R4 = Active (priority 110), R5 = Standby (priority 90), VIP = 10.0.78.1 |
| 8 | EGP (Баруун) | BGP тохируулах | AS 65001 (R3), AS 65002 (R4, R5, SW3, SW4) - eBGP + iBGP |
| 9 | Redistribute | OSPF ↔ BGP | R3 дээр OSPF ↔ BGP redistribute (Зүүн ↔ Баруун холбох) |
| 10 | End-to-End | VPN test | PC3 (192.168.40.10) → Server (192.168.30.10) ping + traceroute |

---

## VPN Data Flow (PC3 → Server)

```
PC3 (192.168.40.10)
  ↓ (Layer 2)
SW4 e0/1
  ↓ (BGP - AS 65002)
SW3 e0/2 → e0/0
  ↓ (BGP iBGP)
R4 f0/1 [HSRP Active] → f0/0
  ↓ (eBGP AS65002 → AS65001)
R3 f1/0 → Tunnel0 [GRE + IPsec шифрлэлт]
  ↓ (Encrypted VPN Tunnel)
R1 Tunnel0 → f0/0 [HSRP Active]
  ↓ (OSPF)
SW2 e0/1 → e0/3
  ↓ (Layer 2 - VLAN 30)
Server (192.168.30.10)
```

---

## Verification Commands (Шалгах командууд)

```
! ===== OSPF шалгах =====
show ip ospf neighbor
show ip route ospf

! ===== HSRP шалгах =====
show standby brief

! ===== GRE + IPsec VPN шалгах =====
show interface tunnel 0
show interface tunnel 1
show crypto ipsec sa
show crypto isakmp sa
show crypto session

! ===== BGP шалгах =====
show ip bgp summary
show ip bgp
show ip route bgp

! ===== End-to-End VPN Connectivity =====
! PC3-аас Server руу ping
ping 192.168.30.10 source 192.168.40.10

! Traceroute - VPN tunnel-ээр дамжиж байгааг шалгах
traceroute 192.168.30.10 source 192.168.40.10

! Server-ээс PC3 руу буцах
ping 192.168.40.10 source 192.168.30.10
```

---

## Тэмдэглэл

- **IPsec Pre-Shared Key:** `LAB11KEY` (бүх VPN endpoint дээр ижил)
- **GRE tunnel mode:** transport (GRE header-ийг IPsec шифрлэнэ)
- **BGP AS Numbering:** AS 65001 = R3 (WAN/VPN hub), AS 65002 = R4, R5, SW3, SW4 (баруун LAN)
- **HSRP:** R1 унтарвал R2 автоматаар Active болно (зүүн), R4 унтарвал R5 Active болно (баруун)
- **Redundancy:** VPN path нь R1 эсвэл R2-оор дамжих боломжтой (2 tunnel)
