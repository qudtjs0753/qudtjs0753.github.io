---
title: "Vagrant + KVM으로 Claude Code 샌드박스 구축하기"
date: 2026-04-11 23:54:23 +0900
categories: [DevEnv]
tags: [VM, AI]
---

> 이 글은 AI Agent가 작성했습니다.

## 최종 구성 요약

```
claude --dangerously-skip-permissions   ← VM 안에서 실행
        ↓
  Vagrant + vagrant-libvirt
        ↓
  libvirtd (KVM 백엔드)
        ↓
  QEMU + KVM (하드웨어 격리)
```

| 구성 요소 | 역할 |
|---|---|
| `qemu-kvm` + `libvirt` | VM 실행 및 관리 |
| `vagrant-libvirt` | Vagrant ↔ libvirt 연결 |
| `generic/ubuntu2204` | libvirt 지원 박스 |
| SSH Agent Forwarding | 키 파일 없이 git 인증 |

---

[이전 글]({% post_url 2026-04-11-claude-code-vm-sandbox-why %})에서 왜 Docker가 아닌 KVM 기반 VM 샌드박스가 필요한지 설명했다.
이번에는 실제로 Vagrant + KVM 환경을 구축하고 `--dangerously-skip-permissions` 모드로 Claude Code를 실행하는 과정을 다룬다.

---

## 사전 준비

- Ubuntu (Linux) 호스트
- KVM 하드웨어 지원 확인

```bash
grep -E 'vmx|svm' /proc/cpuinfo
```

출력이 있으면 Intel VT-x(vmx) 또는 AMD-V(svm)가 활성화된 것이다.

---

## Step 1 — QEMU/KVM + libvirt 설치

```bash
sudo apt-get update && sudo apt-get install -y \
  qemu-kvm \
  libvirt-daemon-system \
  libvirt-clients \
  bridge-utils \
  libvirt-dev \
  libxslt-dev \
  libxml2-dev \
  zlib1g-dev \
  ruby-dev \
  ebtables \
  dnsmasq-base
```

각 패키지의 역할:

| 패키지 | 역할 |
|---|---|
| `qemu-kvm` | VM 실행 엔진. QEMU 에뮬레이터 + KVM 가속을 함께 설치 |
| `libvirt-daemon-system` | `libvirtd` 데몬 본체. Vagrant가 이 데몬과 통신해 VM을 제어함 |
| `libvirt-clients` | `virsh` 등 CLI 도구. 데몬 상태 확인과 디버깅에 사용 |
| `bridge-utils` | VM 네트워크를 호스트에 브리지로 연결할 때 필요 |
| `libvirt-dev` | libvirt C 헤더 파일. `vagrant-libvirt`가 `ruby-libvirt` gem을 네이티브 빌드할 때 필요 |
| `libxslt-dev`, `libxml2-dev`, `zlib1g-dev` | `ruby-libvirt` 빌드 시 필요한 추가 개발 헤더 |
| `ruby-dev` | Ruby 네이티브 확장 빌드에 필요한 Ruby 헤더 |
| `ebtables`, `dnsmasq-base` | libvirt의 VM 네트워크 관리에 필요한 유틸리티 |

> **주의**: `libvirt-dev`만 설치하고 `vagrant plugin install vagrant-libvirt`를 실행하면 실제로는 더 많은 패키지가 없어 빌드에 실패하는 경우가 많다. 위 패키지를 모두 설치한 후 진행하자.

---

## Step 2 — 사용자 그룹 추가

```bash
sudo usermod -aG libvirt $USER
sudo usermod -aG kvm $USER
```

| 그룹 | 역할 |
|---|---|
| `libvirt` | `/var/run/libvirt/libvirt-sock` 소켓 접근 권한. 없으면 `vagrant up` 시 Permission denied |
| `kvm` | `/dev/kvm` 디바이스 접근 권한. QEMU가 KVM 하드웨어 가속을 사용하려면 필요 |

그룹을 즉시 적용하려면 현재 셸을 재로그인 셸로 교체한다.

```bash
exec su -l $USER
```

적용 확인:

```bash
groups
# 출력에 libvirt, kvm이 포함되어야 함
```

---

## Step 3 — libvirtd 데몬 활성화

```bash
sudo systemctl enable --now libvirtd
systemctl status libvirtd
```

`active (running)` 상태면 정상이다.

---

## Step 4 — vagrant-libvirt 플러그인 설치

```bash
vagrant plugin install vagrant-libvirt
```

Vagrant는 기본적으로 VirtualBox 백엔드로 설계되어 있다.
`vagrant-libvirt` 플러그인은 Vagrant가 libvirt/KVM 백엔드와 통신할 수 있게 해주는 어댑터다.

---

## Step 5 — Vagrantfile 작성

샌드박스용 디렉토리를 만들고 Vagrantfile을 작성한다.

```bash
mkdir -p ~/workspace/claude-sandbox
cd ~/workspace/claude-sandbox
```

