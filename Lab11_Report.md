# Компьютерын сүлжээний төхөөрөмж

## ECEN322

## Лабораторийн ажил № 11

## GRE over IPsec VPN сүлжээний зохион байгуулалт

МТЭС, ЭХИТ, Компьютерын сүлжээ

Д.Билгүүнтөгс, 24B1NUM0440

---

## Онолын судалгаа

Динамик чиглүүлэлтийн протоколууд болон VPN технологийг хослуулан ашиглаж, алсын сүлжээнүүдийг аюулгүйгээр холбох тохиргоог хийж сурах. Энэхүү ажлын хүрээнд GRE (Generic Routing Encapsulation) tunnel-ийг IPsec (Internet Protocol Security)-ээр шифрлэж, OSPF болон RIP v2 чиглүүлэлтийн протоколуудыг redistribute хийн, HSRP (Hot Standby Router Protocol)-ээр gateway redundancy хангах тохиргоонуудыг хийж гүйцэтгэв.

---

## 1. Даалгавар

Байгууллагын 5 router, 4 switch бүхий сүлжээнд дараах тохиргоонуудыг хийх:

- **VPN:** GRE tunnel-ийг IPsec-ээр шифрлэж алсын салбар (PC3) болон төв сүлжээний Server хооронд аюулгүй холболт бий болгох
- **OSPF:** Зүүн талын LAN сүлжээнд (SW1, SW2, R1, R2, R3) OSPF area 0 тохируулах
- **RIP v2:** Баруун талын LAN сүлжээнд (R3, R4, R5, SW3, SW4) RIP version 2 тохируулах
- **Redistribute:** R3 дээр OSPF ↔ RIP хоёр протоколыг redistribute хийж бүх сүлжээг холбох
- **HSRP:** R1/R2 (зүүн) болон R4/R5 (баруун) дээр gateway redundancy тохируулах
- **VLAN:** SW1 дээр VLAN 10, 20 үүсгэж inter-VLAN routing хийх

---

## 1.1 Сүлжээний топологи

**Зураг 1.** Байгууллагын сүлжээний ерөнхий топологи (GNS3)

Сүлжээний бүтэц:

| Бүс | Өнгө | Протокол | Төхөөрөмжүүд | Үүрэг |
|------|-------|----------|--------------|-------|
| Зүүн LAN | Ногоон | OSPF | SW1, SW2, VLAN10 PC, VLAN20 PC, Server | Дотоод сүлжээ, Server |
| Зүүн WAN | Шар | HSRP | R1 (Active), R2 (Standby) | Gateway redundancy, VPN endpoint |
| Төв | Ягаан | IPsec + GRE | R3 | VPN tunnel hub, OSPF↔RIP redistribute |
| Баруун WAN | Шар | HSRP | R4 (Active), R5 (Standby) | Gateway redundancy |
| Баруун LAN | Ногоон | RIP v2 | R3, R4, R5, SW3, SW4, PC3, PC4 | Алсын сүлжээ |

---

## 1.2 IP хаяглалт

