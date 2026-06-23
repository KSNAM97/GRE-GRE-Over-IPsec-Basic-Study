# 09. 보안 강화 (참고) — AES-256-GCM 업그레이드

> 학습 토폴로지 기본값(3DES/SHA-1/DH2)은 실습용입니다.
> 실제 환경에서는 아래 현대 표준으로 상향하는 것이 권장됩니다.
> (장비/IOS 버전에 따라 GCM 및 DH14 지원 여부가 다를 수 있습니다.)

---

## 보안 사양 요약

| Phase 1 (IKE/ISAKMP) | 값 |
| --- | --- |
| 인증방식 | Pre-Shared Key |
| 암호화 | AES-256 |
| 인증 | SHA-256 |
| 키 교환 | Diffie-Hellman Group 14 (2048-bit) |
| 키 교환 주기 | 2시간 (7200초) |

| Phase 2 (IPsec) | 값 |
| --- | --- |
| IPsec Protocol | ESP |
| 암호화 | AES-256-GCM |
| 인증 | GCM 자체 결합 인증 태그 (별도 HMAC 미사용) |

---

## 설정 근거 (요약)

- **AES-256 / SHA-256**: 3DES·SHA-1은 암호학적 취약점으로 KISA·NIST에서 사용 중단(Deprecated) 권고 → 무차별 대입·해시 충돌 공격 차단.
- **DH Group 14 (2048-bit)**: Group 2(1024-bit)는 연산 성능 향상으로 안전성 저하 → 초기 키 교환 단계의 MitM·키 탈취 방어.
- **AES-GCM (AEAD)**: 암호화와 무결성 검증을 한 단계로 결합 → 오버헤드 감소, 병렬 연산 지원, 재전송(Replay) 공격 즉시 탐지·폐기.

---

## 설정 예시

### GIT-A (R1)

```
! --- 1. 흥미 트래픽 정의 (보호 대상 사설 대역) ---
access-list 101 permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
!
! --- 2. ISAKMP (Phase 1) ---
crypto isakmp enable
crypto isakmp policy 10
 authentication pre-share
 encryption aes 256          ! 3des -> AES-256 상향
 hash sha256                 ! sha(sha1) -> sha256 상향
 group 14                    ! Group 2 -> Group 14 (2048-bit)
 lifetime 7200
!
! --- 3. 인증 키 / 대향 라우터 공인 IP 매핑 ---
crypto isakmp key 0 cisco address 121.160.20.2
!
! --- 4. IPsec (Phase 2) : AES-256-GCM (별도 esp-sha-hmac 미사용) ---
crypto ipsec transform-set IPSEC esp-aes 256 esp-gcm
!
! --- 5. Crypto Map ---
crypto map IPSEC_VPN 10 ipsec-isakmp
 set peer 121.160.20.2
 set transform-set IPSEC
 match address 101
!
! --- 6. 터널 인터페이스에 적용 ---
interface tunnel 12
 crypto map IPSEC_VPN
```

### GIT-B (R2)

```
! --- 1. 흥미 트래픽 정의 (방향 반대) ---
access-list 101 permit ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
!
! --- 2. ISAKMP (Phase 1) — GIT-A와 반드시 일치 ---
crypto isakmp enable
crypto isakmp policy 10
 authentication pre-share
 encryption aes 256
 hash sha256
 group 14
 lifetime 7200
!
! --- 3. 인증 키 / GIT-A 공인 IP 매핑 ---
crypto isakmp key 0 cisco address 121.160.10.1
!
! --- 4. IPsec (Phase 2) — GIT-A와 반드시 일치 ---
crypto ipsec transform-set IPSEC esp-aes 256 esp-gcm
!
! --- 5. Crypto Map ---
crypto map IPSEC_VPN 10 ipsec-isakmp
 set peer 121.160.10.1
 set transform-set IPSEC
 match address 101
!
! --- 6. 터널 인터페이스에 적용 ---
interface tunnel 12
 crypto map IPSEC_VPN
```
