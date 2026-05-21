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
| Төв | Ягаан | IPsec + GRE | R3 | VPN tunnel hub, OSPF↔RIP redistribute |
| Баруун WAN | Шар | HSRP | R4 (Active), R5 (Standby) | Gateway redundancy |
| Баруун LAN | Ногоон | RIP v2 | R3, R4, R5, SW3, SW4, PC3, PC4 | Алсын сүлжээ, PC3 байрлал |

---

## IP Addressing Scheme

| Segment | Network | Interface Details |
|---------|---------|-------------------|
| VLAN 10 | 192.168.10.0/24 | SW1 Vlan10 (.1) → VLAN10 PC (.10) |
| VLAN 20 | 192.168.20.0/24 | SW1 Vlan20 (.1) → VLAN20 PC (.10) |
| Server LAN (VLAN30) | 192.168.30.0/24 | SW2 Vlan30 (.1), SW1 Vlan30 (.2) → Server (.10) |
| SW2 ↔ R1 | 10.0.1.0/30 | SW2 e0/1 (.1) ↔ R1 f0/0 (.2) |
| SW2 ↔ R2 | 10.0.2.0/30 | SW2 e0/2 (.1) ↔ R2 f0/0 (.2) |
| R1 ↔ R3 | 10.0.3.0/30 | R1 f0/1 (.1) ↔ R3 f0/0 (.2) |
| R2 ↔ R3 | 10.0.4.0/30 | R2 f0/1 (.1) ↔ R3 f0/1 (.2) |
| R3 ↔ R4 | 10.0.5.0/30 | R3 f1/0 (.1) ↔ R4 f0/0 (.2) |
| R3 ↔ R5 | 10.0.6.0/30 | R3 f2/0 (.1) ↔ R5 f0/0 (.2) |
| R4 ↔ SW3 | 10.0.7.0/30 | R4 f0/1 (.1) ↔ SW3 e0/0 (.2) |
| R5 ↔ SW3 | 10.0.8.0/30 | R5 f0/1 (.1) ↔ SW3 e0/1 (.2) |
| SW3 ↔ SW4 | 10.0.9.0/30 | SW3 e0/2 (.1) ↔ SW4 e0/0 (.2) |
| PC3 LAN | 192.168.40.0/24 | SW4 e0/1 (.1) → PC3 (.10) |
| PC4 LAN | 192.168.50.0/24 | SW4 e0/2 (.1) → PC4 (.10) |
| HSRP Left VIP | 10.0.12.1 | R1 (priority 110, Active), R2 (priority 90, Standby) |
| HSRP Right VIP | 10.0.78.1 | R4 (priority 110, Active), R5 (priority 90, Standby) |
| GRE Tunnel 0 | 172.16.1.0/30 | R1 Tunnel0 (.1) ↔ R3 Tunnel0 (.2) |
| GRE Tunnel 1 | 172.16.2.0/30 | R2 Tunnel0 (.1) ↔ R3 Tunnel1 (.2) |

---

## SW1 Configuration

**Үүрэг:** VLAN 10, 20 access port, trunk (SW2 руу), VLAN30 SVI (OSPF neighbor SW2-тай), inter-VLAN routing

```
hostname SW1
!
interface Ethernet0/0
 switchport trunk encapsulation dot1q
 switchport trunk allowed vlan 10,20,30
 switchport mode trunk
!
interface Ethernet0/1
 switchport access vlan 20
 switchport mode access
!
interface Ethernet0/2
 switchport access vlan 10
 switchport mode access
!
interface Vlan10
 ip address 192.168.10.1 255.255.255.0
!
interface Vlan20
 ip address 192.168.20.1 255.255.255.0
!
interface Vlan30
 ip address 192.168.30.2 255.255.255.0
!
router ospf 1
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 0
 network 192.168.30.0 0.0.0.255 area 0
```

---

## SW2 Configuration

**Үүрэг:** Trunk (SW1), Layer 3 uplink (R1, R2), Server access port (VLAN30), OSPF

```
hostname SW2
!
interface Ethernet0/0
 switchport trunk encapsulation dot1q
 switchport trunk allowed vlan 10,20,30
 switchport mode trunk
!
interface Ethernet0/1
 no switchport
 ip address 10.0.1.1 255.255.255.252
!
interface Ethernet0/2
 no switchport
 ip address 10.0.2.1 255.255.255.252
!
interface Ethernet0/3
 switchport access vlan 30
 switchport mode access
!
interface Vlan30
 ip address 192.168.30.1 255.255.255.0
!
router ospf 1
 network 10.0.1.0 0.0.0.3 area 0
 network 10.0.2.0 0.0.0.3 area 0
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 0
 network 192.168.30.0 0.0.0.255 area 0
```

