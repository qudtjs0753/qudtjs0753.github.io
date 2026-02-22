---
title: "Linux 데스크톱은 어떻게 화면을 그리는가 — X11에서 Wayland까지"
date: 2026-02-22 21:42:23 +0900
categories: [Linux]
tags: [gnome, wayland]
---

"앱 하나를 화면에 띄우는 것"이 간단해 보이지만, 그 뒤에는 커널, 시스템 서비스, 컴포지터, 데스크톱 환경이 층층이 협력하고 있다.
이 글에서는 Linux 데스크톱 스택을 **왜 이런 구조가 되었는지**(역사)와 **실제로 어떻게 동작하는지**(부팅 → 앱 실행 플로우) 두 축으로 정리한다.

---

## 구조 요약

```
+---------------------------------------------------+
| Wayland Clients (Apps)                            |
|   Files, Settings, Terminal, Browser, etc.        |
|   (GTK/Qt handles Wayland protocol)               |
+--------------- Wayland Protocol ------------------+
| gnome-shell process                               |
|   +- GNOME Shell (JS) -- top bar, Dash, Overview  |
|   +- libmutter -- Wayland server + compositor + WM |
|      Clutter (scene graph) -> Cogl -> OpenGL/ES   |
+---------------------------------------------------+
| System Services (systemd, NetworkManager, etc.)   |
+---------------------------------------------------+
| Linux Kernel (DRM/KMS, libinput, process mgmt)    |
+---------------------------------------------------+
```

앱(파일관리자, 터미널 등)은 Wayland **클라이언트**로, Wayland 프로토콜을 통해 컴포지터에 접속한다.
상단바, Dash 같은 GNOME Shell UI는 클라이언트가 아니라 컴포지터 **내부 액터**로, Wayland 프로토콜을 거치지 않고 직접 렌더링된다.

> 이 구조가 왜 이렇게 되었는지, 아래에서 각 레이어의 역사와 함께 설명한다.

---

## 각 레이어의 역사와 역할

### Linux 커널 — 하드웨어 추상화

커널은 CPU, 메모리, GPU, 입력 장치 등 하드웨어를 직접 제어한다. 이 중 화면 출력과 관련된 핵심 서브시스템은 다음과 같다:

| 서브시스템 | 역할 |
|-----------|------|
| **DRM/KMS** | GPU 제어, 디스플레이 모드(해상도/주사율) 설정, GPU 메모리 관리 |
| **libinput** | 키보드, 마우스, 터치패드 등 입력 장치 추상화 |

#### 화면 출력의 원시적 시작: 프레임버퍼

가장 단순한 화면 출력 방법은 `/dev/fb0`(Frame Buffer)에 직접 픽셀 데이터를 쓰는 것이다.
JPEG 파일을 쓰듯 화면을 파일처럼 다루는 방식이지만, root 권한이 필요하고 한 번에 하나의 프로그램만 사용할 수 있다는 치명적인 한계가 있었다.

이 한계를 넘기 위해 X Window System이 등장했고, 이후 DRM이 프레임버퍼를 대체하면서 GPU 추상화를 커널 레벨로 끌어올렸다.

#### DRM/KMS란?

**DRM**(Direct Rendering Manager)은 1999년 Linux 커널에 도입된 GPU 접근 서브시스템이다. 원래 3D 게임을 위해 GPU에 직접 접근하는 인터페이스로 시작했다.

프레임버퍼(`/dev/fb0`)가 "화면 = 하나의 파일"이라는 단순한 모델이었다면, DRM은 GPU의 실제 기능을 유저스페이스에서 사용할 수 있게 열어준다:

- **GPU 메모리 관리** (GEM/TTM) — 텍스처, 렌더 버퍼 등을 GPU 메모리에 할당
- **DMA-BUF** — 프로세스 간 GPU 버퍼 공유 (앱이 그린 버퍼를 컴포지터에 넘길 때 사용)
- **명령 제출** — GPU에게 렌더링 작업을 보냄

**KMS**(Kernel Mode Setting)는 DRM의 일부로, **디스플레이 출력**을 담당한다:

