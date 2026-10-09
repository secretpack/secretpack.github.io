---
title: "TEE란 무엇인가: 신뢰 실행 환경과 ARM TrustZone"
excerpt: "TEE가 무엇이고 왜 필요한지, 그리고 ARM TrustZone 위에서 Normal World와 Secure World가 어떻게 분리되고 상호작용하는지 정리한다."
date: 2026-10-09
categories:
  - Study
tags:
  - TEE
  - TrustZone
  - ARM
  - security
toc: true
toc_label: "목차"
toc_sticky: true
---

우리가 사용하는 스마트폰, 자동차, IoT 디바이스에는 다양한 민감 정보가 저장되고 처리된다. 사용자의 인증 정보부터 암호화 키, 생체인증 데이터, 디바이스의 고유 식별 정보까지 그 종류도 다양하다.

일반적으로 이러한 정보는 운영체제가 제공하는 접근 제어와 메모리 보호 기능을 통해 보호된다. 그러나 운영체제의 커널이 취약점으로 인해 공격자에게 장악된다면 어떻게 될까?

공격자가 커널 수준의 권한을 획득하면 일반적인 프로세스 격리나 파일 접근 제어만으로는 중요한 정보를 보호하기 어려워진다. 운영체제 자체가 신뢰할 수 없는 상태가 되었기 때문이다.

이러한 문제를 해결하기 위한 접근 방식 중 하나가 **TEE(Trusted Execution Environment, 신뢰 실행 환경)** 이다.

TEE는 일반 운영체제와 분리된 실행 환경을 제공하여, 민감한 코드와 데이터를 보다 강력한 보안 경계 내에서 처리할 수 있도록 설계된 기술이다. 특히 ARM 아키텍처에서는 TrustZone이라는 하드웨어 보안 기술을 활용하여 일반 실행 환경(Normal World)과 보안 실행 환경(Secure World)을 분리할 수 있다.

이번 글에서는 TEE가 등장하게 된 배경과 기본 개념을 살펴보고, ARM TrustZone의 동작 원리를 중심으로 일반 운영체제와 보안 실행 환경이 어떻게 분리되고 상호작용하는지 알아본다. 이러한 TrustZone 기반 TEE를 실제로 구현한 오픈소스 프로젝트인 **OP-TEE**는 다음 글에서 이어서 다룬다.

## What is TEE?

### 1. TEE 개요
TEE(Trusted Execution Environment) 는 민감한 코드와 데이터를 보호하기 위해 일반 운영체제와 격리된 실행 환경을 제공하는 보안 기술이다.  

일반적으로 Linux, Android와 같은 운영체제는 프로세스별 가상 메모리 공간과 접근 제어 메커니즘을 통해 시스템 자원을 보호한다. 그러나 이러한 보호 메커니즘은 운영체제 커널의 신뢰성을 전제로 한다. 만약 공격자가 커널 취약점을 악용하여 높은 권한을 획득한다면, 운영체제가 제공하는 보안 경계를 우회할 가능성이 있다.  

TEE는 이러한 위협에 대응하기 위해 일반 운영체제와 독립적인 보안 경계를 형성한다. TEE를 지원하는 시스템에서는 일반 운영체제가 실행되는 환경을 REE(Rich Execution Environment), 보안에 민감한 작업을 수행하는 격리된 환경을 TEE(Trusted Execution Environment) 라고 구분한다.

### 2. REE와 TEE의 관계

REE와 TEE는 서로 격리되어 있지만 완전히 독립적으로 동작하는 것은 아니다.

예를 들어 일반 애플리케이션에서 암호화 연산이 필요한 경우, 애플리케이션은 TEE에서 제공하는 인터페이스를 통해 보안 기능을 요청할 수 있다.

이때 일반 실행 환경의 애플리케이션을 CA(Client Application), TEE 내부에서 실행되는 신뢰 애플리케이션을 TA(Trusted Application) 라고 부른다.

