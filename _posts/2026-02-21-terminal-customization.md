---
title: "터미널 커스터마이징 — Gruvbox + Starship + Kitty 세팅 가이드"
date: 2026-02-21 16:20:54 +0900
categories: [DevEnv]
tags: [terminal, starship, kitty, gruvbox, nerd-font]
---

Ubuntu의 기본 터미널(GNOME Terminal)은 보라색 Yaru 테마에 밋밋한 프롬프트가 기본이다. 이번 글에서는 터미널 에뮬레이터를 Kitty로 교체하고, 프롬프트를 Starship으로 세팅하고, Gruvbox Dark 테마를 적용하는 과정을 정리한다.

---

## 최종 구성

| 항목 | Before | After |
|------|--------|-------|
| 터미널 에뮬레이터 | GNOME Terminal | **Kitty** |
| 색상 테마 | Yaru (보라색) | **Gruvbox Dark** |
| 프롬프트 | Oh My Bash `font` 테마 | **Starship** (Gruvbox Rainbow) |
| 폰트 | 시스템 기본 | **Hack Nerd Font Mono** |

---

## 1. 터미널 에뮬레이터 교체 — GNOME Terminal에서 Kitty로

### 왜 Kitty인가

GNOME Terminal 대신 Kitty를 선택한 이유:

| | GNOME Terminal | Kitty |
|---|---|---|
| 렌더링 | VTE (CPU 기반) | GPU 가속 |
| 폰트 설정 | 프로필에서 단일 폰트만 지정 | `symbol_map`으로 Unicode 범위별 폰트 지정 가능 |
| Nerd Font 호환 | VTE가 글리프 너비를 잘못 계산하는 경우 있음 | 자체 렌더링 엔진으로 문제없음 |
| 설정 방식 | GUI 또는 `dconf` | 텍스트 파일 (`kitty.conf`) — dotfiles로 관리하기 좋음 |
| 창 분할 | 탭만 지원 | 자체 창 분할 + 탭 지원 |

특히 `symbol_map`이 핵심이다. 영문은 Hack Nerd Font Mono, 한글은 Noto Sans CJK KR처럼 Unicode 범위별로 다른 폰트를 매핑할 수 있다.

### Kitty 설치

```bash
# 홈 디렉토리에 설치 (sudo 불필요)
curl -L https://sw.kovidgoyal.net/kitty/installer.sh | sh /dev/stdin launch=n

# PATH 및 데스크톱 엔트리 설정
ln -sf ~/.local/kitty.app/bin/kitty ~/.local/kitty.app/bin/kitten ~/.local/bin/
cp ~/.local/kitty.app/share/applications/kitty.desktop ~/.local/share/applications/
```

### Kitty 설정 — `~/.config/kitty/kitty.conf`

```conf
# Input Method (Korean)
env GLFW_IM_MODULE=ibus
env XMODIFIERS=@im=ibus

# Font
font_family      Hack Nerd Font Mono
font_size        11.0

# Fallback for Korean/CJK
symbol_map U+1100-U+11FF,U+3130-U+318F,U+AC00-U+D7AF,U+D7B0-U+D7FF Noto Sans CJK KR

# Gruvbox Dark color scheme
foreground #ebdbb2
background #282828

color0  #282828
color1  #cc241d
color2  #98971a
color3  #d79921
color4  #458588
color5  #b16286
color6  #689d6a
color7  #a89984
color8  #928374
color9  #fb4934
color10 #b8bb26
color11 #fabd2f
color12 #83a598
color13 #d3869b
color14 #8ec07c
color15 #ebdbb2

# Cursor
cursor #ebdbb2
cursor_text_color #282828

# Window
window_padding_width 4
confirm_os_window_close 0
```

### 한글 입력 설정 주의사항

Kitty는 GLFW 기반이라 한글 입력(ibus)이 기본적으로 안 된다. `GLFW_IM_MODULE=ibus` 환경변수가 **Kitty 프로세스 시작 전에** 설정되어야 한다.

```bash
# ~/.profile에 추가 (로그인 시 시스템 전체 적용)
export GLFW_IM_MODULE=ibus
```

