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

아래는 추후 다룰 OP-TEE를 예시로 한 REE와 TEE의 관계를 도식화 한 그림이다. (본 이미지는 AI 를 통해 생성되었음)

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

ARM TrustZone 자체가 TEE 운영체제는 아니다. TrustZone은 보안 상태 분리와 접근 제어를 지원하는 하드웨어 기반을 제공하며, TEE는 이러한 기능을 활용하여 실제 신뢰 실행 환경을 구현하는 소프트웨어이다.

즉, TrustZone이 격리를 위한 하드웨어 기반을 제공한다면, TEE는 그 위에서 동작하는 보안 운영체제와 실행 환경을 제공한다고 이해할 수 있다.


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

위에서 ARM TrustZone이 Normal World와 Secure World를 분리하는 하드웨어 기반을 제공한다고 설명하였다. 이 장에서는 두 실행 환경이 하드웨어 수준에서 구체적으로 어떻게 분리되고, 어떤 경로로 상호작용하는지 살펴본다.

### 1. NS 비트와 두 실행 환경

TrustZone의 핵심은 프로세서에 추가된 **NS 비트(Non-Secure bit)** 이다. 이 단일 상태 비트가 시스템 전체를 Secure World와 Normal World로 구분한다.

- NS = 0 → **Secure World** (TEE가 동작하는 영역)
- NS = 1 → **Normal World** (REE가 동작하는 영역)

이 구분은 CPU 코어에만 적용되는 것이 아니다. NS 비트는 버스 트랜잭션에 실려 메모리 컨트롤러, 캐시, 주변장치까지 전파된다. 따라서 하드웨어 전역이 각 접근의 보안 상태를 인지하며, Non-Secure 접근이 Secure 자원에 접근하려 시도하면 하드웨어 수준에서 차단된다.

![TrustZone SoC 내 Normal World와 Secure World, 그리고 두 영역을 잇는 Secure Monitor(EL3)](/assets/images/tee-two-worlds.svg){: .align-center .diagram}

### 2. Exception Level과 특권 분리 (Isolated Execution)

ARMv8-A(AArch64)에서는 특권 수준을 **Exception Level(EL0~EL3)** 로 구분한다. 이를 TrustZone의 두 실행 환경과 결합하면 각 소프트웨어가 위치하는 자리가 명확해진다.
<div align="center" markdown="1">

| Exception Level | Normal World | Secure World |
| :---: | :---: | :---: |
| EL0 (최저 특권) | 일반 사용자 앱 | **Trusted Application (S-EL0)** |
| EL1 | Linux/Android 커널 | **Trusted OS (S-EL1)** |
| EL2 | 하이퍼바이저 | (보안 하이퍼바이저 / SPM) |
| EL3 (최고 특권) | — | **Secure Monitor** |

</div>

Trusted OS는 **S-EL1**, 그 위에서 동작하는 **Trusted Application(TA)은 S-EL0**에 위치한다. Normal World의 운영체제 커널과 Trusted OS는 동일한 EL1 수준이지만, NS 비트로 완전히 분리된 서로 다른 실행 환경이다.

EL3의 Secure Monitor 는 두 실행 영역 사이를 중재하는 유일한 관문이다. 

### 3. 영역 전환(World Switching): SMC와 Secure Monitor

두 영역은 서로를 임의로 호출할 수 없다. 모든 전환은 반드시 EL3의 Secure Monitor를 거치며, 그 전환을 유발하는 명령이 **SMC(Secure Monitor Call)** 이다.

요청과 응답의 흐름을 단순화하면 다음과 같다.

![영역 전환 흐름 — Normal World 커널이 SMC로 Secure Monitor(EL3)를 거쳐 Trusted OS(S-EL1)와 Trusted App(S-EL0)으로 진입하고 결과를 들고 복귀한다](/assets/images/tee-world-switch.svg){: .align-center .diagram}

Secure Monitor는 전환 시점에 레지스터와 실행 상태를 저장·복원하여, 한 영역의 컨텍스트가 다른 영역으로 유출되지 않도록 한다. 이러한 영역 전환(world switching)은 TEE의 성능과 보안을 좌우하는 핵심 경로이다.

### 4. 메모리와 주변장치 격리

