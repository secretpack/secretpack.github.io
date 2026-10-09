---
title: "TEE와 OP-TEE: 신뢰 실행 환경의 구조 이해하기"
excerpt: "TEE가 무엇이고, ARM TrustZone 위에서 어떤 구조로 동작하며, 그 위에서 OP-TEE가 어떻게 구현되는지 정리한다."
date: 2026-10-09
categories:
  - Study
tags:
  - TEE
  - TrustZone
  - OP-TEE
  - ARM
  - security
toc: true
toc_label: "목차"
toc_sticky: true
---

그동안 공부했던 **TEE(Trusted Execution Environment)** 와 그 대표 구현체인
**OP-TEE** 를 정리한다. "TEE가 대체 무엇인가" 에서 시작해, 그것을 떠받치는
하드웨어 구조(ARM TrustZone)를 거쳐, 실제 오픈소스 구현인 OP-TEE까지
한 흐름으로 따라간다.

## What is TEE?

### 한 줄 정의

> TEE는 메인 프로세서 안에 존재하는, 일반 운영체제로부터 **격리된 실행 환경**이다.
> 이 안에서 돌아가는 코드와 데이터는 바깥 세계(일반 OS)가 침해하거나 훔쳐볼 수
> 없도록 **기밀성(confidentiality)** 과 **무결성(integrity)** 을 보장받는다.

스마트폰을 생각하면 이해가 빠르다. Android나 iOS 같은 일반 OS는 수백만 줄의
코드와 수많은 서드파티 앱, 드라이버로 이루어져 있다. 공격 표면이 거대하고,
한 번 루팅되거나 커널이 뚫리면 그 위의 모든 비밀이 노출된다. 그런데 지문
템플릿, 결제 키, DRM 라이선스 키 같은 자산은 "OS가 뚫려도" 지켜져야 한다.

그래서 등장한 것이 TEE다. 민감한 연산만 떼어내어, 일반 OS가 건드릴 수 없는
별도의 보호 구역에서 처리한다.

### REE vs TEE

TEE를 이야기할 때는 항상 그 반대편인 **REE(Rich Execution Environment)** 가
함께 등장한다. 둘은 같은 칩 위에서 공존한다.

| 구분 | REE (Normal World) | TEE (Secure World) |
| :---: | :---: | :---: |
| 실행 주체 | Android, Linux 등 일반 OS | Trusted OS (소형 보안 커널) |
| 코드 규모 | 수백만~수천만 줄 | 수만~수십만 줄 |
| 공격 표면 | 매우 큼 | 의도적으로 최소화 |
| 신뢰 수준 | Untrusted | Trusted |
| 담당 작업 | 일반 앱, UI, 네트워크 | 키 관리, 암호 연산, 지문 매칭 등 |

핵심은 **신뢰 경계(trust boundary)** 다. REE는 기본적으로 신뢰하지 않는
영역이고, TEE는 하드웨어가 보장하는 신뢰 영역이다. REE가 완전히 장악당한
상황을 가정해도 TEE 안의 비밀은 지켜지는 것이 설계 목표다.

### 무엇을 보장하는가

TEE가 제공하는 보안 속성은 대략 다음과 같다.

- **격리된 실행(Isolated Execution)** — TEE 내부 코드는 REE가 읽거나 변조할 수 없는 메모리에서 실행된다.
- **보안 저장(Secure Storage)** — 디바이스에 묶인 키로 암호화된 데이터를 안전하게 보관한다.
- **무결성 보장 부팅** — 신뢰 체인(chain of trust)을 통해 TEE 자체가 변조되지 않았음을 검증한다.
- **원격 증명(Attestation)** — "이 기기가 정품 TEE에서 특정 코드를 돌리고 있다"를 외부에 증명한다.
- **신뢰 입출력(Trusted I/O)** — (구현에 따라) 화면/지문 센서 같은 주변장치를 REE 몰래 직접 다룬다.

### 표준: GlobalPlatform

TEE는 특정 벤더의 발명품이 아니라 산업 표준이 존재한다.
**GlobalPlatform** 이 TEE의 레퍼런스 아키텍처와 API를 정의한다. 대표적으로:

- **TEE System Architecture** — 전체 구조와 보안 요구사항 규격
- **TEE Internal Core API** — TEE 내부에서 도는 앱(Trusted Application)이 쓰는 API
- **TEE Client API** — 바깥(REE)에서 TEE를 호출할 때 쓰는 API

이 표준 덕분에 서로 다른 TEE 구현 위에서도 비슷한 방식으로 보안 앱을 작성할
수 있다. 뒤에서 볼 OP-TEE도 이 GlobalPlatform API를 충실히 구현한다.

## TEE Architecture

그렇다면 "같은 칩 위에서 격리된 두 세계"는 실제로 어떻게 만들어질까. ARM
진영에서는 **TrustZone** 이라는 하드웨어 기능이 그 토대다.

### ARM TrustZone: 두 개의 세계

TrustZone의 발상은 단순하다. CPU에 **NS 비트(Non-Secure bit)** 라는 상태를
하나 둔다. 이 비트 하나가 시스템 전체를 **Secure World** 와
**Non-Secure World** 로 가른다.