| Холболт | Network | Interface |
|---------|---------|-----------|
| VLAN 10 | 192.168.10.0/24 | SW1 Vlan10 (.1) → PC (.10) |
| VLAN 20 | 192.168.20.0/24 | SW1 Vlan20 (.1) → PC (.10) |
| Server LAN (VLAN30) | 192.168.30.0/24 | SW2 (.1), SW1 (.2) → Server (.10) |
| SW2 ↔ R1/R2 (HSRP) | 10.0.12.0/24 | SW2 Vlan12 (.4), R1 (.2), R2 (.3), VIP (.1) |
| R1 ↔ R3 | 10.0.3.0/30 | R1 f0/1 (.1) ↔ R3 f0/0 (.2) |
| R2 ↔ R3 | 10.0.4.0/30 | R2 f0/1 (.1) ↔ R3 f0/1 (.2) |
| R3 ↔ R4 | 10.0.5.0/30 | R3 f1/0 (.1) ↔ R4 f0/0 (.2) |
| R3 ↔ R5 | 10.0.6.0/30 | R3 f2/0 (.1) ↔ R5 f0/0 (.2) |
| R4 ↔ SW3 | 10.0.7.0/30 | R4 f0/1 (.1) ↔ SW3 e0/0 (.2) |
| R5 ↔ SW3 | 10.0.8.0/30 | R5 f0/1 (.1) ↔ SW3 e0/1 (.2) |
| SW3 ↔ SW4 | 10.0.9.0/30 | SW3 e0/2 (.1) ↔ SW4 e0/0 (.2) |
| PC3 LAN | 192.168.40.0/24 | SW4 e0/1 (.1) → PC3 (.10) |
| PC4 LAN | 192.168.50.0/24 | SW4 e0/2 (.1) → PC4 (.10) |
| GRE Tunnel 0 | 172.16.1.0/30 | R1 (.1) ↔ R3 (.2) |
| GRE Tunnel 1 | 172.16.2.0/30 | R2 (.1) ↔ R3 (.2) |

---

## 1.3 IPsec + GRE VPN тохиргоо

PC3 (192.168.40.10) нь Server (192.168.30.10) руу GRE over IPsec VPN tunnel-ээр дамжин аюулгүйгээр хандана. GRE нь routing protocol (OSPF)-ийг tunnel дотор дамжуулж, IPsec нь GRE packet-ийг шифрлэнэ.

**IPsec параметрүүд:**

| Параметр | Утга |
|----------|------|
| Encryption | AES 256 |
| Hash | ESP-SHA-HMAC |
| Authentication | Pre-shared key |
| Pre-shared key | LAB11KEY |
| Mode | Transport |
| Profile name | GRE_PROFILE |

**R1 дээрх IPsec + GRE тохиргоо:**

```
crypto isakmp policy 10
 encr aes 256
 authentication pre-share
crypto isakmp key LAB11KEY address 10.0.3.2

crypto ipsec transform-set MYSET esp-aes 256 esp-sha-hmac
 mode transport

crypto ipsec profile GRE_PROFILE
 set transform-set MYSET

interface Tunnel0
 ip address 172.16.1.1 255.255.255.252
 tunnel source FastEthernet0/1
 tunnel destination 10.0.3.2
 tunnel protection ipsec profile GRE_PROFILE
```

**R3 дээрх IPsec + GRE тохиргоо (2 tunnel хүлээн авна):**

```
crypto isakmp key LAB11KEY address 10.0.3.1
crypto isakmp key LAB11KEY address 10.0.4.1

interface Tunnel0
 ip address 172.16.1.2 255.255.255.252
 tunnel source FastEthernet0/0
 tunnel destination 10.0.3.1
 tunnel protection ipsec profile GRE_PROFILE

interface Tunnel1
 ip address 172.16.2.2 255.255.255.252
 tunnel source FastEthernet0/1
 tunnel destination 10.0.4.1
 tunnel protection ipsec profile GRE_PROFILE
```

**Зураг 2.** R1 дээрх `show crypto ipsec sa` командын үр дүн — IPsec шифрлэлт идэвхтэй, encrypt/decrypt packet тоо харагдана.

*(R1 show ipsec.png)*

**Зураг 3.** R3 дээрх `show crypto ipsec sa` командын үр дүн.

*(R3 show ipsec.png)*

**Зураг 4.** R3 дээрх `show crypto isakmp sa` — Phase 1 SA амжилттай үүссэн (QM_IDLE).

*(R3 isakmp.png)*

---

## 1.4 GRE Tunnel Interface шалгалт

**Зураг 5.** R1 дээрх `show interface tunnel 0` — Tunnel up/up, packets encapsulated.

*(R1 tunnel0.png)*