CA는 TEE에 요청을 전달하고, TA는 격리된 환경에서 해당 요청을 처리한 뒤 결과를 반환한다.

이러한 구조에서는 민감한 암호화 키를 REE에 직접 노출하지 않고 TEE 내부에서 연산을 수행하도록 설계할 수 있다. 다만 입력 데이터와 반환 결과의 보호 범위는 실제 구현과 API 설계에 따라 달라진다.

![REE와 TEE의 관계 — Normal World의 CA와 Secure World의 TA가 SMC를 통해 통신하며, 둘 다 ARM TrustZone이 적용된 하드웨어 위에서 동작한다](/assets/images/ree-tee.svg){: .align-center .diagram}
  
### 3. TEE 주요 보안 특성
TEE가 제공하고자 하는 대표적인 보안 특성은 다음과 같다.

* Isolation(격리): 일반 운영체제와 보안 실행 환경을 분리하여, REE에서 실행되는 소프트웨어가 TEE의 보호 자원에 임의로 접근하지 못하도록 제한한다.

* Confidentiality(기밀성): 암호화 키와 같은 민감한 데이터가 비인가 주체에게 노출되지 않도록 보호한다.

* Integrity(무결성): TEE 내부에서 실행되는 코드와 데이터가 비인가 주체에 의해 임의로 변경되지 않도록 보호한다.

* Trusted Execution(신뢰 실행): 신뢰된 코드가 보호된 환경에서 실행될 수 있도록 하며, 플랫폼에 따라 Secure Boot 및 하드웨어 신뢰 루트와 연계된다.

다만 TEE가 이러한 보안 특성을 제공한다고 해서 모든 공격으로부터 안전하다는 의미는 아니다. TEE 내부 소프트웨어의 취약점이나 잘못된 인터페이스 설계, 하드웨어 구현상의 결함 등은 여전히 보안 위협이 될 수 있다.

### 4. ARM Trustzone과 TEE

TEE를 구현하기 위한 대표적인 하드웨어 기술 중 하나가 ARM TrustZone이다.

**ARM TrustZone**은 시스템 자원을 보안 상태에 따라 분리할 수 있는 하드웨어 메커니즘을 제공한다. 이를 통해 일반적으로 **Normal World**와 **Secure World**라고 불리는 두 실행 영역을 구성할 수 있다.

* Normal World: Linux, Android와 같은 일반 운영체제와 사용자 애플리케이션이 실행되는 영역

* Secure World: OP-TEE와 같은 보안 운영체제 및 Trusted Application이 실행될 수 있는 영역

여기서 중요한 점은 ARM TrustZone 자체가 TEE 운영체제는 아니라는 것이다.

TrustZone은 보안 상태 분리와 접근 제어를 지원하는 하드웨어 기반을 제공하며, OP-TEE는 이러한 기능을 활용하여 실제 신뢰 실행 환경을 구현하는 소프트웨어이다.

즉, TrustZone이 격리를 위한 하드웨어 기반을 제공한다면, OP-TEE는 그 위에서 동작하는 보안 운영체제와 실행 환경을 제공한다고 이해할 수 있다.


![ARM TrustZone이 NS 비트로 CPU 코어 상태, 메모리(TZASC), 주변장치(TZPC)를 Non-Secure(Normal World)와 Secure(Secure World)로 분리하는 하드웨어 메커니즘](/assets/images/trustzone-hw.svg){: .align-center .diagram}


### 5. 표준: GlobalPlatform

TEE는 특정 하드웨어 제조사나 소프트웨어 벤더에 종속된 기술이 아니다. TEE의 아키텍처와 인터페이스에 대한 산업 표준이 존재하며, 대표적으로 **GlobalPlatform**에서 관련 규격을 정의하고 있다.

GlobalPlatform은 TEE의 보안 모델과 소프트웨어 구조를 정의하고, 서로 다른 TEE 구현에서도 일관된 방식으로 애플리케이션을 개발하고 사용할 수 있도록 표준 API를 제공한다.

대표적인 규격은 다음과 같다.

