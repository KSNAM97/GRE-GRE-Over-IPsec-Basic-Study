# 07. GRE-Over-IPsec (Dynamic / EIGRP)

## 요구사항

GIT-A의 192.168.10.0/24 ↔ GIT-B의 192.168.20.0/24 통신 시 정보보호 적용.

| Phase 1 | 값 |
| --- | --- |
| 인증방식 | pre-share |
| 암호화 | 3DES |
| 인증 | SHA |
| 키 교환 | Diffie-Hellman 2 |
| 키 교환 주기 | 2시간 (7200초) |

| Phase 2 | 값 |
| --- | --- |
| IPsec Protocol | ESP |
| 암호화 | AES |
| 인증 | SHA-HMAC |

---

## 1) Tunnel 인터페이스 crypto map (사설망 ACL 기준)

### GIT-A (R1)

```
access-list 101 permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
!
crypto isakmp enable
crypto isakmp policy 10
 authentication pre-share
 encryption 3des
 hash sha
 group 2
 lifetime 7200
!
crypto isakmp key 0 cisco address 121.160.20.2
!
crypto ipsec transform-set IPSEC esp-aes esp-sha-hmac
!
crypto map IPSEC_VPN 10 ipsec-isakmp
 set peer 121.160.20.2
 set transform-set IPSEC
 match address 101
!
interface tunnel 12
 crypto map IPSEC_VPN
```

### GIT-B (R2)

```
access-list 101 permit ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
!
crypto isakmp enable
crypto isakmp policy 10
 authentication pre-share
 encryption 3des
 hash sha
 group 2
 lifetime 7200
!
crypto isakmp key 0 cisco address 121.160.10.1
!
crypto ipsec transform-set IPSEC esp-aes esp-sha-hmac
!
crypto map IPSEC_VPN 10 ipsec-isakmp
 set peer 121.160.10.1
 set transform-set IPSEC
 match address 101
!
interface tunnel 12
 crypto map IPSEC_VPN
```

### 정보 확인

```
GIT-A# show crypto isakmp peer
Peer: 121.160.20.2 Port: 500 Local: 121.160.10.1
 Phase1 id: 121.160.20.2

GIT-A# show crypto engine connection active
   ID Interface  Type  Algorithm   Encrypt  Decrypt  IP-Address
    1 Tu12       IPsec AES+SHA           0        4  121.160.10.1
    2 Tu12       IPsec AES+SHA           4        0  121.160.10.1
 1001 Tu12       IKE   MD5+3DES         0        0  121.160.10.1
```

### 패킷 구조 (ICMP — 사설망 보호됨)

```
# PC1 -----> PC2 (ICMP Request)
                              |---------- 암호화 ----------|
                         |------------- 인증 -------------|
======================================================================================
 SA:121.160.10.1 | GRE | SA:121.160.10.1 | ESP | SA:192.168.10.1 | ICMP   | ESP     | ESP            |
 DA:121.160.20.2 |     | DA:121.160.20.2 |     | DA:192.168.20.2 | Request| Trailer | Authentication |
======================================================================================
```

---

## 문제점 — EIGRP(Multicast)는 보호되지 않음

```
# EIGRP는 224.0.0.10 (Multicast) 사용 → IPsec은 Unicast만 보호 가능
GIT-A# (EIGRP Hello가 평문으로 노출됨)
```

### EIGRP를 ACL에 추가하면? → 인접성 단절

```
# GIT-A (R1)
access-list 101 permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
access-list 101 permit eigrp any any
```

```
%CRYPTO-4-RECVD_PKT_NOT_IPSEC: Rec'd packet not an IPSEC packet.
   (ip) vrf/dest_addr= /224.0.0.10, src_addr= 192.168.12.2, prot= 88
%DUAL-5-NBRCHANGE: IP-EIGRP(0) 100: Neighbor 192.168.12.2 (Tunnel12) is down: holding time expired
```

> IPsec은 Unicast만 지원 → Multicast(EIGRP/OSPF)를 ACL에 직접 넣으면 인접성이 끊긴다.

---

## 2) 해결책 — GRE Header 기준 보호 (물리 인터페이스 crypto map)

> EIGRP를 ACL에 추가하지 않고, **GRE 자체를 보호 범위로 지정**한다.
> GRE 내부에 EIGRP(Multicast)가 캡슐화되므로 결과적으로 함께 보호된다.

### 기존 설정 해제

```
# R1
no access-list 101
interface tunnel 12
 no crypto map IPSEC_VPN

# R2
interface tunnel 12
 no crypto map IPSEC_VPN
```

### GRE 기준 ACL + 물리 인터페이스 적용

```
# GIT-A (R1)
access-list 101 permit gre host 121.160.10.1 host 121.160.20.2
!
interface serial 1/0.10
 crypto map IPSEC_VPN
```

```
# GIT-B (R2)
access-list 101 permit gre host 121.160.20.2 host 121.160.10.1
!
interface serial 1/0.20
 crypto map IPSEC_VPN
```

### 패킷 구조 (GRE 전체가 암호화 → EIGRP 포함)

```
# GIT-A -----> GIT-B (EIGRP)
       |------------------ 암호화 ------------------|
=============================================================================
 SA:121.160.10.1 | ESP | SA:121.160.10.1 | GRE | SA:192.168.12.1 | EIGRP   |
                 |     | DA:121.160.20.2 |     | DA:224.0.0.10   | Hello   |
=============================================================================
```
