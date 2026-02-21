---
title: "터미널 커스터마이징 — Gruvbox + Starship + Nerd Font 세팅 가이드"
date: 2026-02-21 16:20:54 +0900
categories: [DevEnv]
tags: [terminal, starship, gruvbox, nerd-font]
---

Ubuntu의 기본 터미널(GNOME Terminal)은 보라색 Yaru 테마에 밋밋한 프롬프트가 기본이다. 이번 글에서는 터미널 색상, 프롬프트, 폰트를 커스터마이징한 과정을 정리한다.

---

## 최종 구성

| 항목 | Before | After |
|------|--------|-------|
| 색상 테마 | Yaru (보라색) | **Gruvbox Dark** |
| 프롬프트 | Oh My Bash `font` 테마 | **Starship** (Gruvbox Rainbow) |
| 폰트 | 시스템 기본 | **Hack Nerd Font Mono** |

---

## 1. 터미널 색상 변경 — Gruvbox Dark

```bash
PROFILE="$(gsettings get org.gnome.Terminal.ProfilesList default | tr -d \')"

dconf write /org/gnome/terminal/legacy/profiles:/:$PROFILE/use-theme-colors "false"
dconf write /org/gnome/terminal/legacy/profiles:/:$PROFILE/background-color "'#282828'"
dconf write /org/gnome/terminal/legacy/profiles:/:$PROFILE/foreground-color "'#ebdbb2'"
dconf write /org/gnome/terminal/legacy/profiles:/:$PROFILE/bold-color-same-as-fg "true"

dconf write /org/gnome/terminal/legacy/profiles:/:$PROFILE/palette \
  "['#282828', '#cc241d', '#98971a', '#d79921', '#458588', '#b16286', '#689d6a', '#a89984', '#928374', '#fb4934', '#b8bb26', '#fabd2f', '#83a598', '#d3869b', '#8ec07c', '#ebdbb2']"
```

Gruvbox는 따뜻한 갈색-검정 배경에 크림색 글자가 특징인 레트로 다크 테마다. `dconf`는 GNOME 데스크톱의 설정 저장소로, 윈도우의 레지스트리와 비슷한 개념이다. GUI에서 환경설정을 바꾸는 것과 동일한 결과를 CLI로 수행한다.

---

## 2. Nerd Font 설치

```bash
mkdir -p ~/.local/share/fonts
cd ~/.local/share/fonts

# Hack Nerd Font 다운로드
curl -fLO "https://github.com/ryanoasis/nerd-fonts/releases/latest/download/Hack.tar.xz"
tar -xf Hack.tar.xz && rm Hack.tar.xz

# 폰트 캐시 갱신
fc-cache -fv ~/.local/share/fonts
```

GNOME Terminal에 폰트 적용:

```bash
PROFILE="$(gsettings get org.gnome.Terminal.ProfilesList default | tr -d \')"
dconf write /org/gnome/terminal/legacy/profiles:/:$PROFILE/use-system-font "false"
dconf write /org/gnome/terminal/legacy/profiles:/:$PROFILE/font "'Hack Nerd Font Mono 11'"
```

> **주의**: `fc-cache -fv`로 폰트 캐시를 반드시 갱신해야 한다. 캐시가 갱신되지 않으면 GNOME Terminal에서 글자 간격이 깨질 수 있다.

### 왜 Nerd Font가 필요한가

Starship의 Gruvbox Rainbow 프리셋에는 OS 로고, git 심볼, 파워라인 구분선 같은 특수 아이콘이 포함되어 있다. 이 아이콘은 **Nerd Font**에만 있는 글리프이므로, 일반 폰트에서는 네모(□)로 깨진다. Starship을 설치하기 전에 먼저 폰트를 세팅해야 한다.

### Nerd Font 변형 구분

| 변형 | 설명 |
|------|------|
| `Nerd Font` | 아이콘이 원래 너비 (가변폭) |
| `Nerd Font Mono` | 아이콘이 1칸 고정폭 |
| `Nerd Font Propo` | 비례폭(proportional) |
| `NL` 접미사 | No Ligature — 리거처 제거 |

터미널에서는 **Mono** 변형을 쓰는 게 정석이다.

### 폰트 캐시란

`.bashrc`는 텍스트 파일이라 bash가 매번 직접 읽는다. 캐시가 없으므로 수정하면 즉시 반영된다.

폰트는 다르다. GNOME Terminal이 폰트를 쓸 때 폰트 파일을 직접 스캔하지 않고 **fontconfig** 라이브러리의 **캐시 파일**(바이너리 인덱스)을 통해 접근한다:

```
[폰트 설치 시]
~/.local/share/fonts/에 .ttf 파일 복사
  → 파일은 있지만, fontconfig 캐시에는 아직 없음
  → GNOME Terminal이 폰트 목록을 캐시에서 읽음 → 새 폰트를 모름

[fc-cache 실행 후]
fc-cache -fv
  → fontconfig가 폰트 디렉토리를 스캔
  → 캐시 파일 재생성 (~/.cache/fontconfig/)
  → GNOME Terminal이 캐시를 다시 읽음 → 새 폰트 인식
```

폰트 디렉토리에는 수백~수천 개의 바이너리 파일이 있을 수 있으므로, 매번 스캔하면 느려서 캐시를 쓴다.

---

## 3. 프롬프트 교체 — Oh My Bash에서 Starship으로

### 설치 및 설정

```bash
# Starship 설치 (~/.local/bin에, sudo 불필요)
curl -sS https://starship.rs/install.sh | sh -s -- -y -b ~/.local/bin

# Gruvbox Rainbow 프리셋 적용
~/.local/bin/starship preset gruvbox-rainbow -o ~/.config/starship.toml
```