* TEE System Architecture: TEE의 전반적인 아키텍처와 보안 모델, 구성 요소 및 REE와의 상호작용 방식을 정의한다.

* TEE Internal Core API: TEE 내부에서 실행되는 Trusted Application(TA)이 사용하는 API를 정의한다. 메모리 관리, 암호화 연산, 보안 저장소 및 세션 관리 등의 기능을 포함한다.

* TEE Client API: 일반 실행 환경(REE)의 Client Application(CA)이 TEE와 통신하기 위한 인터페이스를 정의한다. 세션 생성, 명령 호출 및 데이터 전달 등의 기능을 제공한다.

이러한 표준은 TEE 구현체마다 발생할 수 있는 인터페이스 차이를 줄이고, 애플리케이션의 이식성을 높이는 데 목적이 있다. 다만 실제 이식 가능 여부는 각 구현체의 지원 기능과 확장 API 등에 따라 달라질 수 있다.

이후 살펴볼 OP-TEE 역시 GlobalPlatform의 TEE Client API와 TEE Internal Core API를 구현한 대표적인 오픈소스 TEE 프로젝트다. 따라서 OP-TEE의 동작 구조를 이해하는 것은 GlobalPlatform에서 정의한 TEE 아키텍처가 실제 소프트웨어에서 어떻게 구현되는지 살펴보는 과정이기도 하다.

## TEE Architecture

앞 장에서 ARM TrustZone이 Normal World와 Secure World를 분리하는 하드웨어 기반을 제공한다고 설명하였다. 이 장에서는 두 실행 환경이 하드웨어 수준에서 구체적으로 어떻게 분리되고, 어떤 경로로 상호작용하는지 살펴본다.

### 1. NS 비트와 두 실행 환경

TrustZone의 핵심은 프로세서에 추가된 **NS 비트(Non-Secure bit)** 이다. 이 단일 상태 비트가 시스템 전체를 Secure World와 Normal World로 구분한다.

- NS = 0 → **Secure World** (TEE가 동작하는 영역)
- NS = 1 → **Normal World** (REE가 동작하는 영역)

이 구분은 CPU 코어에만 적용되는 것이 아니다. NS 비트는 버스 트랜잭션에 실려 메모리 컨트롤러, 캐시, 주변장치까지 전파된다. 따라서 하드웨어 전역이 각 접근의 보안 상태를 인지하며, Non-Secure 접근이 Secure 자원에 접근하려 시도하면 하드웨어 수준에서 차단된다.

![TrustZone SoC 내 Normal World와 Secure World, 그리고 두 영역을 잇는 Secure Monitor(EL3)](/assets/images/tee-two-worlds.svg){: .align-center .diagram}

### 2. Exception Level과 특권 분리

ARMv8-A(AArch64)에서는 특권 수준을 **Exception Level(EL0~EL3)** 로 구분한다. 이를 TrustZone의 두 실행 환경과 결합하면 각 소프트웨어가 위치하는 자리가 명확해진다.

| Exception Level | Normal World | Secure World |
| :---: | :---: | :---: |
| EL0 (최저 특권) | 일반 사용자 앱 | **Trusted Application (S-EL0)** |
| EL1 | Linux/Android 커널 | **Trusted OS (S-EL1)** |
| EL2 | 하이퍼바이저 | (보안 하이퍼바이저 / SPM) |
| EL3 (최고 특권) | — | **Secure Monitor** |

- **EL3의 Secure Monitor**는 두 실행 환경 사이를 중재하는 유일한 관문이다.
- **Trusted OS는 S-EL1**, 그 위에서 동작하는 **Trusted Application(TA)은 S-EL0**에 위치한다.
- Normal World의 운영체제 커널과 Trusted OS는 동일한 EL1 수준이지만, NS 비트로 완전히 분리된 서로 다른 실행 환경이다.

### 3. 월드 스위칭(World Switch): SMC와 Secure Monitor

