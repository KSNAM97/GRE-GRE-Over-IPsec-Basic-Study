# 06. Dynamic Routing (EIGRP) — 터널 위 사설망 구성

## 요구사항

- GRE Tunnel IP는 `192.168.12.0/24` 사용 (Dynamic 시나리오)
- EIGRP AS = 100, auto-summary 미사용
- 업데이트가 필요한 인터페이스로만 EIGRP Packet 송신 (`passive-interface`)
- GIT-A/B 사설망, Loopback 0, Tunnel 12 네트워크를 EIGRP에 포함

> ※ 본 챕터는 Tunnel IP가 `192.168.12.0/24` 로 설정된 토폴로지를 사용합니다.

---

## GRE Tunnel (192.168.12.0/24)

### GIT-A (R1)

```
interface tunnel 12
 ip address 192.168.12.1 255.255.255.0
 tunnel source 121.160.10.1
 tunnel destination 121.160.20.2
```

### GIT-B (R2)

```
interface tunnel 12
 ip address 192.168.12.2 255.255.255.0
 tunnel source 121.160.20.2
 tunnel destination 121.160.10.1
```

---

## EIGRP 설정

### GIT-A (R1)

```
router eigrp 100
 no auto-summary
 passive-interface default
 no passive-interface tunnel 12
 network 100.100.1.0 0.0.0.255
 network 192.168.10.0
 network 192.168.12.0
```

### GIT-B (R2)

```
router eigrp 100
 no auto-summary
 passive-interface default
 no passive-interface tunnel 12
 network 100.100.2.0 0.0.0.255
 network 192.168.20.0
 network 192.168.12.0
```

---

## 정보 확인

```
GIT-A# show ip eigrp neighbors
H  Address       Interface  Hold Uptime  SRTT  RTO   Q  Seq
0  192.168.12.2  Tu12        11  00:00:03 106  5000  0  3

GIT-A# show ip route
C    192.168.12.0/24 is directly connected, Tunnel12
      100.0.0.0/24 is subnetted, 2 subnets
C        100.100.1.0 is directly connected, Loopback0
D        100.100.2.0 [90/297372416] via 192.168.12.2, Tunnel12
C    192.168.10.0/24 is directly connected, FastEthernet0/0          <-- GIT-A 사설망
D    192.168.20.0/24 [90/297270016] via 192.168.12.2, Tunnel12       <-- GIT-B 사설망
```

```
PC1# ping 192.168.20.2
PC2# ping 192.168.10.1
```

---

## 패킷 구조 (EIGRP Hello, Multicast 224.0.0.10)

```
# GIT-A ----> GIT-B (EIGRP Hello)
=================================================================
 SA:121.160.10.1 |  GRE  | SA:192.168.12.1 |   EIGRP Hello      |
 DA:121.160.20.2 |       | DA:224.0.0.10   |                    |
=================================================================
```

> EIGRP는 Multicast(224.0.0.10)를 사용한다 → 다음 챕터에서 IPsec 보호 이슈 발생.