- 해상도, 주사율 설정
- 어떤 버퍼를 모니터에 보여줄지 결정 (page flip)
- 다중 모니터 관리

```
DRM/KMS 이전:
  렌더링: 앱 → X Server(유저스페이스에서 GPU 제어) → GPU
  화면 출력: X Server가 해상도/주사율까지 직접 설정
  (X가 GPU를 독점, 다른 앱은 접근 불가)

DRM/KMS 이후:
  렌더링: 앱 → OpenGL/Vulkan → Mesa → DRM(커널) → GPU
  화면 출력: 컴포지터 → KMS(커널) → 모니터에 page flip
  (커널이 GPU 접근을 중재, 여러 앱이 동시에 GPU 사용 가능)
```

DRM이 렌더링을, KMS가 디스플레이 출력을 각각 커널 레벨에서 처리하면서 X Server가 하던 역할이 대부분 사라졌다. 이것이 Wayland가 X를 대체할 수 있었던 기술적 기반이기도 하다.

---

### X Window System — 1984년, 그리고 한계

**누가**: Bob Scheifler, Jim Gettys (MIT Project Athena, DEC+IBM 협력)

**왜**: 유닉스 워크스테이션에서 여러 프로그램이 동시에 화면을 사용해야 했다. root 권한을 가진 하나의 프로그램(X Server)이 화면을 독점하고, 나머지 앱들은 X 프로토콜로 "사각형 그려줘", "텍스트 써줘" 같은 명령을 보내는 구조다.

```
앱 (X Client) →  X 프로토콜  → X Server → 화면
                "사각형 그려줘"
                "텍스트 써줘"
```

X의 핵심 설계 원칙은 **"메커니즘은 제공하되, 정책은 정하지 않는다"**였다.
X는 창을 그릴 수 있게 해주지만, 창을 어디에 배치할지, 닫기 버튼을 어떻게 생기게 할지는 결정하지 않는다.
그래서 빠진 것들을 채우기 위해 **Window Manager**(창 위치, 닫기 버튼 담당)와 **Compositor**(창을 오프스크린 버퍼에 그린 뒤 합성)가 별도 프로그램으로 등장했다.

#### X가 미들맨이 된 이유

1980년대에는 X의 "사각형 그려라" 같은 단순 명령이 효율적이었다. 하지만 시간이 지나면서:

- 2D 가속을 위해 벤더마다 DDX(Device Dependent X) 드라이버가 필요했고
- 3D 가속(OpenGL), 비디오 재생, 스크린 캡처 등 새 기능이 필요할 때마다 **확장(extension)**을 덕지덕지 붙여야 했고
- GTK, Qt 같은 툴킷과 Chrome 같은 앱이 자체 렌더링을 하게 되면서 X의 그리기 명령은 아무도 안 쓰게 됨

결국 X는 **아무것도 안 하면서 중간에 앉아서 버퍼만 전달하는 미들맨**이 되었다:

```
앱 → X Server → Compositor → X Server → DRM → GPU
     (무의미한 왕복)
```

---

### Wayland — 2008년, 미들맨 제거

**누가**: Kristian Høgsberg (Red Hat). X.Org와 DRI(Direct Rendering Infrastructure) 개발에 참여하면서 "X가 이제 아무 일도 안 하는데 왜 거쳐야 하지?"를 직접 체감한 사람이다.

**핵심 아이디어**: DRM과 Vulkan/OpenGL이 하드웨어 추상화를 다 해주니, X Server를 제거하고 **컴포지터가 직접 디스플레이 서버 역할을 하면 된다.**

```
Wayland 시대:
  앱 → 컴포지터 → DRM → GPU  (직통)
```

Wayland는 **프로토콜(규약)**이지, 프로그램이 아니다. HTTP가 브라우저와 서버 사이의 통신 규약이듯, Wayland는 앱과 컴포지터 사이의 통신 규약이다.

| 구분 | HTTP 세계 | Wayland 세계 |
|------|-----------|-------------|
| 프로토콜 | HTTP | Wayland |
| 서버 | nginx, Apache | Mutter, KWin, Sway |
| 클라이언트 | Chrome, curl | GTK 앱, Qt 앱 |

