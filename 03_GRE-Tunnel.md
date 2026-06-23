# 03. GRE Tunnel 생성

## 요구사항

- GIT-A ↔ GIT-B 구간을 GRE Tunnel로 연결
- Tunnel IP 주소: `172.16.10.0/24`
- Tunnel source/destination: ISP-1, ISP-4에 직접 연결된 물리 인터페이스 IP 사용
  - GIT-A: source `121.160.10.1`, destination `121.160.20.2`
  - GIT-B: source `121.160.20.2`, destination `121.160.10.1`

---

## 설정

### GIT-A (R1)

```
interface tunnel 12
 ip address 172.16.10.1 255.255.255.0
 tunnel source 121.160.10.1
 tunnel destination 121.160.20.2
```

### GIT-B (R2)

```
interface tunnel 12
 ip address 172.16.10.2 255.255.255.0
 tunnel source 121.160.20.2
 tunnel destination 121.160.10.1
```

---

## 정보 확인

```
GIT-A# show ip route
     172.16.0.0/24 is subnetted, 1 subnets
C       172.16.10.0 is directly connected, Tunnel12

GIT-A# ping 172.16.10.2
```

```
GIT-B# show ip route
     172.16.0.0/24 is subnetted, 1 subnets
C       172.16.10.0 is directly connected, Tunnel12

GIT-B# ping 172.16.10.1
```

---

## 패킷 구조 (GRE Encapsulation)

```
# GIT-A -----> GIT-B
========================================================
 SA : 121.160.10.1 |  GRE  | SA : 172.16.10.1 |  ICMP   |
 DA : 121.160.20.2 |       | DA : 172.16.10.2 | Request |
========================================================

# GIT-A <----- GIT-B
========================================================
 SA : 121.160.20.2 |  GRE  | SA : 172.16.10.2 |  ICMP   |
 DA : 121.160.10.1 |       | DA : 172.16.10.1 |  Reply  |
========================================================
```

> GRE는 원본 패킷에 새로운 외부 IP Header(공인 IP)를 추가하여 공중망을 통과시킨다.