`~/.bashrc` 수정:

```bash
# Oh My Bash 테마 비활성화
OSH_THEME=""

# 파일 맨 아래에 추가
eval "$(~/.local/bin/starship init bash)"
```

이 프리셋은 파워라인 스타일로 각 섹션이 주황 → 노랑 → 청록 → 파랑 → 회색으로 이어진다.

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

### eval "$(starship init bash)"는 무엇을 하는가

이 한 줄은 2단계에 걸쳐 동작한다.

**1단계**: `$()`는 명령어 치환이다. 안에 있는 명령어를 실행하고 출력 결과를 문자열로 바꿔치기한다.

```bash
starship init bash
# 출력: eval -- "$(starship init bash --print-full-init)"
```

**2단계**: `eval`은 받은 문자열을 bash 코드로 실행한다. 위 출력이 다시 실행되면서 `--print-full-init`이 호출되고, 실제 초기화 스크립트가 출력된다. `eval`이 이 스크립트를 현재 셸에서 실행한다.

```
eval "$(starship init bash)"
│      └─ 1. starship init bash 실행
│         → 출력: 'eval -- "$(starship init bash --print-full-init)"'
│
└─ 2. eval이 그 출력을 코드로 실행
      → starship init bash --print-full-init 실행
      → 출력: starship_precmd() 함수 정의 + PROMPT_COMMAND 등록 스크립트
      → eval이 그 스크립트를 현재 셸에서 실행
```

초기화 스크립트에서 PROMPT_COMMAND를 등록하는 핵심 부분:

```bash
# PROMPT_COMMAND가 비어있으면 바로 등록
if [[ -z "${PROMPT_COMMAND-}" ]]; then
    PROMPT_COMMAND="starship_precmd"

# 이미 뭔가 있으면 (Oh My Bash 등) 백업 후 교체
elif [[ "$PROMPT_COMMAND" != *"starship_precmd"* ]]; then
    STARSHIP_PROMPT_COMMAND="$PROMPT_COMMAND"
    PROMPT_COMMAND="starship_precmd"
fi
```

기존 `PROMPT_COMMAND`를 덮어쓰는 게 아니라 `STARSHIP_PROMPT_COMMAND`에 백업해두고, `starship_precmd` 안에서 실행해준다:

```bash
starship_precmd() {
    STARSHIP_CMD_STATUS=$?  # 이전 명령어 종료 코드 저장

    # 백업해둔 기존 PROMPT_COMMAND 실행
    if [[ -n "${STARSHIP_PROMPT_COMMAND-}" ]]; then
        eval "$STARSHIP_PROMPT_COMMAND"
    fi

    # starship 바이너리를 호출해서 PS1 생성
    PS1="$(starship prompt "${ARGS[@]}")"
}
```

`echo`로 바꾸면 실행 없이 출력만 볼 수 있다:

```bash
echo "$(starship init bash)"           # 1단계 출력
starship init bash --print-full-init   # 2단계 출력 (전체 스크립트)
```

### bash는 PROMPT_COMMAND를 언제 실행하는가

bash 5.2 소스코드의 `eval.c`를 보면 두 함수가 핵심이다.

먼저 `PROMPT_COMMAND` 변수를 찾아서 실행하는 함수:

```c
// eval.c:290
static void
execute_prompt_command ()
{
  pcv = find_variable ("PROMPT_COMMAND");  // 변수 찾기
  if (pcv == 0 || var_isset (pcv) == 0)
    return;                                 // 없으면 스킵

  command_to_execute = value_cell (pcv);    // 값 꺼내기
  execute_variable_command (command_to_execute, "PROMPT_COMMAND");  // 실행
}
```

그리고 이 함수를 호출하는 쪽:

```c
// eval.c:323
int
parse_command ()
{
  // 인터랙티브 셸이고, 프롬프트를 출력할 조건이면
  if (interactive && bash_input.type != st_string && parser_expanding_alias() == 0)
    {
      execute_prompt_command ();   // PROMPT_COMMAND 실행
    }

  r = yyparse ();   // 사용자 입력 대기 & 파싱
  return (r);
}
```

`parse_command()`는 bash의 메인 루프에서 다음 명령을 읽을 때마다 호출된다. 흐름을 정리하면:

```
parse_command()                        ← 매 명령 읽기 전 호출
  → execute_prompt_command()           ← PROMPT_COMMAND 변수를 찾아 실행
    → find_variable("PROMPT_COMMAND")
    → execute_variable_command(...)    ← starship_precmd() 등이 여기서 실행
  → yyparse()                          ← 프롬프트 표시, 사용자 입력 파싱
```

매 프롬프트마다 `PROMPT_COMMAND`가 실행되므로, git 브랜치를 바꾸거나 디렉토리를 이동하면 다음 프롬프트에 바로 반영되는 것이다.

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

---

## 최종 변경 파일 요약

| 파일 | 변경 내용 |
|------|-----------|
| `~/.bashrc` | Oh My Bash 테마 비활성화 + Starship init 추가 |
| `~/.config/starship.toml` | Gruvbox Rainbow 프리셋 |
| `~/.local/share/fonts/` | Hack Nerd Font 설치 |

---

## 최종 결과

![터미널 커스터마이징 완료](/assets/img/posts/2026-02-21-terminal-customization/result.png)

Gruvbox Dark 색상 테마, Starship Gruvbox Rainbow 프롬프트, Hack Nerd Font Mono가 적용된 모습이다.
