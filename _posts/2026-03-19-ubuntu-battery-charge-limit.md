---
title: "Ubuntu에서 배터리 충전 제한 및 전원 프로필 자동 전환 설정하기"
date: 2026-03-19 00:00:00 +0900
categories: [Linux]
tags: [battery, tlp, power-management]
---

## 최종 결과

| 조건 | 동작 | 담당 |
|------|------|------|
| AC 연결 | 성능(performance) 모드 | TLP |
| 배터리 사용 | 균형(balanced) 모드 | TLP |
| 배터리 20% 미만 | 절전(low-power) 모드 | systemd timer + 스크립트 |
| 충전 제한 | ~60%에서 충전 중단 | TLP (conservation mode) |

```bash
# 검증 명령어
cat /sys/bus/platform/drivers/ideapad_acpi/VPC2004:00/conservation_mode  # 1이면 활성화
cat /sys/firmware/acpi/platform_profile  # AC: performance, BAT: balanced
cat /sys/class/power_supply/BAT1/status  # conservation mode + 60% 이상이면 "Not charging"
systemctl list-timers battery-low-powersave.timer  # 타이머 동작 확인
```

> 이 글은 **Lenovo Yoga Slim 7 (IdeaPad 계열)** + **Ubuntu 24.04** 환경에서 작성되었다. 다른 모델에서는 sysfs 경로가 다를 수 있다.
{: .prompt-info }

---

## 1. TLP 설치

### power-profiles-daemon 비활성화

`power-profiles-daemon`은 GNOME이 기본 제공하는 전원 관리 데몬으로, 설정 > 전원에서 균형/성능/절전을 선택하는 기능을 담당한다. TLP과 역할이 겹치므로 반드시 먼저 비활성화해야 한다.

```bash
sudo systemctl disable --now power-profiles-daemon
sudo apt install -y tlp
```

> 비활성화하면 GNOME 설정의 전원 프로필 선택기(성능/균형/절전 드롭다운)가 사라진다. TLP가 AC/배터리 상태에 따라 자동으로 프로필을 전환하므로 수동 선택이 불필요하다. 현재 프로필은 `cat /sys/firmware/acpi/platform_profile`로 확인할 수 있다.
{: .prompt-info }

### 커널 모듈 확인

conservation mode가 동작하려면 `ideapad_laptop` 커널 모듈이 로드되어야 한다.

```bash
lsmod | grep ideapad_laptop
# 출력이 없으면:
sudo modprobe ideapad_laptop
```

> 최신 Lenovo 모델 중 일부는 UEFI(BIOS) 설정에서 직접 충전 제한을 설정할 수 있다. 재부팅 후 F2(또는 Fn+F2)로 BIOS에 진입하여 Configuration > Battery 메뉴를 확인하자. UEFI에서 이미 설정했다면 OS에서 별도로 conservation mode를 켤 필요 없다.
{: .prompt-tip }

### TLP 설정 파일 작성

`/etc/tlp.conf`를 직접 수정해도 되지만, `/etc/tlp.d/`에 drop-in 파일을 만드는 것이 관리에 유리하다. drop-in 파일은 패키지 업데이트 시 덮어씌워지지 않고, 설정 변경을 한 파일에 모아 관리할 수 있다.

```ini
# /etc/tlp.d/01-custom.conf

# --- CPU Performance ---
CPU_ENERGY_PERF_POLICY_ON_AC=performance
CPU_ENERGY_PERF_POLICY_ON_BAT=balance_power

# Platform profile: AC=performance, BAT=balanced
PLATFORM_PROFILE_ON_AC=performance
PLATFORM_PROFILE_ON_BAT=balanced

# Turbo boost: enable on AC, disable on BAT
CPU_BOOST_ON_AC=1
CPU_BOOST_ON_BAT=0

# --- Battery Care ---
# Lenovo IdeaPad conservation mode (0=off, 1=on, charges up to ~60%)
START_CHARGE_THRESH_BAT0=0
STOP_CHARGE_THRESH_BAT0=1

# --- Wi-Fi ---
WIFI_PWR_ON_AC=off
WIFI_PWR_ON_BAT=on
```

> IdeaPad에서 `STOP_CHARGE_THRESH_BAT0`은 퍼센트가 아니라 conservation mode의 on/off 값이다. `1`이 활성화를 의미한다. `START_CHARGE_THRESH_BAT0=0`은 IdeaPad에서는 무시되지만, TLP가 경고 없이 동작하도록 함께 지정한다.
{: .prompt-warning }

### TLP 시작

```bash
sudo systemctl enable --now tlp
sudo tlp start
```

---

## 2. 배터리 20% 미만 절전 모드 자동 전환

TLP는 AC/BAT 두 상태만 구분한다. 배터리 잔량에 따른 프로필 전환은 별도 스크립트 + systemd timer로 구현한다.

