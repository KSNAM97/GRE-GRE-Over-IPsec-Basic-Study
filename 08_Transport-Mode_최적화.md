# 08. Transport Mode 최적화

## GRE over IPsec에서 Transport Mode를 사용하는 이유

GRE는 이미 원본 패킷에 외부 IP Header를 추가하여 공중망 전달 형태로 캡슐화한다.

```
# GRE 캡슐화 결과
[GRE용 외부 IP Header] [GRE Header] [원본 IP Header] [Data]
```

여기에 **IPsec Tunnel Mode** 를 적용하면 IPsec이 또 다른 외부 IP Header를 추가 → 외부 IP Header 2개 생성(중복).

```
# Tunnel Mode (중복)
[IPsec용 새 외부 IP] [ESP] [GRE용 외부 IP] [GRE] [원본 IP] [Data]
```

반면 **Transport Mode** 는 기존 GRE 외부 IP Header를 그대로 사용한다.

```
# Transport Mode (효율적)
[기존 GRE용 외부 IP] [ESP] [GRE] [원본 IP] [Data]
```

---

## 사용 이유 요약

1. **중복 IP Header 생성 방지** — GRE가 이미 터널 출발지/목적지 IP를 가진 외부 헤더 생성.
2. **패킷 오버헤드 감소** — IPv4 Header 20Byte 추가를 줄여 대역폭 효율 향상.
3. **Fragmentation 가능성 감소** — 헤더 누적으로 MTU 초과 시 발생하는 분할/지연/성능 저하 완화.
4. **역할 분리** — GRE = 터널링/멀티캐스트 전달, IPsec = 암호화·무결성·인증만 담당.
5. **GRE 내부 전체 보호** — 외부 IP는 평문이나 GRE Header부터 내부 전체가 보호됨.

---

## 설정 (GRE 기준 보호 + Transport Mode)

> ⚠️ 실제 IOS 명령은 `mode transport` 입니다. (원문 `tranport` 오타 교정)

### GIT-A (R1)

```
access-list 101 permit gre host 121.160.10.1 host 121.160.20.2
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
 mode transport
!
crypto map IPSEC_VPN 10 ipsec-isakmp
 set peer 121.160.20.2
 set transform-set IPSEC
 match address 101
!
interface serial 1/0.10
 crypto map IPSEC_VPN
```

### GIT-B (R2)

```
access-list 101 permit gre host 121.160.20.2 host 121.160.10.1
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
 mode transport
!
crypto map IPSEC_VPN 10 ipsec-isakmp
 set peer 121.160.10.1
 set transform-set IPSEC
 match address 101
!
interface serial 1/0.20
 crypto map IPSEC_VPN
```

---

## 정보 확인 / 패킷 구조

```
GIT-A# show crypto engine connection active
   ID Interface  Type  Algorithm   Encrypt  Decrypt  IP-Address
    1 Tu12       IPsec AES+SHA           0        4  121.160.10.1
    2 Tu12       IPsec AES+SHA           4        0  121.160.10.1
```

```
# GIT-A -----> GIT-B (ICMP)  -- Transport Mode
       |-------------- 암호화 --------------|
===========================================================================
 SA:121.160.10.1 | ESP |  GRE  | SA:192.168.10.1 | ICMP    |
 DA:121.160.20.2 |     |       | DA:192.168.20.2 | Request |
===========================================================================

# GIT-A -----> GIT-B (EIGRP) -- Transport Mode
       |-------------- 암호화 --------------|
===========================================================================
 SA:121.160.10.1 | ESP |  GRE  | SA:192.168.12.1 | EIGRP   |
 DA:121.160.20.2 |     |       | DA:224.0.0.10   | Hello   |
===========================================================================
```