Wayland 프로토콜이 정의하는 것:
- 앱이 그릴 버퍼를 어떻게 공유할지
- 입력 이벤트를 어떻게 전달할지
- 창 크기/위치 협상을 어떻게 할지

Wayland 자체는 화면을 그리거나 합성하지 않는다. 그건 전부 컴포지터의 몫이다.

---

### 데스크톱 환경 — 1996~1997년

| DE | 시작 | 만든 사람 | 이유 |
|----|------|-----------|------|
| **KDE** | 1996 | Matthias Ettrich | 유닉스 데스크톱이 파편화되어 통일된 환경이 필요 |
| **GNOME** | 1997 | Miguel de Icaza, Federico Mena | KDE가 Qt(당시 독점 라이선스)를 사용해서 완전한 자유 소프트웨어 대안이 필요 |

데스크톱 환경은 아키텍처 레이어가 아니라 **패키징 개념**이다:
- Wayland 컴포지터 (GNOME → Mutter, KDE → KWin)
- 기본 앱들 (파일 관리자, 설정, 터미널 등)
- 상단바, 독/Dash, 앱 런처
- 테마, 아이콘

---

### GNOME Shell과 Mutter — 하나의 프로세스

GNOME의 경우, 컴포지터(Mutter)와 데스크톱 UI(GNOME Shell)가 **하나의 `gnome-shell` 프로세스**로 동작한다.

```
gnome-shell process
+--------------------------------------------+
|  GNOME Shell (JavaScript + C)              |
|    top bar, Dash, Overview, notifications  |
|    St (Shell Toolkit) + CSS for UI         |
|         |                                  |
|         | MetaPlugin interface             |
|         v                                  |
|  libmutter.so                              |
|    Wayland server, window mgmt, input      |
|    Clutter (scene graph) -> Cogl (GL/ES)   |
+--------------------------------------------+
    ^ Wayland protocol       ^ Kernel interface
    |                        |
  Apps (clients)         DRM/KMS, libinput
```

| 역할 | 담당 |
|------|------|
| Wayland 프로토콜 구현 (디스플레이 서버) | **Mutter** (libmutter) |
| 창 합성, 윈도우 관리, GPU 렌더링 | **Mutter** (libmutter) |
| 상단바, Dash, Overview, 알림, 애니메이션 | **GNOME Shell** (JS 플러그인) |

GNOME Shell의 UI(상단바, Dash 등)는 Wayland 프로토콜을 거치지 않는다. Mutter의 Clutter 씬 그래프에 직접 액터로 등록되어, 앱 창들과 함께 한 번에 렌더링된다.

| 조합 | 가능 여부 |
|------|----------|
| Mutter 없이 GNOME Shell | 불가능 — GNOME Shell은 libmutter 위의 플러그인 |
| GNOME Shell 없이 Mutter | 가능 — 창 관리만 되고 상단바/Dash/앱 런처가 없는 상태. 디버깅 용도 |
| 다른 DE가 Mutter 사용 | 가능 — elementary OS의 Gala가 libmutter 사용 |

---

### systemd — 시스템 서비스 관리

커널이 하드웨어를 준비한 뒤, **가장 먼저 실행하는 프로그램**(PID 1)이 systemd다.
부팅 시 네트워크, 오디오, 블루투스, 로그인 화면 등 모든 서비스를 어떤 순서로 시작할지 관리한다.

---

## 전원 ON → 앱 실행까지의 실제 플로우

### 1단계: 전원 → 커널

```
전원 버튼
  ↓
BIOS/UEFI → GRUB → Linux 커널 로드
  ↓
커널:
  - CPU, 메모리 초기화
  - GPU 드라이버 → DRM/KMS 활성화
  - 입력 장치 드라이버 → evdev 활성화
  - PID 1로 systemd 실행
```

### 2단계: systemd → 서비스 시작

```
systemd:
  - D-Bus, NetworkManager, PipeWire 등 시작
  - GDM(GNOME Display Manager) 시작
```

### 3단계: GDM → 로그인

```
GDM → gnome-shell --mode=gdm 실행
  (로그인 전용 모드의 gnome-shell 프로세스)
  ↓
로그인 화면 표시 → 사용자 인증 → 사용자 세션 시작
```

