# GRE & GRE-Over-IPsec Basic Study

![Cisco](https://img.shields.io/badge/Cisco-IOS-1BA0D7?logo=cisco&logoColor=white)
![GRE](https://img.shields.io/badge/GRE-Tunnel-blue)
![IPsec](https://img.shields.io/badge/IPsec-VPN-green)
![VPN](https://img.shields.io/badge/VPN-Site--to--Site-orange)
![Routing](https://img.shields.io/badge/Routing-OSPF%20%7C%20EIGRP-yellow)
![GNS3](https://img.shields.io/badge/GNS3-Dynamips-purple)
![License](https://img.shields.io/badge/License-MIT-lightgrey)

**Cisco GRE Tunnel & GRE-Over-IPsec (Site-to-Site VPN / Tunnel Mode / Transport Mode / Static & Dynamic Routing)**

> - **Part 1 — GRE Tunnel 기본 (공중망 구성 + GRE 터널)**
> - **Part 2 — GRE-Over-IPsec (Static Route 기반 정보보호)**
> - **Part 3 — GRE-Over-IPsec (Dynamic Routing / EIGRP 기반 정보보호)**

---

Cisco IOS 환경에서 **본사(GIT-A) ↔ 지사(GIT-B)** 를 공중망(ISP)을 경유하여
**GRE Tunnel** 로 연결하고, **IPsec** 으로 사설 트래픽을 암호화하는 과정을 정리한 학습 저장소입니다.
**GRE Encapsulation / Crypto Map / ISAKMP(Phase 1) / IPsec(Phase 2) / Tunnel & Transport Mode** 를
캡처(Wireshark) 기반 패킷 구조와 함께 단계별로 다룹니다.

|  항목  |  내용  |
| --- | --- |
| 장비 | Cisco IOS Router (c7200), Frame-Relay Switch |
| 주제 | GRE, IPsec, ISAKMP, ESP, Crypto Map, Tunnel/Transport Mode, OSPF, EIGRP |
| 환경 | GNS3 / Dynamips |
| 핵심 명령 | `interface tunnel`, `crypto isakmp policy`, `crypto ipsec transform-set`, `crypto map`, `mode transport` |

---

## 디렉터리 구조

```
GRE-OVER-IPSEC-STUDY/
└─ WAN/GRE-IPsec/
    ├─ 00_Topology_and_Base-Config.md
    ├─ 01_공중망구성_OSPF.md
    ├─ 02_Default-Route.md
    ├─ 03_GRE-Tunnel.md
    ├─ 04_Static-Route_사설망연결.md
    ├─ 05_GRE-Over-IPsec_Static.md
    ├─ 06_Dynamic-Routing_EIGRP.md
    ├─ 07_GRE-Over-IPsec_Dynamic.md
    ├─ 08_Transport-Mode_최적화.md
    ├─ 09_보안강화_AES256-GCM.md
    └─ images/
```

---

## 목차 (Chapters)

### Part 1 — GRE Tunnel 기본

| #  | 챕터 | 내용 |
| --- | --- | --- |
| 00 | [Topology & Base-Config](./00_Topology_and_Base-Config.md) | 토폴로지, 장비 역할, R1~R8 기본 설정 |
| 01 | [공중망 구성 (OSPF)](./01_공중망구성_OSPF.md) | ISP 구간 OSPF, passive-interface, point-to-point |
| 02 | [Default-Route](./02_Default-Route.md) | GIT-A/B → ISP 정적 기본 경로 |
| 03 | [GRE Tunnel](./03_GRE-Tunnel.md) | Tunnel source/destination, 터널 IP 할당 |
| 04 | [Static Route 사설망 연결](./04_Static-Route_사설망연결.md) | 본사-지사 사설망 정적 경로 |

### Part 2 — GRE-Over-IPsec (Static)

| #  | 챕터 | 내용 |
| --- | --- | --- |
| 05 | [GRE-Over-IPsec (Static)](./05_GRE-Over-IPsec_Static.md) | Tunnel/물리 인터페이스 crypto map, Tunnel & Transport Mode |

### Part 3 — GRE-Over-IPsec (Dynamic)

| #  | 챕터 | 내용 |
| --- | --- | --- |
| 06 | [Dynamic Routing (EIGRP)](./06_Dynamic-Routing_EIGRP.md) | 터널 위 EIGRP 사설망 구성 |
| 07 | [GRE-Over-IPsec (Dynamic)](./07_GRE-Over-IPsec_Dynamic.md) | EIGRP(Multicast) 보호 이슈, GRE 기준 ACL |
| 08 | [Transport Mode 최적화](./08_Transport-Mode_최적화.md) | 중복 캡슐화 방지, 오버헤드/Fragmentation 감소 |
| 09 | [보안 강화 (AES-256-GCM)](./09_보안강화_AES256-GCM.md) | 현대 표준 알고리즘 업그레이드 (참고) |

---

## 핵심 개념 요약

- **GRE (Generic Routing Encapsulation)**: 원본 패킷에 새 IP Header를 추가하여 터널링하는 프로토콜. Multicast/라우팅 프로토콜 전달 가능.
- **IPsec**: 암호화·무결성·인증 제공. **Unicast 트래픽만** 보호 가능 → EIGRP/OSPF(Multicast)는 직접 보호 불가.
- **GRE-Over-IPsec**: GRE로 터널링 → IPsec으로 GRE 패킷 보호. 동적 라우팅 프로토콜까지 안전하게 전달.
- **Crypto Map 적용 위치**
  - **Tunnel 인터페이스 적용**: 사설망 ACL(`permit ip`) 기준 보호
  - **물리 인터페이스 적용**: GRE Header 기준(`permit gre`) 보호 → EIGRP까지 포함
- **Tunnel Mode vs Transport Mode**
  - GRE-Over-IPsec에서는 **Transport Mode 권장** (외부 IP Header 중복 생성 방지)

---

## Phase 1 / Phase 2 정리

| 구분 | Phase 1 (ISAKMP/IKE) | Phase 2 (IPsec) |
| --- | --- | --- |
| 목적 | 관리 터널 협상 | 실제 데이터 보호 |
| 인증 | pre-share / RSA / 인증서 | - |
| 암호화 | DES / 3DES / AES | ESP (3DES / AES / AES-GCM) |
| 무결성 | MD5 / SHA | MD5-HMAC / SHA-HMAC |
| 키 교환 | Diffie-Hellman (1/2/5/14) | - |

---

## 학습 환경

|  도구  |  용도  |
| --- | --- |
| **GNS3** | 토폴로지 구성 |
| **Dynamips** | Cisco IOS 에뮬레이션 |
| **Cisco IOS (c7200)** | Router / FR-Switch |
| **Wireshark** | 패킷 캡처 / 암호화 검증 |
| **Git / GitHub** | 학습 정리 |

---

## License

[MIT License](./LICENSE)