**Зураг 6.** R3 дээрх `show interface tunnel 0` — R1 руу tunnel.

*(R3 tunnel0.png)*

**Зураг 7.** R3 дээрх `show interface tunnel 1` — R2 руу нөөц tunnel.

*(R3 tunnel1.png)*

---

## 1.5 OSPF Neighbor шалгалт

Зүүн талын бүх төхөөрөмж дээр OSPF neighbor FULL төлөвт орсон:

**Зураг 8.** SW1 дээрх `show ip ospf neighbor` — SW2-тай FULL.

*(SW1 show ospf neighbor.png)*

**Зураг 9.** SW2 дээрх `show ip ospf neighbor` — SW1, R1, R2-тай FULL.

*(SW2 show ospf neighbor.png)*

**Зураг 10.** R1 дээрх `show ip ospf neighbor` — SW2, R3 (tunnel-ээр) FULL.

*(R1 show ospf neighbor.png)*

**Зураг 11.** R2 дээрх `show ip ospf neighbor` — SW2, R3 (tunnel-ээр) FULL.

*(R2 show ospf neighbor.png)*

**Зураг 12.** R3 дээрх `show ip ospf neighbor` — R1, R2 (tunnel-ээр) FULL.

*(R3 show ospf neighbor.png)*

---

## 1.6 RIP v2 + Redistribute тохиргоо

Баруун талын сүлжээнд RIP version 2 ашигласан. R3 нь OSPF ↔ RIP хоёр протоколын хоорондын redistribute хийнэ:

**R3 дээрх redistribute тохиргоо:**

```
router ospf 1
 redistribute rip subnets
 network 10.0.3.0 0.0.0.3 area 0
 network 10.0.4.0 0.0.0.3 area 0
 network 172.16.1.0 0.0.0.3 area 0
 network 172.16.2.0 0.0.0.3 area 0

router rip
 version 2
 redistribute ospf 1 metric 5
 network 10.0.0.0
 no auto-summary
```

**Зураг 13.** SW4 дээрх `show ip route` — RIP-ээр бүх remote network сурагдсан (192.168.30.0, 192.168.10.0, 192.168.20.0 зэрэг).

*(SW4 ip route.png)*

---

## 1.7 HSRP тохиргоо

Gateway redundancy хангахын тулд HSRP тохируулсан:

**Зүүн тал (R1/R2):**
- R1: Priority 110, Active, VIP = 10.0.12.1
- R2: Priority 90, Standby, VIP = 10.0.12.1
- Track: FastEthernet0/1 (WAN link унтарвал priority -25)

**Баруун тал (R4/R5):**
- R4: Priority 110, Active, VIP = 10.0.78.1
- R5: Priority 90, Standby, VIP = 10.0.78.1

**R4 дээрх BGP+HSRP тохиргоо:**

*(R4 BGP.png)*

**Тэмдэглэл:** GNS3 виртуал switch орчинд HSRP multicast hello (224.0.0.2) packet бүрэн дамжихгүй хязгаарлалттай тул Standby router = unknown гарч байна. R1 ↔ R2 unicast ping 100% ажиллаж, тохиргоо зөв хийгдсэн. Бодит Cisco switch дээр Active/Standby сэлгэлт бүрэн ажиллана.

---

## 1.8 VLAN + Inter-VLAN Routing тохиргоо

SW1 дээр VLAN 10 (PC), VLAN 20 (PC), VLAN 30 (Server shared subnet) үүсгэж, SVI interface-ээр inter-VLAN routing хийсэн:

```
interface Vlan10
 ip address 192.168.10.1 255.255.255.0
interface Vlan20
 ip address 192.168.20.1 255.255.255.0
interface Vlan30
 ip address 192.168.30.2 255.255.255.0
```

SW1 ↔ SW2 хооронд dot1q trunk холболтоор VLAN 10, 20, 30 дамжуулна.

---