### 4단계: gnome-shell 시작 → 데스크톱 표시

```
GDM → gnome-shell 프로세스 실행
  ↓
libmutter 초기화 (순서대로):
  1. DRM/KMS 연결 (GPU 제어)
  2. EGL/OpenGL 렌더러 초기화
  3. Clutter 컨텍스트 생성 (씬 그래프)
  4. libinput 연결 (입력 장치)
  5. 모니터 설정
  6. Wayland 소켓 생성 (앱 접속용)
  ↓
GNOME Shell JS 로드:
  - 상단바, Dash, 바탕화면 → Clutter 액터로 씬 그래프에 등록
  ↓
씬 그래프 렌더링:
  Clutter → Cogl → OpenGL/ES (EGL) → DRM/KMS → GPU → 모니터
  ↓
데스크톱 표시
```

### 5단계: 앱 실행 (예: 터미널 열기)

```
[1. 입력]

  마우스 클릭 → 커널(evdev)
    → libinput → Mutter 입력 스레드 (ClutterEvent 생성)
    → 메인 스레드 씬 그래프 히트 테스트
    → GNOME Shell JS 핸들러 호출
    (Wayland 프로토콜 거치지 않음 — 씬 그래프 내부 액터이므로)


[2. 프로세스 생성]

  GNOME Shell → 커널에 터미널 실행 요청
  커널: fork() + exec() → 터미널 프로세스 생성


[3. Wayland 연결]

  터미널(GTK):
    → Wayland 소켓(wayland-0)에 접속
    → wl_surface 생성 요청


[4. 버퍼 할당 및 렌더링]

  터미널(클라이언트):
    → 클라이언트가 직접 GPU 버퍼 할당 (DMA-BUF via EGL/Mesa)
    → 그 버퍼에 UI 렌더링
    → wl_surface.attach() + damage() + commit()
    → "다 그렸어"
    (버퍼는 Mutter가 아니라 앱이 만드는 게 핵심)


[5. 합성 및 출력]

  Mutter:
    → 터미널 버퍼를 MetaWindowActor로 씬 그래프에 등록
    → 씬 그래프:
       ├─ MetaBackgroundActor (바탕화면)
       ├─ 상단바 (GNOME Shell 액터)
       ├─ Dash (GNOME Shell 액터)
       └─ MetaWindowActor:터미널 (Wayland 클라이언트)
    → Clutter 전체 합성
    → Cogl → OpenGL/ES (EGL) → DRM/KMS → GPU → 모니터
    ↓
  터미널 창이 화면에 보임
```

---

## 참고 자료

- [Wayland Architecture (공식)](https://wayland.freedesktop.org/architecture.html) — "the compositor is the display server" 명시
- [Wayland Documentation - Chapter 1](https://wayland.freedesktop.org/docs/html/ch01.html) — Wayland 프로토콜 설계 철학
- [Mutter 공식 사이트](https://mutter.gnome.org/) — "Wayland display server and compositor library"
- [GNOME Shell Architecture - GJS Guide](https://gjs.guide/extensions/overview/architecture.html) — GNOME Shell이 Mutter 플러그인인 구조
- [FOSDEM 2012 - Kristian Høgsberg 인터뷰](https://archive.fosdem.org/2012/interview/kristian-hogsberg.html) — Wayland을 만든 이유
- [The Art of (Not) Painting Pixels - GNOME Blog](https://blogs.gnome.org/shell-dev/2020/10/29/the-art-of-not-painting-pixels/) — Mutter 렌더링 최적화
- [Threaded Input Adventures - GNOME Blog](https://blogs.gnome.org/shell-dev/2021/01/21/threaded-input-adventures/) — Mutter 입력 처리 구조
- [Wayland Book - Shared Memory](https://wayland-book.com/surfaces/shared-memory.html) — 클라이언트가 버퍼를 할당하는 구조
- [Wayland Book - DMA-BUF](https://wayland-book.com/surfaces/dmabuf.html) — GPU 버퍼 공유 메커니즘
- [GNOME 48 릴리스 노트](https://release.gnome.org//48/) — 최신 GNOME/Mutter 동향