CPU의 보안 상태를 구분하는 것만으로는 충분하지 않다. 메모리와 주변장치 역시 함께 격리되어야 한다. ARM 플랫폼은 일반적으로 다음과 같은 하드웨어를 통해 이를 강제한다.

- **TZASC (TrustZone Address Space Controller)** : DRAM 영역을 Secure/Non-Secure로 분할하고, Non-Secure 접근이 Secure 영역에 닿지 못하게 막는다.
- **TZPC (TrustZone Protection Controller)** : 주변장치(peripheral)를 어느 영역에 할당할지 제어한다.
- **캐시/MMU의 NS 태깅** : 캐시 라인과 TLB 엔트리에도 Secure 여부가 태깅되어, 영역 간에 캐시가 혼선되지 않는다.

이러한 메커니즘을 통해 Secure 전용 메모리 영역이 물리적으로 구분되며, Normal World가 해당 영역에 접근을 시도하더라도 하드웨어 수준에서 거부된다.

## TEE의 주요 기능
위 문단에서, TEE의 핵심 기능인 Isolated Execution에 대해 설명했다. 본 문단에서는 TEE의 또 다른 핵심 기능인 Secure Boot, Secure Storage, Secure Cryptographic Services에 대해 설명한다.

### 1. Secure Boot

**Secure Boot**는 디바이스의 부팅 과정에서 실행되는 소프트웨어의 무결성과 신뢰성을 검증하는 보안 기술이다. 일반적으로 디바이스는 ROM에 저장된 초기 부팅 코드에서 시작하여 부트로더(Bootloader)를 거쳐 운영체제(OS)를 실행한다.

이 과정에서 공격자가 부트로더나 OS 이미지를 변조할 수 있다면, 운영체제가 시작되기도 전에 악성 코드를 실행하거나 시스템의 보안 기능을 무력화할 수 있다.

Secure Boot는 이러한 위협을 방지하기 위해 **각 부팅 단계에서 다음에 실행할 소프트웨어의 디지털 서명을 검증하고, 검증에 성공한 경우에만 실행을 허용**한다.

이처럼 신뢰할 수 있는 초기 코드에서 시작하여 다음 단계로 신뢰를 확장하는 구조를 **Chain of Trust(신뢰 체인)** 라고 한다.

TEE(Trusted Execution Environment)는 일반 운영체제와 격리된 환경에서 중요한 데이터와 보안 기능을 보호한다. 그러나 TEE 자체가 공격자에 의해 변조된 상태로 실행된다면, 아무리 강력한 실행 환경의 격리 기능을 제공하더라도 그 안전성을 신뢰하기 어렵다.

따라서 **TEE의 보안은 Secure OS가 실행되는 시점이 아니라, Secure OS를 메모리에 로드하고 실행하기 이전의 부팅 단계에서부터 시작**된다.

이를 위해 TEE가 탑재된 시스템에서는 일반 운영체제뿐만 아니라 Secure Monitor, Secure OS 등의 보안 펌웨어 역시 신뢰 체인에 포함되어야 한다.

ARM의 레퍼런스 펌웨어 구현인 Trusted Firmware-A(TF-A) 를 예로 들어 살펴보자.


<div align="center" markdown="1">

| Stage | 주요 역할 | 실행 환경 |
| :---: | :---: | :---: |
| BL1 | 초기 부팅 및 신뢰 체인 시작 | S-EL3 |
| BL2 | 후속 부팅 이미지 로드 및 인증 | S-EL1 |
| BL31 | **Secure Monitor** 및 런타임 서비스 | EL3 |
| BL32 | **Trusted OS (e.g., OP-TEE)** | S-EL1 |
| BL33 | Normal World 부트로더(e.g., U-Boot) | Non-Secure EL2 / EL1 |

</div>

일반적인 TF-A Trusted Board Boot 구성에서는 변경 불가능한 ROM 코드 또는 하드웨어에 내장된 신뢰의 뿌리(Root of Trust)를 기반으로 초기 부팅 이미지의 신뢰성을 확보한다.

이후 BL2는 후속 부팅 이미지의 인증 정보를 확인하고, 검증된 BL31, BL32, BL33 이미지를 로드한다. BL31은 EL3에서 Secure Monitor 역할을 수행하며, Secure World와 Normal World 사이의 전환을 관리한다.