- NS = 0 → **Secure World** (TEE가 사는 곳)
- NS = 1 → **Non-Secure World** (REE가 사는 곳)

중요한 것은 이 구분이 CPU 코어에만 머물지 않는다는 점이다. NS 비트는 버스
트랜잭션에 실려 메모리 컨트롤러, 캐시, 주변장치까지 전파된다. 즉 "지금 이
접근이 Secure인지 Non-Secure인지"를 하드웨어 전역이 알고 있고, Non-Secure
접근이 Secure 자원을 건드리려 하면 하드웨어가 차단한다.

![TrustZone — 하나의 SoC 위의 Normal World와 Secure World, 그리고 두 세계를 잇는 Secure Monitor(EL3)](/assets/images/tee-two-worlds.svg){: .align-center .diagram}

### Exception Level: 누가 어느 특권에서 도는가

ARMv8-A(AArch64)에서는 특권 수준을 **Exception Level(EL0~EL3)** 로 나눈다.
TrustZone의 두 세계와 결합하면 각 소프트웨어의 자리가 분명해진다.

| Exception Level | Normal World | Secure World |
| :---: | :---: | :---: |
| EL0 (최저 특권) | 일반 사용자 앱 | **Trusted Application (S-EL0)** |
| EL1 | Linux/Android 커널 | **Trusted OS (S-EL1)** |
| EL2 | 하이퍼바이저 | (보안 하이퍼바이저 / SPM) |
| EL3 (최고 특권) | — | **Secure Monitor** |

- **EL3의 Secure Monitor** 가 두 세계 사이를 중재하는 유일한 관문이다.
- **Trusted OS는 S-EL1**, 그 위에서 도는 **보안 앱(TA)은 S-EL0** 에 위치한다.
- 일반 OS 커널과 Trusted OS는 같은 EL1 "레벨"이지만 NS 비트로 완전히 분리된 다른 세계다.

### 세계 전환: SMC와 Secure Monitor

두 세계는 마음대로 서로를 호출할 수 없다. 전환은 반드시 EL3의 Secure
Monitor를 거친다. 그 트리거가 **SMC(Secure Monitor Call)** 명령이다.

흐름을 단순화하면 이렇다.

![세계 전환 흐름 — Normal World 커널이 SMC로 Secure Monitor(EL3)를 거쳐 Trusted OS(S-EL1)와 Trusted App(S-EL0)으로 진입하고 결과를 들고 복귀한다](/assets/images/tee-world-switch.svg){: .align-center .diagram}

Secure Monitor는 전환 시점에 레지스터와 상태를 저장·복원해서, 한 세계의
컨텍스트가 다른 세계로 새어 나가지 않게 한다. 이 "월드 스위칭"이 TEE
성능과 보안의 핵심 길목이다.

### 메모리와 주변장치 격리

CPU 상태만 나눠서는 부족하다. 메모리와 장치도 함께 격리되어야 한다. ARM
플랫폼은 보통 다음과 같은 하드웨어로 이를 강제한다.

- **TZASC (TrustZone Address Space Controller)** — DRAM 영역을 Secure/Non-Secure로 분할하고, Non-Secure 접근이 Secure 영역에 닿지 못하게 막는다.
- **TZPC (TrustZone Protection Controller)** — 주변장치(peripheral)를 어느 세계에 할당할지 제어한다.
- **캐시/MMU의 NS 태깅** — 캐시 라인과 TLB 엔트리에도 Secure 여부가 태깅되어, 세계 간에 캐시가 혼선되지 않는다.

덕분에 "Secure 전용 RAM 영역"이 물리적으로 존재하고, Normal World는 그
주소에 접근해도 하드웨어 단에서 거부된다.

### 신뢰 체인: 보안 부팅

TEE가 믿을 만하려면 "변조되지 않은 TEE가 로드됐다"는 것부터 보장돼야 한다.
그래서 부팅 단계부터 서명 검증이 사슬처럼 이어진다. ARM 레퍼런스 구현인
**TF-A(Trusted Firmware-A)** 의 부트 스테이지로 보면:

| 스테이지 | 역할 | 세계/레벨 |
| :---: | :---: | :---: |
| BL1 | ROM 부트코드 (신뢰의 뿌리) | Secure |
| BL2 | 다음 이미지 로드·검증 | Secure |
| BL31 | **Secure Monitor** (런타임 상주) | EL3 |
| BL32 | **Trusted OS (여기가 OP-TEE 자리)** | S-EL1 |
| BL33 | Normal World 부트로더(U-Boot 등) → 일반 OS | Non-Secure |

각 단계는 다음 단계의 서명을 검증한 뒤에만 넘긴다. 이 사슬이 끊기지 않아야
TEE 전체의 신뢰가 성립한다.

## OP-TEE

여기까지가 "하드웨어가 깔아준 무대"였다면, 그 무대 위에서 실제로 돌아가는
Trusted OS의 대표적 오픈소스 구현이 **OP-TEE(Open Portable TEE)** 다.

