# 05. GRE-Over-IPsec (Static Route 기반)

## 요구사항

GIT-A의 192.168.10.0/24 ↔ GIT-B의 192.168.20.0/24 통신 시 정보보호(암호화) 적용.

| Phase 1 | 값 |
| --- | --- |
| 인증방식 | 사전 인증 (pre-share) |
| 암호화 | AES |
| 인증 | SHA |
| 키 교환 | Diffie-Hellman 2 |
| 키 교환 주기 | 30분 (1800초) |

| Phase 2 | 값 |
| --- | --- |
| IPsec Protocol | ESP |
| 암호화 | 3DES |
| 인증 | MD5-HMAC |

---

## 1) Tunnel 인터페이스에 crypto map 적용

### GIT-A (R1)

```
access-list 101 permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
!
crypto isakmp enable
crypto isakmp policy 10
 authentication pre-share
 encryption aes
 hash sha
 group 2
 lifetime 1800
!
crypto isakmp key 0 cisco address 121.160.20.2
!
crypto ipsec transform-set IPSEC esp-3des esp-md5-hmac
!
crypto map GIT-A_IPSEC 10 ipsec-isakmp
 set peer 121.160.20.2
 set transform-set IPSEC
 match address 101
!
interface tunnel 12
 crypto map GIT-A_IPSEC
```

### GIT-B (R2)

```
access-list 101 permit ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
!
crypto isakmp enable
crypto isakmp policy 10
 authentication pre-share
 encryption aes
 hash sha
 group 2
 lifetime 1800
!
crypto isakmp key 0 cisco address 121.160.10.1
!
crypto ipsec transform-set IPSEC esp-3des esp-md5-hmac
!
crypto map GIT-B_IPSEC 10 ipsec-isakmp
 set peer 121.160.10.1
 set transform-set IPSEC
 match address 101
!
interface tunnel 12
 crypto map GIT-B_IPSEC
```

### 정보 확인

```
GIT-A# show crypto isakmp peer
Peer: 121.160.20.2 Port: 500 Local: 121.160.10.1
 Phase1 id: 121.160.20.2

GIT-A# show crypto engine connections active
   ID Interface  Type  Algorithm     Encrypt  Decrypt  IP-Address
    1 Tu12       IPsec 3DES+MD5            0        4  121.160.10.1
    2 Tu12       IPsec 3DES+MD5            4        0  121.160.10.1
 1001 Tu12       IKE   SHA+AES            0        0  121.160.10.1
```

### 패킷 구조 (tunnel mode, Tunnel 인터페이스 적용)

```
====================================================================================
 (3)New IP Header |  GRE  | (2)New IP Header |  ESP  | (1)원본 IP Header |  Data   |
------------------------------------------------------------------------------------
 SA:121.160.10.1  |       | SA:121.160.10.1  |       | SA:192.168.10.1   | ICMP    |
 DA:121.160.20.2  |       | DA:121.160.20.2  |       | DA:192.168.20.2   | Request |
====================================================================================
1) 원본  2) IPsec에 의한 new IP header  3) Tunnel에 의한 new IP header
```

---

## 2) 물리 인터페이스에 crypto map 적용

> Tunnel 적용 시에는 GRE Header가 ICMP보다 바깥에 위치하지만,
> 물리 인터페이스 적용 시에는 ESP가 가장 바깥쪽에 위치한다.

### GIT-A (R1)

```
no access-list 101 permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
access-list 101 permit gre host 121.160.10.1 host 121.160.20.2
!
interface tunnel 12
 no crypto map GIT-A_IPSEC
!
interface serial 1/0.10
 crypto map GIT-A_IPSEC
```

### GIT-B (R2)

```
no access-list 101 permit ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
access-list 101 permit gre host 121.160.20.2 host 121.160.10.1
!
interface tunnel 12
 no crypto map GIT-B_IPSEC
!
interface serial 1/0.20
 crypto map GIT-B_IPSEC
```

### 정보 확인

```
GIT-A# show crypto isakmp peers
Peer: 121.160.20.2 Port: 500 Local: 121.160.10.1
 Phase1 id: 121.160.20.2

GIT-A# show crypto engine connections active
   ID Interface  Type  Algorithm     Encrypt  Decrypt  IP-Address
    9 Se1/0.10   IPsec 3DES+MD5            0        4  121.160.10.1
   10 Se1/0.10   IPsec 3DES+MD5            4        0  121.160.10.1
 1005 Se1/0.10   IKE   SHA+AES            0        0  121.160.10.1
```

### 패킷 구조 (tunnel mode, 물리 인터페이스 적용)

```
====================================================================================
 (3)New IP Header |  ESP  | (2)New IP Header |  GRE  | (1)원본 IP Header |  Data   |
------------------------------------------------------------------------------------
 SA:121.160.10.1  |       | SA:121.160.10.1  |       | SA:192.168.10.1   | ICMP    |
 DA:121.160.20.2  |       | DA:121.160.20.2  |       | DA:192.168.20.2   | Request |
====================================================================================
1) 원본  2) Tunnel에 의한 new IP header  3) IPsec에 의한 new IP header
```

---

## 3) 물리 인터페이스 + Transport Mode 변경

### 패킷 구조 (transport mode)

```
==================================================================
 (2)New IP Header |  ESP  |  GRE  | (1)원본 IP Header |  Data   |
------------------------------------------------------------------
 SA:121.160.10.1  |       |       | SA:192.168.10.1   | ICMP    |
 DA:121.160.20.2  |       |       | DA:192.168.20.2   | Request |
==================================================================
```

### GIT-A (R1)

```
access-list 101 permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
!
crypto isakmp enable
crypto isakmp policy 10
 authentication pre-share
 encryption aes
 hash sha
 group 2
 lifetime 1800
!
crypto isakmp key 0 cisco address 121.160.20.2
!
crypto ipsec transform-set IPSEC esp-3des esp-md5-hmac
 mode transport
!
crypto map GIT-A_IPSEC 10 ipsec-isakmp
 set peer 121.160.20.2
 set transform-set IPSEC
 match address 101
!
interface serial 1/0.10
 crypto map GIT-A_IPSEC
```

### GIT-B (R2)

```
access-list 101 permit ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
!
crypto isakmp enable
crypto isakmp policy 10
 authentication pre-share
 encryption aes
 hash sha
 group 2
 lifetime 1800
!
crypto isakmp key 0 cisco address 121.160.10.1
!
crypto ipsec transform-set IPSEC esp-3des esp-md5-hmac
 mode transport
!
crypto map GIT-B_IPSEC 10 ipsec-isakmp
 set peer 121.160.10.1
 set transform-set IPSEC
 match address 101
!
interface serial 1/0.20
 crypto map GIT-B_IPSEC
```
