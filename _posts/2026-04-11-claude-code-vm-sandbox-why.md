---
title: "Claude code 샌드박스 구축하기 - 사전 지식"
date: 2026-04-11 23:54:23 +0900
categories: [DevEnv]
tags: [claude-code, VM]
---

> 이 글은 AI Agent가 초안 작성 후 본인이 수정했습니다.

## 요약

| 항목 | Docker (컨테이너) | KVM (VM) |
|---|---|---|
| 격리 경계 | Linux 커널 공유 | 별도 커널, 하드웨어 가상화 |
| 탈출 난이도 | 커널/런타임 취약점 하나면 탈출 가능 | 하이퍼바이저 자체를 뚫어야 함 |
| 공격 표면 | Linux 커널 전체 | 하이퍼바이저 코드 (훨씬 좁음) |
| 격리 보장 | 소프트웨어 수준 | 하드웨어(VT-x/AMD-V) 수준 |

---

Claude Code의 `--dangerously-skip-permissions` 모드를 써서 권한 점검할 필요 없이 바로바로 처리하게 해봤다.
안전한 구성을 위해 VM을 활용해 샌드박스를 구축해보려고 한다.
[GeekNews](https://news.hada.io/topic?id=26002)를 참고했습니다 :)

---

## --dangerously-skip-permissions 란?

Claude Code는 파일 읽기, 코드 실행, 외부 명령어 호출 등 민감한 작업을 수행할 때마다 사용자에게 권한을 요청한다.
`--dangerously-skip-permissions` 모드는 이 모든 권한 요청을 **자동으로 승인**한다.

```bash
claude --dangerously-skip-permissions
```

CI 파이프라인이나 자동화 스크립트에서 유용하지만, 위험성이 있다.
Claude가 실행하는 코드나 명령어 중 악의적인 것이 섞여 있어도 **아무런 제동 없이 실행**된다.
이 상태에서 Claude가 `rm -rf /`를 실행한다면? 시스템 전체가 날아간다.

따라서 이 모드는 **격리된 환경 안에서만** 사용하는 것이 원칙이다.

---

## Docker로 충분하지 않을까?

가장 먼저 떠오르는 격리 수단은 Docker다. 하지만 Docker는 컨테이너이지 VM이 아니다.
이 차이가 보안에 있어 결정적이다.

### 컨테이너는 커널을 공유한다

Docker 컨테이너는 호스트 OS의 커널을 그대로 공유한다.
컨테이너 내부에서 실행되는 프로세스도 결국 같은 Linux 커널 위에서 돌아간다.

```
[호스트 프로세스]  [컨테이너 프로세스]
        ↓                   ↓
    ┌──────────────────────────┐
    │      Linux Kernel        │  ← 공유
    └──────────────────────────┘
```

컨테이너는 `namespace`와 `cgroup`으로 격리를 구현한다.

| 기술 | 역할 |
|---|---|
| namespace | 프로세스, 네트워크, 파일시스템 등을 분리된 뷰로 제공 |
| cgroup | CPU, 메모리 등 리소스 사용량 제한 |

이것들은 **가시성과 리소스**를 제한할 뿐, 커널 자체는 공유된다.

### 커널 공유 = 탈출 가능성

커널이나 컨테이너 런타임에 취약점이 있으면, 컨테이너 안에서도 그 취약점을 이용할 수 있다.
대표적인 역사적 사례가 **CVE-2019-5736**이다.
`runc`(Docker 런타임)의 취약점으로, 컨테이너 안에서 호스트의 `runc` 바이너리를 덮어쓰는 방식으로 탈출이 가능했다.
이 취약점은 2019년에 패치됐지만, 이런 **컨테이너 탈출(container escape) 공격 유형 자체는 지속적으로 발견**되고 있다.

**namespace/cgroup 격리 ≠ 하드웨어 격리**

커널을 공유하는 이상, 런타임이나 커널 레벨의 취약점 하나가 호스트 전체를 노출시킬 수 있다.
Docker 컨테이너는 그 구조적 한계를 벗어날 수 없다.

> seccomp, AppArmor, rootless Docker 같은 보안 강화 옵션을 적용해도 커널 공유라는 구조 자체는 해소되지 않는다.

---

## 하이퍼바이저와 KVM
VM은 완전히 안전하진 않지만, 그래도 공격 표면을 현저히 줄일 수 있다.

### 하이퍼바이저란?

하이퍼바이저는 물리 하드웨어 위에서 여러 개의 가상 머신을 실행하는 소프트웨어다.
두 가지 유형이 있다.

| 구분 | Type 1 (Bare-metal) | Type 2 (Hosted) |
|---|---|---|
| 실행 위치 | 하드웨어 위에서 직접 실행 | 호스트 OS 위에서 실행 |
| 예시 | Xen, VMware ESXi | VirtualBox, VMware Workstation |
| 성능 | 높음 | 낮음 |
| 격리 수준 | 강함 | 상대적으로 약함 |

### KVM — Linux 커널과 통합된 하이퍼바이저

KVM(Kernel-based Virtual Machine)은 Linux 커널 모듈로 내장된 하이퍼바이저다.
커널 모듈로 이미 내장되어 있어 외부 소프트웨어를 따로 설치할 필요가 없다.

Intel CPU의 VT-x, AMD CPU의 AMD-V 같은 하드웨어 가상화 확장을 활용해
VM이 CPU 명령어를 **직접 실행**할 수 있게 한다.

> KVM은 통상 Type 1으로 분류되지만, 호스트 Linux OS 위에서 동작한다는 특성상 "Type 1.5" 또는 하이브리드로 보는 시각도 있다. Red Hat, IBM 등 주요 업체는 Type 1으로 분류한다.

### QEMU — VM 실행을 담당하는 에뮬레이터

KVM은 커널 모듈이라 단독으로 VM을 띄울 수 없다.
QEMU가 디스크, 네트워크 카드, 메모리 등 가상 하드웨어를 에뮬레이션하고,
KVM이 CPU 가상화를 담당하는 방식으로 함께 동작한다.

```
VM 프로세스
    ↓
QEMU (가상 하드웨어 에뮬레이션)
    ↓
KVM (CPU 하드웨어 가속)
    ↓
Intel VT-x / AMD-V
```

VM 안의 프로세스는 격리된 **별도의 커널** 위에서 실행된다.
QEMU 자체의 취약점을 통한 VM escape 가능성이 이론적으로는 남아있지만,
공격 표면이 Linux 커널 전체를 노출하는 컨테이너 방식 대비 현저히 좁다.

---

## libvirt — 하이퍼바이저 통합 관리 API

KVM/QEMU를 직접 다루는 것은 복잡하다. 이 복잡함을 해소하는 것이 **libvirt**다.

### libvirt란?

libvirt는 Red Hat이 개발한 오픈소스 프로젝트로,
**여러 하이퍼바이저를 하나의 통일된 API로 관리**할 수 있게 해준다.

```
[관리 도구: virsh, Vagrant, virt-manager ...]
                    ↓
            libvirt API / libvirtd
                    ↓
    ┌───────────────────────────────┐
    │  KVM │  Xen │  VMware │  LXC │  ← 하이퍼바이저들
    └───────────────────────────────┘
```

### hypervisor-agnostic 설계

libvirt의 핵심 설계 철학은 **hypervisor-agnostic**이다.
KVM을 쓰든 Xen을 쓰든, 관리 도구는 동일한 libvirt API를 호출하면 된다.
백엔드 하이퍼바이저가 바뀌어도 상위 레이어의 코드는 변경할 필요가 없다.

### libvirtd 데몬

`libvirtd`는 백그라운드에서 실행되는 데몬으로,
VM 생성/시작/중지/삭제 같은 요청을 받아서 하이퍼바이저에 전달한다.

> Fedora 37+ 등 일부 배포판에서는 모놀리식 `libvirtd` 대신 `virtqemud` 등 모듈형 데몬 아키텍처로 전환하고 있다. Ubuntu에서는 여전히 `libvirtd`가 기본이다.

### virsh

`virsh`는 libvirt API를 직접 호출하는 CLI 도구다.

```bash
virsh list --all       # VM 목록 조회
virsh start myvm       # VM 시작
virsh shutdown myvm    # VM 종료
```

강력하지만 저수준이다. VM 하나 만들려면 XML 정의 파일을 직접 작성해야 한다.

---

## Vagrant — libvirt 위의 개발자 친화적 추상화

virsh로 VM을 관리하는 것은 번거롭다. **Vagrant**는 이 복잡함을 감춰준다.

### Vagrant란?

Vagrant는 VM 환경을 코드(`Vagrantfile`)로 정의하고,
프로비저닝(소프트웨어 자동 설치)까지 자동화해주는 도구다.

```ruby
# Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2204"
  config.vm.provider "libvirt" do |lv|
    lv.memory = 2048
    lv.cpus = 2
  end
end
```

`vagrant up` 한 줄로 VM이 생성되고 부팅된다.

### virsh vs Vagrant

| 항목 | virsh | Vagrant |
|---|---|---|
| 추상화 수준 | 저수준 (libvirt 직접 호출) | 고수준 (선언적 설정) |
| VM 정의 방식 | XML 파일 직접 작성 | Vagrantfile (Ruby DSL) |
| 프로비저닝 | 없음 (직접 구성) | 내장 (`shell`, `ansible` 등) |
| 멀티 프로바이더 | libvirt 전용 | VirtualBox, libvirt, VMware 등 |
| 사용 편의성 | 낮음 | 높음 |

Vagrant는 libvirt를 포함한 여러 VM 백엔드를 동일한 인터페이스로 다룰 수 있다.
`vagrant-libvirt` 플러그인이 Vagrant와 libvirt 사이의 어댑터 역할을 한다.

---

## 최종 스택 구조

정리하면 전체 스택은 다음과 같다.

```
Claude Code (--dangerously-skip-permissions)
            ↓
    Vagrant (Vagrantfile, 선언적 VM 관리)
            ↓
  libvirt / libvirtd (하이퍼바이저 통합 API)
            ↓
    QEMU + KVM (실제 VM 실행 + CPU 가속)
            ↓
    하드웨어 (Intel VT-x / AMD-V)
```

각 레이어가 하는 일:

| 레이어 | 역할 |
|---|---|
| Vagrant | VM 환경 정의 및 프로비저닝 자동화 |
| libvirt | 하이퍼바이저 추상화 API |
| QEMU | 가상 하드웨어 에뮬레이션 |
| KVM | 하드웨어 CPU 가상화 가속 |
| VT-x/AMD-V | 물리 CPU의 하드웨어 지원 |

VM 안에서 실행되는 Claude Code는 호스트 시스템과 커널 수준에서 분리되어 있다.
설령 Claude가 위험한 명령을 실행하더라도, 피해가 VM 안에 국한될 가능성이 컨테이너 방식보다 훨씬 높다.

> 실무에서는 Firecracker microVM이나 Kata Containers 같은 경량 VM 대안도 쓰인다. 이 시리즈에서는 로컬 개발 환경에서 범용성 높은 KVM + Vagrant 조합을 선택했다.

다음 편에서는 이 스택을 직접 설치하고 Vagrantfile을 작성해 실제로 Claude Code를 돌려본다.

---

## 참고 자료

- [KVM - Kernel Virtual Machine (Red Hat)](https://www.redhat.com/en/topics/virtualization/what-is-KVM)
- [libvirt: The virtualization API](https://libvirt.org/)
- [CVE-2019-5736 Detail - NVD](https://nvd.nist.gov/vuln/detail/cve-2019-5736)
- [Firecracker: Lightweight Virtualization for Serverless Applications](https://firecracker-microvm.github.io/)
- [Claude Code Sandboxing Docs](https://docs.anthropic.com/en/docs/claude-code/security)