### OP-TEE란

- 원래 ST-Ericsson이 개발했고, 이후 Linaro를 거쳐 현재는 **TrustedFirmware.org** 프로젝트로 오픈소스로 유지된다.
- 앞서 본 **GlobalPlatform의 TEE Internal Core API / Client API 를 구현** 한다.
- 위 부트 체인에서 **BL32(= S-EL1의 Trusted OS)** 자리에 올라간다.
- 실제 하드웨어뿐 아니라 **QEMU** 로도 쉽게 돌려볼 수 있어, TEE를 공부하고 실험하기에 가장 접근성이 좋다.

### 구성 요소

OP-TEE는 하나의 바이너리가 아니라, 두 세계에 걸친 여러 컴포넌트의 집합이다.

**Secure World 쪽**

- **OP-TEE OS (optee_os)** — S-EL1에서 도는 보안 커널. 스케줄링, 메모리 관리, 암호 연산, TA 로딩을 담당한다.
- **Trusted Application (TA)** — S-EL0에서 도는 보안 앱. 각 TA는 **UUID** 로 식별되며 서명되어 로드된다.

**Normal World 쪽**

- **OP-TEE 리눅스 커널 드라이버** — `/dev/tee0`, `/dev/teepriv0` 디바이스를 노출하고, SMC로 Secure World와 통신한다.
- **libteec (TEE Client API 라이브러리)** — 일반 앱(CA)이 TEE를 호출할 때 쓰는 사용자 공간 라이브러리.
- **tee-supplicant** — Secure World가 Normal World의 자원(파일 시스템 기반 보안 저장, RPC 등)을 필요로 할 때 이를 대신 처리해 주는 데몬.

![OP-TEE 구성 요소 — Normal World의 CA·libteec·커널 드라이버·tee-supplicant와 Secure World의 Trusted App·OP-TEE OS가 SMC(Secure Monitor)와 RPC로 연결된다](/assets/images/optee-components.svg){: .align-center .diagram}

### 호출 흐름 (CA → TA)

일반 앱이 TEE 안의 보안 앱을 호출하는 전형적인 순서는 다음과 같다.

1. **CA가 Client API로 세션을 연다** — `TEEC_InitializeContext` → `TEEC_OpenSession(UUID)`.
2. **커널 드라이버가 SMC를 발생** — 요청이 Secure Monitor(EL3)를 거쳐 Secure World로 넘어간다.
3. **OP-TEE OS가 대상 TA를 로드·실행** — UUID로 지정된 TA를 S-EL0에서 구동한다.
4. **파라미터는 공유 메모리로 전달** — 두 세계가 합의한 shared memory 영역을 통해 버퍼를 주고받는다.
5. **TA가 연산 수행 후 결과 반환** — 다시 Monitor를 경유해 Normal World로 복귀한다.

여기서 **공유 메모리 경계와 파라미터 검증**이 보안상 가장 민감한 지점이다.
Normal World가 넘긴 포인터·크기 값을 Secure World가 그대로 믿으면, 바로
그곳이 취약점의 온상이 된다. (이 경계면이 TEE 보안 연구에서 핵심 공격
표면이 된다.)

### 어디에 쓰이나

OP-TEE를 포함한 TEE는 모바일/임베디드 전반에서 이미 광범위하게 쓰인다.

- **키 저장소** — Android Keystore/Keymint의 하드웨어 백엔드
- **생체 인증** — 지문/얼굴 템플릿의 저장·매칭을 TEE 안에서 수행
- **모바일 결제·DRM** — 결제 토큰, 콘텐츠 보호 키 관리
- **디스크/파일 암호화 키 보호**, **원격 증명** 등

### 왜 공부할 가치가 있나

TEE는 "OS가 뚫려도 지켜지는 최후의 방어선"을 표방한다. 그만큼 그 방어선
자체의 견고함이 중요하고, 세계 경계(SMC, 공유 메모리, TA 로더, RPC)의
작은 실수 하나가 전체 신뢰 모델을 무너뜨릴 수 있다. 구조를 제대로 이해해야
그 경계에서 무엇이 잘못될 수 있는지 보이기 시작한다.

## 마치며

이번 글에서는 TEE의 개념, TrustZone 기반의 아키텍처, 그리고 그 위의 구현체인
OP-TEE까지 큰 그림을 따라갔다. 요약하면:

- **TEE** 는 일반 OS와 격리된 신뢰 실행 환경으로, 기밀성·무결성을 보장한다.
- **TrustZone** 은 NS 비트로 하드웨어 전역을 두 세계로 나누고, EL3의 Secure Monitor가 그 경계를 지킨다.
- **OP-TEE** 는 그 Secure World에 올라가는 오픈소스 Trusted OS로, GlobalPlatform API를 구현하며 두 세계에 걸친 컴포넌트들로 동작한다.

다음 글에서는 QEMU 위에 OP-TEE를 직접 올려 보고, CA와 TA를 간단히 작성해
호출 흐름을 눈으로 확인하는 과정을 정리할 예정이다.
