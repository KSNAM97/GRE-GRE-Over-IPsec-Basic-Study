# 01. 공중망 구성 (OSPF)

## 요구사항

- ISP-1 ~ ISP-4 구간에 OSPF Routing Protocol 사용
- OSPF Process = 1, Area = 0
- Router-ID = `XX.XX.XX.XX` (X = Router 번호)
- ISP의 Serial, Loopback 0 네트워크는 OSPF에 포함
- 모든 네트워크는 인터페이스 SubnetMask로 확인 (`ip ospf network point-to-point`)
- GIT-A, GIT-B는 동적 라우팅 미구성
- OSPF 업데이트가 필요한 인터페이스로만 OSPF Packet 송신 (`passive-interface`)

---

## 설정

### ISP-1 (R3)

```
router ospf 1
 router-id 11.11.11.11
 passive-interface default
 no passive-interface serial 1/0.10
 no passive-interface serial 1/0.12
 network 100.100.11.0 0.0.0.255 area 0
 network 121.160.10.0 0.0.0.255 area 0
 network 121.160.12.0 0.0.0.255 area 0
!
interface loopback 0
 ip ospf network point-to-point
```

### ISP-2 (R4)

```
router ospf 1
 router-id 22.22.22.22
 passive-interface default
 no passive-interface serial 1/0.12
 no passive-interface serial 1/0.23
 network 100.100.12.0 0.0.0.255 area 0
 network 121.160.12.0 0.0.0.255 area 0
 network 121.160.23.0 0.0.0.255 area 0
!
interface loopback 0
 ip ospf network point-to-point
```

### ISP-3 (R5)

```
router ospf 1
 router-id 33.33.33.33
 passive-interface default
 no passive-interface serial 1/0.23
 no passive-interface serial 1/0.34
 network 100.100.13.0 0.0.0.255 area 0
 network 121.160.23.0 0.0.0.255 area 0
 network 121.160.34.0 0.0.0.255 area 0
!
interface loopback 0
 ip ospf network point-to-point
```

### ISP-4 (R6)

```
router ospf 1
 router-id 44.44.44.44
 passive-interface default
 no passive-interface serial 1/0.20
 no passive-interface serial 1/0.34
 network 100.100.14.0 0.0.0.255 area 0
 network 121.160.20.0 0.0.0.255 area 0
 network 121.160.34.0 0.0.0.255 area 0
!
interface loopback 0
 ip ospf network point-to-point
```

---

## 정보 확인

```
ISP-1# show ip ospf neighbor   [인접성 1개 확인]
ISP-2# show ip ospf neighbor   [인접성 2개 확인]
ISP-3# show ip ospf neighbor   [인접성 2개 확인]
ISP-4# show ip ospf neighbor   [인접성 1개 확인]
```

```
ISP-1# show ip route
     100.0.0.0/24 is subnetted, 4 subnets
C       100.100.11.0 is directly connected, Loopback0
O       100.100.12.0 [110/65]  via 121.160.12.2, Serial1/0.12
O       100.100.13.0 [110/129] via 121.160.12.2, Serial1/0.12
O       100.100.14.0 [110/193] via 121.160.12.2, Serial1/0.12
     121.0.0.0/24 is subnetted, 5 subnets
O       121.160.20.0 [110/256] via 121.160.12.2, Serial1/0.12   <-- GIT-B/ISP-4 연결망
O       121.160.23.0 [110/128] via 121.160.12.2, Serial1/0.12
C       121.160.10.0 is directly connected, Serial1/0.10        <-- GIT-A/ISP-1 연결망
C       121.160.12.0 is directly connected, Serial1/0.12
O       121.160.34.0 [110/192] via 121.160.12.2, Serial1/0.12
```

```
ISP-4# show ip route
     100.0.0.0/24 is subnetted, 4 subnets
O       100.100.11.0 [110/193] via 121.160.34.3, Serial1/0.34
O       100.100.12.0 [110/129] via 121.160.34.3, Serial1/0.34
O       100.100.13.0 [110/65]  via 121.160.34.3, Serial1/0.34
C       100.100.14.0 is directly connected, Loopback0
     121.0.0.0/24 is subnetted, 5 subnets
C       121.160.20.0 is directly connected, Serial1/0.20        <-- GIT-B/ISP-4 연결망
O       121.160.23.0 [110/128] via 121.160.34.3, Serial1/0.34
O       121.160.10.0 [110/256] via 121.160.34.3, Serial1/0.34   <-- GIT-A/ISP-1 연결망
O       121.160.12.0 [110/192] via 121.160.34.3, Serial1/0.34
C       121.160.34.0 is directly connected, Serial1/0.34
```