---

## R1 Configuration

**Үүрэг:** HSRP Active (зүүн), OSPF, GRE Tunnel + IPsec → R3, VPN endpoint

```
hostname R1
!
crypto isakmp policy 10
 encr aes 256
 authentication pre-share
crypto isakmp key LAB11KEY address 10.0.3.2
!
crypto ipsec transform-set MYSET esp-aes 256 esp-sha-hmac
 mode transport
!
crypto ipsec profile GRE_PROFILE
 set transform-set MYSET
!
interface Tunnel0
 ip address 172.16.1.1 255.255.255.252
 tunnel source FastEthernet0/1
 tunnel destination 10.0.3.2
 tunnel protection ipsec profile GRE_PROFILE
!
interface FastEthernet0/0
 ip address 10.0.1.2 255.255.255.252
 standby 1 ip 10.0.12.1
 standby 1 priority 110
 standby 1 preempt
!
interface FastEthernet0/1
 ip address 10.0.3.1 255.255.255.252
!
router ospf 1
 network 10.0.1.0 0.0.0.3 area 0
 network 10.0.3.0 0.0.0.3 area 0
 network 172.16.1.0 0.0.0.3 area 0
```

---

## R2 Configuration

**Үүрэг:** HSRP Standby (зүүн), OSPF, GRE Tunnel + IPsec → R3, Нөөц VPN endpoint

```
hostname R2
!
crypto isakmp policy 10
 encr aes 256
 authentication pre-share
crypto isakmp key LAB11KEY address 10.0.4.2
!
crypto ipsec transform-set MYSET esp-aes 256 esp-sha-hmac
 mode transport
!
crypto ipsec profile GRE_PROFILE
 set transform-set MYSET
!
interface Tunnel0
 ip address 172.16.2.1 255.255.255.252
 tunnel source FastEthernet0/1
 tunnel destination 10.0.4.2
 tunnel protection ipsec profile GRE_PROFILE
!
interface FastEthernet0/0
 ip address 10.0.2.2 255.255.255.252
 standby 1 ip 10.0.12.1
 standby 1 priority 90
 standby 1 preempt
!
interface FastEthernet0/1
 ip address 10.0.4.1 255.255.255.252
!
router ospf 1
 network 10.0.2.0 0.0.0.3 area 0
 network 10.0.4.0 0.0.0.3 area 0
 network 172.16.2.0 0.0.0.3 area 0
```

---

## R3 Configuration

**Үүрэг:** VPN Hub - IPsec/GRE tunnel (R1, R2), OSPF ↔ RIP redistribute, Зүүн (OSPF) ↔ Баруун (RIP) холбох

```
hostname R3
!
crypto isakmp policy 10
 encr aes 256
 authentication pre-share
crypto isakmp key LAB11KEY address 10.0.3.1
crypto isakmp key LAB11KEY address 10.0.4.1
!
crypto ipsec transform-set MYSET esp-aes 256 esp-sha-hmac
 mode transport
!
crypto ipsec profile GRE_PROFILE
 set transform-set MYSET
!
interface Tunnel0
 ip address 172.16.1.2 255.255.255.252
 tunnel source FastEthernet0/0
 tunnel destination 10.0.3.1
 tunnel protection ipsec profile GRE_PROFILE
!
interface Tunnel1
 ip address 172.16.2.2 255.255.255.252
 tunnel source FastEthernet0/1
 tunnel destination 10.0.4.1
 tunnel protection ipsec profile GRE_PROFILE
!
interface FastEthernet0/0
 description To R1
 ip address 10.0.3.2 255.255.255.252
!
interface FastEthernet0/1
 description To R2
 ip address 10.0.4.2 255.255.255.252
!
interface FastEthernet1/0
 description To R4
 ip address 10.0.5.1 255.255.255.252
!
interface FastEthernet2/0
 description To R5
 ip address 10.0.6.1 255.255.255.252
!
router ospf 1
 redistribute rip subnets
 network 10.0.3.0 0.0.0.3 area 0
 network 10.0.4.0 0.0.0.3 area 0
 network 172.16.1.0 0.0.0.3 area 0
 network 172.16.2.0 0.0.0.3 area 0
!
router rip
 version 2
 redistribute ospf 1 metric 5
 network 10.0.0.0
 no auto-summary
```

---

## R4 Configuration

**Үүрэг:** HSRP Active (баруун), RIP v2

```
hostname R4
!
interface FastEthernet0/0
 description To R3
 ip address 10.0.5.2 255.255.255.252
!
interface FastEthernet0/1
 description To SW3
 ip address 10.0.7.1 255.255.255.252
 standby 2 ip 10.0.78.1
 standby 2 priority 110
 standby 2 preempt
!
router rip
 version 2
 network 10.0.0.0
 no auto-summary
```