이때 BL32에 해당하는 Secure OS가 바로 추후 언급 할 OP-TEE가 실행되는 영역이다. 즉, OP-TEE 역시 부팅 과정에서 무결성과 신뢰성이 검증되어야 하는 대상이다.

다만 모든 부팅 스테이지가 반드시 바로 다음 스테이지를 직접 검증하는 것은 아니다. 실제 인증 및 로딩 순서는 플랫폼과 TF-A 구성에 따라 달라지며, 중요한 것은 신뢰할 수 있는 초기 코드에서 시작된 검증 관계가 최종 실행 이미지까지 유지된다는 점이다.

결국 Secure Boot는 단순히 운영체제의 변조를 방지하는 기능을 넘어, TEE를 구성하는 보안 펌웨어와 Secure OS가 신뢰할 수 있는 상태로 실행되도록 보장하는 기반 기술이라고 할 수 있다.  

아래는 Secure Boot와 TEE의 관계가 포함된 부팅 Structure를 표현한 그림이다. 본 그림은 생성형 AI를 활용하여 생성된 이미지임을 명시한다.  

![ARM Secure Boot 구조와 TEE의 관계](/assets/images/ARM_Secure_Boot_Structure_With_TEE.png){: .align-center .diagram}

### 2. Secure Storage
#### 2.1. 개요
**Secure Storage**는 암호화 키(Cryptographic Key), 인증 정보(Credential), 보안 설정값 등 민감한 데이터를 안전하게 저장하고 관리하기 위한 TEE의 핵심 보안 기능 중 하나이다.

일반적인 운영체제에서는 파일 시스템의 접근 권한이나 데이터 암호화를 통해 중요한 정보를 보호한다. 그러나 공격자가 운영체제의 관리자 권한을 획득하거나 커널을 장악한다면, 파일에 대한 접근 통제를 우회하거나 저장된 데이터를 탈취할 가능성이 있다.

TEE는 이러한 위협에 대응하기 위해 일반 운영체제와 격리된 환경에서 민감한 데이터를 관리하는 Secure Storage 기능을 제공한다.

여기서 중요한 점은 Secure Storage가 반드시 별도의 보안 전용 메모리나 저장장치를 의미하는 것은 아니라는 사실이다.

실제로 많은 TEE 구현에서는 일반 운영체제가 사용하는 eMMC, UFS 등의 비휘발성 저장장치에 암호화된 데이터를 저장한다. 다만 데이터의 암호화와 무결성 검증에 필요한 핵심 연산 및 키 관리는 Secure World에서 수행된다.

따라서 공격자가 Normal World의 파일 시스템에 접근하더라도, 암호화 키를 확보하지 못한다면 저장된 데이터의 평문을 확인하거나 유효한 데이터로 위조하기 어렵다.

#### 2.2 Secure Storage 동작 원리
Secure Storage의 핵심은 데이터를 어디에 저장하는가보다 데이터를 보호하는 암호화 키를 누가 관리하고, 저장된 데이터의 무결성을 어떻게 검증하는가에 있다.

일반적으로 TEE 기반 Secure Storage는 다음과 같은 과정을 통해 데이터를 보호한다.

![Secure Storage Architecture](/assets/images/Secure_Storage_Architecture.png){: .align-center .diagram}

1. 데이터 저장 요청: Trusted Application(TA)이 TEE의 보안 저장소 API를 통해 데이터 저장을 요청한다.

2. 데이터 암호화: Secure OS는 보호된 키 계층을 이용해 데이터를 암호화하고, 무결성 검증을 위한 인증 정보를 생성한다.

3. 비휘발성 저장: 암호화된 데이터는 일반 파일 시스템이나 보안 기능을 갖춘 저장 영역에 기록된다.

4. 데이터 복원 및 검증: 저장된 데이터를 읽을 때 Secure OS가 무결성을 확인하고 복호화하여 TA에 제공한다.

이러한 구조에서는 일반 운영체제가 실제 저장장치에 접근하더라도, Secure World에서 관리하는 암호화 키와 보안 정책까지 직접 제어할 수는 없다.

Secure Storage는 단순히 데이터를 암호화하여 저장하는 기능이 아니다.

TEE의 격리된 실행 환경과 하드웨어 기반 키 보호 기능을 활용하여, 일반 운영체제가 침해된 상황에서도 민감한 데이터의 기밀성과 무결성을 보호하기 위한 보안 메커니즘이다.