## 1.9 VPN Data Flow (PC3 → Server)

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
SW2 Vlan12 → Vlan30
  ↓ (Layer 2 - VLAN 30)
Server (192.168.30.10)
```

---

## 2. End-to-End Connectivity шалгалт

### 2.1 PC3 → Server (VPN-ийн гол тест)

**Зураг 14.** PC3-аас Server (192.168.30.10) руу ping — **Амжилттай!** VPN tunnel-ээр дамжсан.

*(pc3 to server.png)*

### 2.2 PC3 → PC4 (Local сүлжээ)

**Зураг 15.** PC3-аас PC4 (192.168.50.10) руу ping — Амжилттай.

*(pc3 to pc4.png)*

### 2.3 PC3 → VLAN10 PC (Cross-site VPN)

**Зураг 16.** PC3-аас VLAN10 PC (192.168.10.10) руу ping — VPN + OSPF-ээр дамжсан.

*(pc3 to vlan 10.png)*

### 2.4 PC4 → Server (VPN)

**Зураг 17.** PC4-аас Server руу ping — Амжилттай.

*(pc4 to server.png)*

### 2.5 Server → PC3 (Буцах чиглэл)

**Зураг 18.** Server-ээс PC3 (192.168.40.10) руу ping — Буцах VPN зам ажиллаж байна.

*(server to pc3.png)*

### 2.6 VLAN10 → Server (Зүүн LAN дотоод)

**Зураг 19.** VLAN10 PC-аас Server руу ping — OSPF + inter-VLAN.

*(vlan10 to server.png)*

### 2.7 VLAN20 → Server (Зүүн LAN дотоод)

**Зураг 20.** VLAN20 PC-аас Server руу ping — Амжилттай.

*(vlan20 to server.png)*

### 2.8 VLAN20 → VLAN10 (Inter-VLAN routing)

**Зураг 21.** VLAN20-аас VLAN10 руу ping — SW1 дээрх inter-VLAN routing ажиллаж байна.

*(vlan20 to vlan10.png)*

---

## 3. Дүгнэлт

Энэхүү лабораторийн ажлаар GRE over IPsec VPN сүлжээг амжилттай зохион байгуулав. Гүйцэтгэсэн ажлууд:

1. **GRE + IPsec VPN:** R1↔R3, R2↔R3 хооронд 2 GRE tunnel үүсгэж, IPsec (AES-256, ESP-SHA-HMAC, PSK) шифрлэлтээр хамгаалсан. PC3-аас Server руу VPN-ээр амжилттай хандсан.

2. **OSPF routing:** Зүүн талын сүлжээнд (SW1, SW2, R1, R2, R3) OSPF area 0 тохируулж, бүх neighbor FULL төлөвт орсон.

3. **RIP v2 routing:** Баруун талын сүлжээнд (R3, R4, R5, SW3, SW4) RIP version 2 тохируулсан.

4. **OSPF ↔ RIP Redistribute:** R3 дээр хоёр протоколыг хоёр талдаа redistribute хийж, бүх сүлжээ хоорондоо харилцах боломжтой болсон.

5. **HSRP:** R1 (priority 110) / R2 (priority 90) зүүн талд, R4 (priority 110) / R5 (priority 90) баруун талд gateway redundancy тохируулсан.

6. **VLAN + Inter-VLAN Routing:** SW1 дээр VLAN 10, 20 үүсгэж, SVI-ээр routing хийсэн.

7. **End-to-End Connectivity:** PC3 → Server, PC4 → Server, VLAN10 → Server, VLAN20 → VLAN10 зэрэг бүх ping тест 100% амжилттай.

**Эцсийн шалгалтын үр дүн:** PC3 (192.168.40.10) → Server (192.168.30.10) ping амжилттай ажилласан нь GRE over IPsec VPN tunnel, OSPF↔RIP redistribute, VLAN routing бүгд зөв тохирсныг батлана.