두 실행 환경은 서로를 임의로 호출할 수 없다. 모든 전환은 반드시 EL3의 Secure Monitor를 거치며, 그 전환을 유발하는 명령이 **SMC(Secure Monitor Call)** 이다.

요청과 응답의 흐름을 단순화하면 다음과 같다.

![월드 스위칭 흐름 — Normal World 커널이 SMC로 Secure Monitor(EL3)를 거쳐 Trusted OS(S-EL1)와 Trusted App(S-EL0)으로 진입하고 결과를 들고 복귀한다](/assets/images/tee-world-switch.svg){: .align-center .diagram}

Secure Monitor는 전환 시점에 레지스터와 실행 상태를 저장·복원하여, 한 실행 환경의 컨텍스트가 다른 실행 환경으로 유출되지 않도록 한다. 이러한 월드 스위칭(world switching)은 TEE의 성능과 보안을 좌우하는 핵심 경로이다.

### 4. 메모리와 주변장치 격리

CPU의 보안 상태를 구분하는 것만으로는 충분하지 않다. 메모리와 주변장치 역시 함께 격리되어야 한다. ARM 플랫폼은 일반적으로 다음과 같은 하드웨어를 통해 이를 강제한다.

- **TZASC (TrustZone Address Space Controller)** — DRAM 영역을 Secure/Non-Secure로 분할하고, Non-Secure 접근이 Secure 영역에 닿지 못하게 막는다.
- **TZPC (TrustZone Protection Controller)** — 주변장치(peripheral)를 어느 실행 환경에 할당할지 제어한다.
- **캐시/MMU의 NS 태깅** — 캐시 라인과 TLB 엔트리에도 Secure 여부가 태깅되어, 실행 환경 간에 캐시가 혼선되지 않는다.

이러한 메커니즘을 통해 Secure 전용 메모리 영역이 물리적으로 구분되며, Normal World가 해당 영역에 접근을 시도하더라도 하드웨어 수준에서 거부된다.

### 5. 신뢰 체인: Secure Boot

TEE를 신뢰할 수 있으려면 변조되지 않은 TEE가 로드되었다는 사실부터 보장되어야 한다. 이를 위해 부팅 단계부터 서명 검증이 사슬처럼 이어진다. ARM의 레퍼런스 구현인 **TF-A(Trusted Firmware-A)** 의 부트 스테이지를 기준으로 보면 다음과 같다.

| 스테이지 | 역할 | 실행 환경/레벨 |
| :---: | :---: | :---: |
| BL1 | ROM 부트코드 (신뢰의 뿌리) | Secure |
| BL2 | 다음 이미지 로드·검증 | Secure |
| BL31 | **Secure Monitor** (런타임 상주) | EL3 |
| BL32 | **Trusted OS (여기가 OP-TEE 자리)** | S-EL1 |
| BL33 | Normal World 부트로더(U-Boot 등) → 일반 OS | Non-Secure |

각 단계는 다음 단계의 서명을 검증한 뒤에만 제어를 넘긴다. 이 사슬이 어느 지점에서도 끊기지 않아야 TEE 전체의 신뢰가 성립한다.

## 마치며

이번 글에서는 TEE의 개념과 등장 배경, 그리고 ARM TrustZone이 Normal World와 Secure World를 어떻게 분리하고 중재하는지 살펴보았다. 요약하면 다음과 같다.

- **TEE**는 일반 운영체제와 격리된 신뢰 실행 환경으로, 격리·기밀성·무결성을 제공한다.
- **REE와 TEE**는 CA와 TA를 통해 상호작용하며, GlobalPlatform이 그 표준 API를 정의한다.
- **TrustZone**은 NS 비트를 통해 CPU·메모리·주변장치를 두 실행 환경으로 분리하고, EL3의 Secure Monitor가 월드 스위칭을 중재한다.

다음 글에서는 이러한 TrustZone 기반 위에서 동작하는 대표적인 오픈소스 Trusted OS인 **OP-TEE**의 구조와 호출 흐름을 살펴본다.