다만 Secure Storage 역시 모든 공격을 방어하는 것은 아니다. Secure OS나 Trusted Application 자체에 취약점이 존재한다면 데이터 보호 기능이 우회될 수 있으며, 저장 방식에 따라 Rollback Attack에 대한 보호 수준도 달라진다.

결국 **Secure Boot가 신뢰할 수 있는 TEE의 실행을 보장하는 기반이라면, Secure Storage는 그 신뢰 환경을 활용하여 중요한 데이터를 지속적으로 보호하는 기능**이라고 할 수 있다.

### 3. Secure Cryptographic Services
#### 3.1. 개요
**Secure Cryptographic Services**는 TEE의 격리된 실행 환경에서 암호화(Encryption), 복호화(Decryption), 해시(Hash), 전자서명(Digital Signature) 등의 암호학적 연산을 수행하는 보안 기능이다.

일반적인 운영체제에서도 OpenSSL과 같은 암호화 라이브러리를 이용하여 다양한 암호학적 연산을 수행할 수 있다. 그러나 암호화 연산에 사용되는 비밀키가 일반 운영체제의 메모리에 존재한다면, 공격자가 커널 취약점이나 메모리 노출 취약점을 악용하여 해당 키를 탈취할 가능성이 있다.

TEE는 이러한 위협에 대응하기 위해 암호학적 연산과 키 관리를 일반 운영체제로부터 격리된 Secure World에서 수행할 수 있도록 한다.

암호화 알고리즘 자체가 일반 운영체제에서 사용하는 것과 다른 것은 아니다. AES, RSA, ECC, SHA 등의 표준 암호 알고리즘을 사용하더라도, 암호화 키를 관리하고 연산을 수행하는 환경이 다르기 때문에 보안상 차이가 발생한다.

즉, Secure Cryptographic Services의 핵심은 새로운 암호화 알고리즘을 제공하는 것이 아니라, **암호화 키와 민감한 중간 연산 데이터를 일반 운영체제로부터 보호된 환경에서 처리하는 것**에 있다.

#### 3.2. Secure Cryptographic Service 동작 원리
ARM TrustZone 기반 TEE에서는 일반 운영체제가 직접 비밀키를 처리하는 대신, Secure World에서 실행되는 Trusted Application(TA)에 암호학적 연산을 요청할 수 있다.

대표적인 동작 과정은 다음과 같다.

![TEE Cryptographic Service Architecture](/assets/images/TEE_Cryptographic_Service.png){: .align-center .diagram}

1. 암호 연산 요청: 일반 애플리케이션(Client Application)이 TEE Client API를 통해 암호화, 복호화 또는 전자서명 등의 작업을 요청한다.

2. Secure World 전환: Linux TEE Driver와 Secure Monitor를 거쳐 OP-TEE에 요청이 전달된다.

3. 암호화 키 접근: Trusted Application은 Secure Storage에 저장된 키를 불러오거나 TEE 내부에서 생성한 키를 사용한다.

4. 암호 연산 수행: Secure OS가 제공하는 암호화 API를 통해 요청된 암호학적 연산을 수행한다.

5. 결과 반환: 연산 결과를 Trusted Application에 전달하고, 필요한 경우 일반 애플리케이션에 반환한다.

예를 들어 전자서명을 생성하는 경우, 일반 애플리케이션은 서명할 데이터 또는 해시값을 TEE에 전달하고, Secure World에서 보호되는 개인키를 사용하여 서명을 생성할 수 있다.

이때 일반 애플리케이션에는 생성된 전자서명만 반환되며, 개인키 자체를 Normal World로 전달하지 않고 연산을 수행하도록 설계할 수 있다.

따라서 일반 운영체제가 침해되더라도 개인키를 직접 추출하는 공격에 대한 보호 수준을 높일 수 있다.

#### 3.3. HSM vs TEE Cryptographic Services
TEE의 암호화 기능은 HSM(Hardware Security Module)과 유사한 보안 목적을 갖지만, 두 기술은 동일하지 않다.

HSM은 암호화 키의 보호와 암호학적 연산을 위해 설계된 전용 보안 하드웨어인 반면, ARM TrustZone 기반 TEE는 일반 프로세서의 하드웨어 보안 기능을 활용하여 격리된 실행 환경을 제공한다.