### 스크립트 작성

```bash
#!/bin/bash
# /usr/local/bin/battery-low-powersave.sh

THRESHOLD=20
BAT_PATH=$(find /sys/class/power_supply -maxdepth 1 -name "BAT*" -print -quit 2>/dev/null)

# Exit if no battery or platform_profile not supported
[ -n "$BAT_PATH" ] && [ -f "$BAT_PATH/capacity" ] || exit 0
[ -f /sys/firmware/acpi/platform_profile ] || exit 0

CAPACITY=$(cat "$BAT_PATH/capacity")
STATUS=$(cat "$BAT_PATH/status")

# Only act when on battery (Discharging)
if [ "$STATUS" = "Discharging" ] && [ "$CAPACITY" -lt "$THRESHOLD" ]; then
    CURRENT_PROFILE=$(cat /sys/firmware/acpi/platform_profile)
    if [ "$CURRENT_PROFILE" != "low-power" ]; then
        echo low-power > /sys/firmware/acpi/platform_profile || logger "battery-low-powersave: failed to set platform_profile"
        echo power | tee /sys/devices/system/cpu/cpu*/cpufreq/energy_performance_preference > /dev/null 2>&1
        logger "battery-low-powersave: battery at ${CAPACITY}%, switched to low-power"
    fi
elif [ "$STATUS" = "Discharging" ] && [ "$CAPACITY" -ge "$THRESHOLD" ]; then
    CURRENT_PROFILE=$(cat /sys/firmware/acpi/platform_profile)
    if [ "$CURRENT_PROFILE" = "low-power" ]; then
        echo balanced > /sys/firmware/acpi/platform_profile || logger "battery-low-powersave: failed to set platform_profile"
        echo balance_power | tee /sys/devices/system/cpu/cpu*/cpufreq/energy_performance_preference > /dev/null 2>&1
        logger "battery-low-powersave: battery at ${CAPACITY}%, restored to balanced"
    fi
fi
```

```bash
sudo chmod +x /usr/local/bin/battery-low-powersave.sh
```

주요 설계 결정:
- `BAT_PATH`를 `find`로 자동 감지한다. 시스템마다 `BAT0`일 수도 `BAT1`일 수도 있기 때문이다
- `echo > cpu*` 대신 `echo | tee cpu*`를 사용한다. `tee`는 표준 입력을 받아 지정된 파일 **모두**에 동시에 쓰는 명령이다. redirect(`>`)는 glob이 여러 파일로 확장되면 마지막 파일에만 쓰기되지만, `tee`는 모든 CPU 코어에 적용된다
- `platform_profile` 쓰기 실패 시 `logger`로 syslog에 기록한다

> 이 스크립트는 `platform_profile`과 `energy_performance_preference`에 직접 값을 쓴다. TLP도 같은 sysfs 경로를 관리하므로, TLP가 재적용되면(예: suspend 복귀 시) 스크립트가 설정한 값이 TLP 설정으로 덮어씌워질 수 있다. 이 경우 타이머가 2분 내에 다시 `low-power`로 복구한다.
{: .prompt-warning }

### systemd 타이머 등록

systemd의 타이머(timer)는 cron과 비슷하게 주기적으로 서비스를 실행하는 기능이다. `Type=oneshot`는 실행 후 바로 종료되는 일회성 서비스를 의미한다.

```ini
# /etc/systemd/system/battery-low-powersave.service
[Unit]
Description=Switch to low-power when battery below 20%
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/battery-low-powersave.sh
```

```ini
# /etc/systemd/system/battery-low-powersave.timer
[Unit]
Description=Check battery level every 2 minutes

[Timer]
OnBootSec=1min
OnUnitActiveSec=2min
AccuracySec=10s

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload   # systemd에게 새 서비스 파일을 인식시키는 명령
sudo systemctl enable --now battery-low-powersave.timer
```

2분마다 배터리 잔량을 확인하고, 20% 미만이면 `low-power` 프로필로 전환한다. 20% 이상으로 복귀하면 다시 `balanced`로 돌아간다.

> 2분 폴링 방식이므로 최악의 경우 20% 아래로 떨어진 후 약 2분간 `balanced`로 동작할 수 있다. 더 즉각적인 반응이 필요하면 `upower --monitor-detail`을 파이프로 받아 이벤트 기반으로 전환하는 방식도 있지만, 구현과 디버깅이 복잡해지므로 여기서는 단순한 폴링 방식을 선택했다.
{: .prompt-info }

---

## 왜 TLP인가

충전 제한만 걸려면 systemd 서비스 하나로 충분하다. 하지만 전원 프로필 전환까지 함께 관리하려면 TLP가 더 적합하다.