```ruby
# Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2204"

  # 호스트 SSH 에이전트 포워딩 (키 파일을 VM에 복사하지 않고 인증)
  config.ssh.forward_agent = true

  # 워크스페이스 마운트 (rsync 방식 명시 — 기본값이 NFS라 nfs-kernel-server 없으면 실패)
  config.vm.synced_folder "~/workspace", "/workspace", type: "rsync"

  # VM 네트워크 (호스트-전용, DHCP)
  config.vm.network "private_network", type: "dhcp"

  # libvirt 프로바이더 설정
  config.vm.provider "libvirt" do |lv|
    lv.memory = 2048
    lv.cpus = 2
  end

  # Claude Code 자동 설치 프로비저닝
  config.vm.provision "shell", inline: <<-SHELL
    apt-get update -q
    curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
    apt-get install -y nodejs git
    npm install -g @anthropic-ai/claude-code
  SHELL
end
```

### 박스 선택 주의사항

`ubuntu/jammy64`는 VirtualBox 전용 박스라 libvirt를 지원하지 않는다.

```
The box you're attempting to add doesn't support the provider
you requested. Requested provider: libvirt
```

libvirt를 지원하는 `generic/ubuntu2204`를 사용해야 한다.

| 박스 | VirtualBox | libvirt |
|---|---|---|
| `ubuntu/jammy64` | O | X |
| `generic/ubuntu2204` | O | O |

---

## Step 6 — SSH Agent Forwarding 설정

git 작업을 위한 SSH 키를 VM 안으로 어떻게 전달할까?

### 파일 마운트 방식의 문제

`~/.ssh`를 그대로 마운트하면 키 파일이 VM 안에 노출된다.
`--dangerously-skip-permissions` 환경에서 VM이 악의적인 코드를 실행한다면,
키 파일이 탈취될 수 있다.

### SSH Agent Forwarding

Agent Forwarding은 **키 파일을 VM에 복사하지 않고**,
호스트의 SSH 에이전트 소켓을 VM 안으로 포워딩하는 방식이다.

```
VM 안의 ssh 명령
    ↓ (forwarded socket)
호스트 ssh-agent
    ↓
GitHub SSH 서버
```

VM 안에서 인증 요청이 발생하면, 소켓을 통해 호스트 에이전트가 대신 서명한다.
키는 호스트를 떠나지 않는다.

| | Agent Forwarding | 파일 마운트 |
|---|---|---|
| 키 파일 VM 내 존재 여부 | 없음 | 있음 |
| VM 탈취 시 키 파일 직접 노출 | 없음 | 있음 |
| 포워딩 소켓 통한 서명 요청 가능 여부 | 가능 | 해당 없음 |
| 설정 난이도 | 낮음 | 낮음 |

> Agent Forwarding은 키 파일 자체는 VM에 존재하지 않지만, VM 안에서 루트 권한을 얻은 프로세스가 포워딩 소켓(`SSH_AUTH_SOCK`)을 통해 호스트 에이전트에 서명 요청을 보내는 것은 가능하다. `--dangerously-skip-permissions` 환경에서 이 점을 인지하고 사용해야 한다.

Vagrantfile에 `config.ssh.forward_agent = true`를 추가하면 활성화된다 (이미 위에 포함).

VM 시작 전에 호스트에서 에이전트에 키를 등록해둔다.

```bash
ssh-add ~/.ssh/id_ed25519
ssh-add -l  # 등록 확인
```

---

## Step 7 — VM 시작 및 접속

```bash
cd ~/workspace/claude-sandbox
vagrant up --provider=libvirt
```

처음 실행 시 박스 이미지 다운로드 + Claude Code 프로비저닝까지 시간이 걸린다.

완료 후 접속:

```bash
vagrant ssh
```

VM 안에서 확인:

```bash
# Claude Code 설치 확인
claude --version

# SSH Agent Forwarding 확인
ssh -T git@github.com
# → Hi username! You've successfully authenticated...
```

---

## 실제 사용법

```bash
# 1. 호스트에서 SSH 키 등록
ssh-add ~/.ssh/id_ed25519

# 2. VM 시작
cd ~/workspace/claude-sandbox
vagrant up --provider=libvirt

# 3. VM 접속
vagrant ssh

# 4. VM 안에서 Claude Code 실행
claude --dangerously-skip-permissions
```

작업이 끝나면:

```bash
# VM 종료 (데이터 유지)
vagrant halt

# VM 삭제 (완전 초기화)
vagrant destroy
```

---

## 마무리

이제 `--dangerously-skip-permissions` 모드가 VM 안에서 실행된다.
호스트 커널과 분리된 환경이므로, 컨테이너 방식 대비 훨씬 좁은 공격 표면만 노출된다.
SSH 키 파일도 VM에 존재하지 않는다. 단, Agent Forwarding 소켓을 통한 서명 요청은 여전히 가능하므로 이 점은 인지하고 사용하자.

---

## 참고 자료

- [vagrant-libvirt 공식 문서](https://vagrant-libvirt.github.io/vagrant-libvirt/)
- [How To Use Vagrant with Libvirt (KVM) on Linux - ComputingForGeeks](https://computingforgeeks.com/using-vagrant-with-libvirt-on-linux/)
- [SSH Agent Forwarding considered harmful - heipei.github.io](https://heipei.github.io/2015/02/26/SSH-Agent-Forwarding-considered-harmful/)
- [Safer SSH agent forwarding - Vincent Bernat](https://vincent.bernat.ch/en/blog/2020-safer-ssh-agent-forwarding)