> `kitty.conf`의 `env` 지시문은 kitty 안의 자식 프로세스(셸)에만 적용된다. kitty 자체의 IM 모듈 로딩에는 영향을 주지 않으므로, 반드시 `~/.profile` 등 로그인 환경에서 설정해야 한다.

---

## 2. 프롬프트 교체 — Oh My Bash에서 Starship으로

### Oh My Bash의 구조

Oh My Bash는 bash 셸을 위한 프레임워크다. 디렉토리 구조를 보면 역할이 나뉘어 있다:

```
~/.oh-my-bash/
├── oh-my-bash.sh    # 핵심 엔트리포인트 — 아래 모듈을 순서대로 로드
├── lib/             # 핵심 라이브러리 (프롬프트 기반, 유틸 함수 등)
├── themes/          # 프롬프트(PS1) 테마 모음
├── plugins/         # 기능 확장 (git, bashmarks 등)
├── completions/     # 탭 자동완성
├── aliases/         # 단축 명령어 모음
└── custom/          # 사용자 커스텀 설정
```

`~/.bashrc`에서 `source "$OSH"/oh-my-bash.sh`를 실행하면, `oh-my-bash.sh`가 다음 순서로 모듈을 로드한다:

```
1. lib/        — 핵심 유틸, 프롬프트 기반 함수
2. plugins/    — ~/.bashrc의 plugins=(...) 목록
3. aliases/    — ~/.bashrc의 aliases=(...) 목록
4. completions/ — ~/.bashrc의 completions=(...) 목록
5. themes/     — OSH_THEME 변수에 지정된 테마
```

### 테마가 프롬프트를 만드는 원리

테마 파일(예: `themes/font/font.theme.sh`)은 `PROMPT_COMMAND`에 함수를 등록한다. bash는 매 명령어 실행 후 `PROMPT_COMMAND`에 등록된 함수를 호출하는데, 이 함수가 `PS1` 변수를 매번 새로 구성한다:

```bash
# themes/font/font.theme.sh (핵심 부분)
function _omb_theme_PROMPT_COMMAND() {
    local RC="$?"  # 이전 명령어의 종료 코드

    # 종료 코드에 따라 화살표 색상 결정
    if [[ ${RC} == 0 ]]; then
        ret_status="${_omb_prompt_bold_green}"
    else
        ret_status="${_omb_prompt_bold_brown}"
    fi

    # PS1 조립: 시간 + 유저@호스트 + 디렉토리 + git정보 + 화살표
    PS1="$(clock_prompt)${hostname} ${_omb_prompt_bold_teal}\W $(scm_prompt_char_info)${ret_status}→ "
}

# PROMPT_COMMAND에 등록
_omb_util_add_prompt_command _omb_theme_PROMPT_COMMAND
```

정리하면 이런 흐름이다:

```
[사용자가 Enter 입력]
  → bash가 PROMPT_COMMAND 실행
    → _omb_theme_PROMPT_COMMAND() 호출
      → PS1 변수 재구성
        → bash가 새 PS1을 화면에 출력
```

### Starship은 이걸 어떻게 교체하는가

Starship도 정확히 같은 메커니즘을 사용한다. `eval "$(starship init bash)"`를 실행하면, Starship이 **자신의 함수를 `PROMPT_COMMAND`에 등록**한다.

```
[Oh My Bash 테마가 하던 일]
PROMPT_COMMAND → _omb_theme_PROMPT_COMMAND() → PS1 설정 (bash 함수)

[Starship이 대신하는 일]
PROMPT_COMMAND → starship_precmd() → PS1 설정 (starship 바이너리 호출)
```

차이점은 PS1을 **만드는 방식**이다:
- Oh My Bash: bash 함수 안에서 문자열을 조립
- Starship: 외부 바이너리(`starship prompt`)를 호출해서 결과를 PS1에 대입

Starship 바이너리는 Rust로 작성되어 있어서 git 상태 파싱, 언어 버전 감지 등을 빠르게 수행하고, `~/.config/starship.toml` 설정에 따라 프롬프트를 렌더링한다.

### 왜 Oh My Bash를 제거하지 않는가

`OSH_THEME=""`로 비워두면, `oh-my-bash.sh`의 테마 로딩 코드가 스킵된다:

```bash
# oh-my-bash.sh 148번줄
if [[ $OSH_THEME ]]; then
  _omb_module_require_theme "$OSH_THEME"  # OSH_THEME이 비어있으면 실행 안 됨
fi
```

테마만 빠지고 나머지(lib, plugins, aliases, completions)는 그대로 로드된다. 즉, Oh My Bash가 제공하는 git 자동완성, bashmarks 플러그인 등은 유지하면서 프롬프트만 Starship으로 교체하는 구조다.

```
~/.bashrc 실행 흐름:
1. source oh-my-bash.sh
   ├── lib/       ✅ 로드
   ├── plugins/   ✅ 로드 (git, bashmarks)
   ├── aliases/   ✅ 로드
   ├── completions/ ✅ 로드 (git, ssh)
   └── themes/    ❌ 스킵 (OSH_THEME="")
2. eval "$(starship init bash)"
   └── PROMPT_COMMAND에 starship 등록 → PS1 담당
```

### Starship 설치

```bash
# ~/.local/bin에 설치 (sudo 불필요)
curl -sS https://starship.rs/install.sh | sh -s -- -y -b ~/.local/bin
```

### Gruvbox Rainbow 프리셋 적용

```bash
~/.local/bin/starship preset gruvbox-rainbow -o ~/.config/starship.toml
```

이 프리셋은 파워라인 스타일로 각 섹션이 주황 → 노랑 → 청록 → 파랑 → 회색으로 이어진다.

### bashrc 설정

```bash
# ~/.bashrc 에서
OSH_THEME=""  # Oh My Bash 테마 비활성화

# 파일 맨 아래에 추가
eval "$(~/.local/bin/starship init bash)"
```

---

## 3. Nerd Font 설치

Starship의 Gruvbox Rainbow 프리셋에는 OS 로고, git 심볼, 파워라인 구분선 같은 특수 아이콘이 포함되어 있다. 이 아이콘은 **Nerd Font**에만 있는 글리프이므로, 일반 폰트에서는 네모(□)로 깨진다.

```bash
mkdir -p ~/.local/share/fonts
cd ~/.local/share/fonts

# Hack Nerd Font 다운로드
curl -fLO "https://github.com/ryanoasis/nerd-fonts/releases/latest/download/Hack.tar.xz"
tar -xf Hack.tar.xz && rm Hack.tar.xz

# 폰트 캐시 갱신
fc-cache -fv ~/.local/share/fonts
```

### Nerd Font 변형 구분

| 변형 | 설명 |
|------|------|
| `Nerd Font` | 아이콘이 원래 너비 (가변폭) |
| `Nerd Font Mono` | 아이콘이 1칸 고정폭 |
| `Nerd Font Propo` | 비례폭(proportional) |
| `NL` 접미사 | No Ligature — 리거처 제거 |

터미널에서는 **Mono** 변형을 쓰는 게 정석이다.

---

## 터미널 에뮬레이터 vs tmux

혼동하기 쉬운 개념이니 정리한다.

| | 터미널 에뮬레이터 | tmux |
|---|---|---|
| 역할 | 화면에 글자를 그리는 **창 프로그램** | 터미널 안에서 **세션/분할** 관리 |
| 예시 | GNOME Terminal, Kitty, Alacritty | tmux, screen |
| 폰트 렌더링 | 담당함 | 관여 안 함 |

Kitty로 바꿔도 tmux는 그 안에서 그대로 사용할 수 있다. Kitty 자체에도 창 분할 기능이 있으니 tmux 없이도 가능하다.

---

## 최종 변경 파일 요약

| 파일 | 변경 내용 |
|------|-----------|
| `~/.bashrc` | Oh My Bash 테마 비활성화 + Starship init 추가 |
| `~/.config/starship.toml` | Gruvbox Rainbow 프리셋 |
| `~/.config/kitty/kitty.conf` | Gruvbox 색상 + Hack Nerd Font + 한글 설정 |
| `~/.local/share/fonts/` | Hack Nerd Font 설치 |
| `~/.profile` | `GLFW_IM_MODULE=ibus` 환경변수 |