---

## R5 Configuration

**Үүрэг:** HSRP Standby (баруун), RIP v2

```
hostname R5
!
interface FastEthernet0/0
 description To R3
 ip address 10.0.6.2 255.255.255.252
!
interface FastEthernet0/1
 description To SW3
 ip address 10.0.8.1 255.255.255.252
 standby 2 ip 10.0.78.1
 standby 2 priority 90
 standby 2 preempt
!
router rip
 version 2
 network 10.0.0.0
 no auto-summary
```

---

## SW3 Configuration

**Үүрэг:** R4, R5-аас ирсэн холболтыг SW4 руу дамжуулах, RIP v2

```
hostname SW3
!
interface Ethernet0/0
 no switchport
 ip address 10.0.7.2 255.255.255.252
!
interface Ethernet0/1
 no switchport
 ip address 10.0.8.2 255.255.255.252
!
interface Ethernet0/2
 no switchport
 ip address 10.0.9.1 255.255.255.252
!
router rip
 version 2
 network 10.0.0.0
 no auto-summary
```

---

## SW4 Configuration

**Үүрэг:** PC3, PC4 gateway, RIP v2

```
hostname SW4
!
interface Ethernet0/0
 no switchport
 ip address 10.0.9.2 255.255.255.252
!
interface Ethernet0/1
 no switchport
 ip address 192.168.40.1 255.255.255.0
!
interface Ethernet0/2
 no switchport
 ip address 192.168.50.1 255.255.255.0
!
router rip
 version 2
 network 10.0.0.0
 network 192.168.40.0
 network 192.168.50.0
 no auto-summary
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

### Server (SW2 e0/3 - VLAN30, VPN-ийн зорилтот хост)
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
| 1 | OSPF (Зүүн) | VLAN үүсгэх | SW1: VLAN 10, 20 access port, trunk SW2 руу |
| 2 | OSPF (Зүүн) | OSPF routing | SW1, SW2, R1, R2, R3 дээр OSPF area 0 |
| 3 | HSRP (Зүүн) | HSRP тохируулах | R1 Active (priority 110), R2 Standby (priority 90), VIP=10.0.12.1 |
| 4 | VPN (Төв) | GRE Tunnel үүсгэх | R1↔R3, R2↔R3 хооронд GRE tunnel |
| 5 | VPN (Төв) | IPsec шифрлэлт | GRE дээр IPsec (AES-256, ESP-SHA-HMAC, PSK="LAB11KEY", transport mode) |
| 6 | Redistribute | OSPF ↔ RIP | R3 дээр `redistribute rip subnets` + `redistribute ospf 1 metric 5` |
| 7 | RIP (Баруун) | RIP v2 routing | R3, R4, R5, SW3, SW4 дээр RIP version 2 |
| 8 | HSRP (Баруун) | HSRP тохируулах | R4 Active (priority 110), R5 Standby (priority 90), VIP=10.0.78.1 |
| 9 | End-to-End | VPN test | PC3 (192.168.40.10) → Server (192.168.30.10) ping |

---

## VPN Data Flow (PC3 → Server)

```
PC3 (192.168.40.10)
  ↓ (Layer 2)
SW4 e0/1
  ↓ (RIP v2)
SW3 e0/2 → e0/0
  ↓ (RIP v2)
R4 f0/1 [HSRP Active] → f0/0
  ↓ (RIP v2)
R3 f1/0 → Tunnel0 [GRE + IPsec шифрлэлт]
  ↓ (Encrypted VPN Tunnel)
R1 Tunnel0 → f0/0 [HSRP Active]
  ↓ (OSPF)
SW2 e0/1 → Vlan30
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

! ===== RIP шалгах =====
show ip route rip
show ip protocols

! ===== End-to-End VPN Connectivity =====
! PC3-аас Server руу
ping 192.168.30.10

! Server-ээс PC3 руу
ping 192.168.40.10
```

---

## Тэмдэглэл

- **IPsec Pre-Shared Key:** `LAB11KEY` (R1, R2, R3 дээр ижил)
- **IPsec Transform:** esp-aes 256 + esp-sha-hmac, transport mode
- **GRE Tunnel:** R1↔R3 (Tunnel0), R2↔R3 (Tunnel0/Tunnel1)
- **HSRP зүүн:** R1 Active (110), R2 Standby (90), VIP=10.0.12.1
- **HSRP баруун:** R4 Active (110), R5 Standby (90), VIP=10.0.78.1
- **Redistribute:** R3 дээр OSPF↔RIP хоёр талдаа redistribute
- **Redundancy:** VPN path R1 эсвэл R2-оор дамжих боломжтой (2 tunnel)