따라서 TEE는 별도의 HSM 없이도 암호화 키와 연산을 일반 운영체제로부터 격리할 수 있지만, 모든 TEE가 HSM과 동일한 물리적 공격 저항성이나 키 보호 수준을 제공하는 것은 아니다.

또한 일부 SoC에서는 TEE와 별도의 하드웨어 암호화 엔진 또는 보안 프로세서를 함께 사용하기도 한다.

#### 3.4 비고 : OP-TEE Cryptographic API
추후 다룰 OP-TEE는 GlobalPlatform에서 정의한 **TEE Internal Core API**를 구현하여 Trusted Application이 암호학적 연산을 수행할 수 있도록 지원한다.

대표적인 API는 다음과 같다.

<div align="center" markdown="1">

| API | 주요 역할 |
|---|---|
| `TEE_AllocateOperation()` | 암호 연산 객체 생성 |
| `TEE_GenerateKey()` | 암호화 키 생성 |
| `TEE_SetOperationKey()` | 연산에 사용할 키 설정 |
| `TEE_CipherEncrypt()` | 데이터 암호화 |
| `TEE_CipherDecrypt()` | 데이터 복호화 |
| `TEE_DigestDoFinal()` | 해시 연산 수행 |
| `TEE_AsymmetricSignDigest()` | 비대칭키 기반 전자서명 생성 |
| `TEE_AsymmetricVerifyDigest()` | 전자서명 검증 |
| `TEE_GenerateRandom()` | 암호학적 난수 생성 |

</div>

이러한 API를 이용하면 Trusted Application 개발자는 암호 알고리즘의 세부 구현을 직접 작성하지 않고도 Secure OS에서 제공하는 암호화 기능을 사용할 수 있다.   

OP-TEE는 내부적으로 소프트웨어 암호화 라이브러리 또는 플랫폼에서 지원하는 하드웨어 암호화 가속 기능을 활용할 수 있다. 다만 실제 사용 가능한 알고리즘과 하드웨어 가속 여부는 OP-TEE의 빌드 설정 및 플랫폼 구현에 따라 달라진다.

## 마치며

이번 글에서는 TEE(Trusted Execution Environment)의 기본 개념부터 ARM TrustZone을 활용한 실행 환경의 격리 원리, 그리고 TEE가 제공하는 주요 보안 기능에 대해 살펴보았다.

TEE의 핵심은 단순히 중요한 데이터를 암호화하거나 별도의 공간에 저장하는 것이 아니다. 일반 운영체제가 침해되더라도 민감한 코드와 데이터를 보호할 수 있도록 **하드웨어 수준에서 분리된 신뢰 실행 환경을 제공한다는 점**에 있다.

이를 위해 ARM TrustZone은 Normal World와 Secure World를 구분하고, Secure Boot는 신뢰할 수 있는 보안 소프트웨어가 실행되도록 지원한다. 또한 Secure Storage와 Secure Cryptographic Services는 이러한 격리 환경을 기반으로 민감한 데이터와 암호화 키를 보호한다.

그러나 TEE가 일반 운영체제와 격리되어 있다는 사실이 모든 보안 위협으로부터 안전하다는 의미는 아니다. Secure OS나 Trusted Application 역시 소프트웨어로 구현되며, 일반 운영체제와 통신하기 위한 인터페이스가 존재한다. 따라서 이러한 구성 요소에 취약점이 존재한다면 TEE의 보안 경계 역시 공격 대상이 될 수 있다.

결국 TEE의 보안성을 이해하기 위해서는 하드웨어가 제공하는 격리 기능뿐만 아니라, **그 위에서 실행되는 Secure OS가 어떻게 구현되고 외부 요청을 처리하는지** 살펴볼 필요가 있다.

다음 글에서는 ARM TrustZone 기반의 대표적인 오픈소스 TEE 구현체인 **OP-TEE(Open Portable Trusted Execution Environment)** 를 다룰 예정이다. OP-TEE의 전체 아키텍처와 주요 구성 요소를 살펴보고, Normal World의 Client Application이 Secure World의 Trusted Application을 호출하는 과정이 실제로 어떻게 구현되어 있는지 알아보고자 한다.
