# 02. Default-Route (GIT 내부망 → 외부 통신)

## 요구사항

- 내부 GIT Router에서 외부 네트워크로 통신하기 위한 Static Default-route 생성
- GIT-A, GIT-B는 동적 라우팅을 쓰지 않으므로 기본 경로를 ISP로 향하게 설정

---

## 설정

### GIT-A (R1)

```
ip route 0.0.0.0 0.0.0.0 121.160.10.11
```

### GIT-B (R2)

```
ip route 0.0.0.0 0.0.0.0 121.160.20.4
```

---

## 정보 확인

```
GIT-A# show ip route
      100.0.0.0/24 is subnetted, 1 subnets
C        100.100.1.0 is directly connected, Loopback0
C    192.168.10.0/24 is directly connected, FastEthernet0/0
      121.0.0.0/24 is subnetted, 1 subnets
C        121.160.10.0 is directly connected, Serial1/0.10
S*   0.0.0.0/0 [1/0] via 121.160.10.11

GIT-A# ping 121.160.20.2   : GIT-B WAN 네트워크로 통신 확인
```

```
GIT-B# show ip route
      100.0.0.0/24 is subnetted, 1 subnets
C        100.100.2.0 is directly connected, Loopback0
C    192.168.20.0/24 is directly connected, FastEthernet0/1
      121.0.0.0/24 is subnetted, 1 subnets
C        121.160.20.0 is directly connected, Serial1/0.20
S*   0.0.0.0/0 [1/0] via 121.160.20.4

GIT-B# ping 121.160.10.1   : GIT-A WAN 네트워크로 통신 확인
```