| | systemd 서비스 | TLP |
|---|---|---|
| 추가 패키지 | 불필요 | `tlp` 설치 필요 |
| 충전 제한 | 스크립트 직접 작성 | 설정 한 줄 |
| 전원 프로필 | 직접 구현 필요 | AC/BAT별 프로필 내장 |
| power-profiles-daemon 충돌 | 없음 | **있음 — 비활성화 필요** |

---

## 배터리 충전 제한의 원리

### 왜 충전을 제한하면 배터리 수명이 늘어나는가

리튬이온 배터리는 높은 충전 상태(State of Charge, SoC)에서 오래 유지될수록 양극 물질의 산화와 SEI(Solid Electrolyte Interphase) 층 성장이 가속되어 용량이 감소한다. Battery University에 따르면 100% 충전 상태에서 보관하면 연간 약 20%의 용량 감소가 발생하지만, 50~60% 상태에서는 약 4% 수준으로 줄어든다.

배터리 관리에서 널리 알려진 **"40-80 규칙"**(40%~80% 사이에서 충전/방전)을 따르면 수명을 2~4배 연장할 수 있다. IdeaPad의 conservation mode는 ~60%에서 충전을 멈추므로, 이 범위의 중간값에 해당한다. ThinkPad처럼 정밀 제어가 가능한 기종에서는 start=40, stop=80으로 설정하는 것이 일반적이다.

> 온도도 배터리 수명에 큰 영향을 미친다. 고온(35도 이상)에서 높은 SoC를 유지하면 열화가 훨씬 빨라진다. 배터리 사용 시 `balanced`/`low-power`로 전환하면 CPU 발열이 줄어들어, 충전 제한과 함께 이중으로 배터리 수명을 보호하는 효과가 있다.
{: .prompt-tip }

### sysfs를 통한 제어

Linux 커널은 일부 노트북 벤더의 배터리 관리 인터페이스를 sysfs를 통해 노출한다. Lenovo IdeaPad 계열은 아래 경로로 conservation mode를 제어할 수 있다.

```bash
# 경로는 모델마다 다를 수 있으므로 아래 명령으로 확인
find /sys/bus/platform/drivers/ideapad_acpi/ -name conservation_mode

# 현재 상태 확인
cat /sys/bus/platform/drivers/ideapad_acpi/VPC2004:00/conservation_mode
```

이 값을 `1`로 설정하면 배터리가 약 60%에서 충전을 멈춘다. 직접 `echo`로 쓰면 재부팅 시 초기화되므로, TLP로 영구 적용하는 것이 좋다.

> ThinkPad 계열은 `charge_control_end_threshold`를 지원하며, 정확한 퍼센트 지정이 가능하다. IdeaPad은 conservation mode(on/off)만 지원하여 ~60% 고정이다.
{: .prompt-info }

### 주의: AC 연결 시 배터리 퍼센트가 안 내려가는 건 정상

conservation mode를 켜도 AC가 연결되어 있으면 전원 어댑터에서 직접 전력을 공급하기 때문에 배터리 퍼센트가 줄어들지 않는다. AC를 뽑아서 배터리를 60% 아래로 소모한 뒤 다시 꽂으면, 그때부터 60%에서 충전이 멈추는 것을 확인할 수 있다.

---

## 되돌리기

설정을 원래 상태로 복구하려면 아래 명령을 순서대로 실행한다.

```bash
# 타이머 및 스크립트 제거
sudo systemctl disable --now battery-low-powersave.timer
sudo rm /etc/systemd/system/battery-low-powersave.service
sudo rm /etc/systemd/system/battery-low-powersave.timer
sudo rm /usr/local/bin/battery-low-powersave.sh
sudo systemctl daemon-reload

# TLP 제거 및 GNOME 전원 관리 복구
sudo apt remove tlp
sudo systemctl enable --now power-profiles-daemon
```

---

## 정리

- Lenovo IdeaPad의 conservation mode는 **~60% 고정**이며 퍼센트를 직접 지정할 수 없다
- TLP를 쓰면 `power-profiles-daemon`은 반드시 비활성화해야 한다 — GNOME 설정의 전원 프로필 선택기가 사라지지만, TLP가 자동 전환하므로 기능 손실은 없다
- 배터리 잔량 기반 프로필 전환은 TLP만으로는 불가능하며, systemd timer + 스크립트 조합이 필요하다
- GNOME UI 상태 표시가 느릴 수 있으므로, `upower -i /org/freedesktop/UPower/devices/battery_BAT1`로 실제 상태를 확인하자

---

## 참고 자료

- [TLP - Battery Care 설정](https://linrunner.de/tlp/settings/battery.html)
- [TLP - Battery Care Vendor Specifics (Lenovo IdeaPad)](https://linrunner.de/tlp/settings/bc-vendors.html)
- [BU-808: How to Prolong Lithium-based Batteries](https://batteryuniversity.com/article/bu-808-how-to-prolong-lithium-based-batteries)
- [ArchWiki - TLP](https://wiki.archlinux.org/title/TLP)
