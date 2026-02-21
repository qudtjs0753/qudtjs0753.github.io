---
title: "bash 자동완성을 zsh처럼 — readline과 inputrc 설정"
date: 2026-02-21 17:12:38 +0900
categories: [DevEnv]
tags: [bash, readline, autocomplete]
---

bash의 기본 탭 자동완성은 솔직히 불편하다. 후보가 여러 개면 목록만 출력하고 끝이고, 대소문자를 틀리면 아무것도 안 나온다. zsh나 fish를 쓰는 사람들이 자동완성을 이유로 꼽는 경우가 많은데, 사실 bash도 설정만 바꾸면 비슷한 수준으로 만들 수 있다. 핵심은 **readline**이다.

---

## readline이란

bash가 사용자 입력을 처리할 때 내부적으로 사용하는 라이브러리가 **GNU readline**이다. 터미널에서 글자를 치고, 백스페이스로 지우고, 화살표로 커서를 옮기고, 탭을 눌러 자동완성하는 동작 전부가 readline이 담당한다.

readline의 설정 파일이 `~/.inputrc`다. 시스템 기본은 `/etc/inputrc`에 있고, 사용자별 `~/.inputrc`가 있으면 이쪽이 우선한다.

---

## 기본 상태의 문제점

| 상황 | 기본 동작 | 불편한 점 |
|------|-----------|-----------|
| 후보가 여러 개일 때 Tab | 목록만 출력 | 직접 더 타이핑해야 함 |
| 대소문자가 다를 때 | 완성 실패 | `cd doc` → `Documents` 안 됨 |
| 히스토리 탐색 (↑↓) | 전체 순회 | 원하는 명령 찾기 어려움 |
| 파일 타입 구분 | 없음 | 디렉토리인지 파일인지 모름 |

---

## ~/.inputrc 설정

```bash
$include /etc/inputrc

# Tab을 누르면 후보 목록에서 순환 선택 (zsh처럼)
TAB: menu-complete
"\e[Z": menu-complete-backward

# 대소문자 구분 없이 자동완성
set completion-ignore-case on

# 후보가 여러 개여도 즉시 표시 (두 번 누를 필요 없음)
set show-all-if-ambiguous on

# 순환 선택 시에도 후보 목록 표시
set menu-complete-display-prefix on

# 파일 타입별 색상 표시 (ls 색상과 동일)
set colored-stats on

# 디렉토리에 / 자동 붙이기
set mark-directories on
set mark-symlinked-directories on

# 공통 접두사를 먼저 완성한 뒤 후보 표시
set show-all-if-unmodified on

# 위/아래 화살표로 히스토리 접두사 검색
"\e[A": history-search-backward
"\e[B": history-search-forward
```

새 터미널을 열면 적용된다. 현재 세션에 바로 적용하려면 `bind -f ~/.inputrc`를 실행하면 된다.

---

## 각 설정이 하는 일

### menu-complete — 탭 순환 선택

기본 bash에서 Tab을 누르면 후보 목록이 출력되고 끝이다. `menu-complete`로 바꾸면 Tab을 누를 때마다 후보를 하나씩 순환하며 입력란에 채워준다. zsh의 기본 동작과 같다.

`Shift+Tab`(`\e[Z`)에는 `menu-complete-backward`를 바인딩해서 역방향 순환도 가능하게 한다.

### completion-ignore-case — 대소문자 무시

`cd doc`을 치고 Tab을 누르면 `Documents`로 완성된다. 리눅스 파일시스템은 대소문자를 구분하지만, 자동완성에서는 무시하는 게 편하다.

### show-all-if-ambiguous — 즉시 후보 표시

기본값에서는 후보가 여러 개일 때 Tab을 한 번 누르면 벨만 울리고, **두 번** 눌러야 목록이 나온다. 이 옵션을 켜면 Tab 한 번으로 바로 후보 목록이 표시된다.

### history-search-backward/forward — 접두사 히스토리 검색

기본 ↑↓는 전체 히스토리를 순서대로 순회한다. `history-search-backward`로 바꾸면 **현재 입력한 텍스트로 시작하는 명령어만** 검색한다.

예를 들어 `ssh`를 치고 ↑를 누르면 `ssh`로 시작하는 이전 명령어만 순환한다. `git`을 치고 ↑를 누르면 `git` 명령어만 나온다.

### colored-stats — 파일 타입 색상

자동완성 후보 목록에 `ls --color`와 같은 색상을 적용한다. 디렉토리는 파란색, 실행파일은 초록색 등으로 구분된다.

---

## 적용 전후 비교

| 기능 | Before (기본) | After |
|------|--------------|-------|
| Tab 동작 | 후보 목록만 출력 | **후보 사이를 순환 선택** |
| Shift+Tab | 없음 | **역방향 순환** |
| 대소문자 | 구분함 | **무시** |
| 후보 표시 | Tab 두 번 | **즉시 표시** |
| ↑↓ 화살표 | 전체 히스토리 순회 | **입력한 접두사로 검색** |
| 파일 타입 | 구분 없음 | **색상 표시** |

---

## 설정 확인 방법

현재 적용된 readline 설정은 `bind` 명령어로 확인할 수 있다:

```bash
bind -v            # 모든 readline 변수와 현재 값
bind -p            # 모든 키 바인딩
bind -s            # 사용자 정의 매크로
```

특정 설정이 적용됐는지 확인:

```bash
bind -v | grep completion-ignore-case
# completion-ignore-case on
```

매뉴얼은 `man readline` 또는 `man bash`의 READLINE 섹션에서 볼 수 있다.

---

## zsh에서는 왜 설정 안 해도 되는가

bash와 zsh는 입력 처리 라이브러리가 다르다:

| 셸 | 입력 라이브러리 | 설정 파일 |
|-----|----------------|-----------|
| bash | **GNU readline** | `~/.inputrc` |
| zsh | **ZLE** (Zsh Line Editor) | `~/.zshrc` |

zsh는 readline을 사용하지 않는다. 자체 입력 엔진인 ZLE(Zsh Line Editor)을 쓰고, ZLE의 **기본값**이 이미 우리가 inputrc에서 켠 것들을 포함하고 있다:

- Tab 순환 선택 → zsh 기본
- 대소문자 무시 완성 → zsh 기본 (또는 Oh My Zsh가 켜줌)
- 후보 즉시 표시 → zsh 기본
- 컬러 표시 → zsh 기본

zsh에서 따로 설정한 적이 없어도 자동완성이 잘 됐던 이유다. bash는 readline의 기본값이 보수적이라 직접 켜줘야 한다.

---

## man 페이지란

`man readline`으로 볼 수 있는 문서는 어디서 오는가? man 페이지는 **소프트웨어 패키지에 포함된 문서**다. distro별로 다른 게 아니라, 설치된 패키지 버전에 따라 내용이 결정된다.

```
readline 패키지
├── libreadline.so   ← bash가 링크해서 쓰는 공유 라이브러리
├── man page         ← man readline으로 보는 문서
└── 설정 파일        ← /etc/inputrc, ~/.inputrc
```

`man git`은 git 패키지에, `man bash`는 bash 패키지에, `man readline`은 readline 패키지에 들어있다. Ubuntu든 Fedora든 Arch든 같은 버전의 readline이 설치되어 있으면 같은 man 페이지가 나온다.

---

## 최종 변경 파일

| 파일 | 변경 내용 |
|------|-----------|
| `~/.inputrc` | readline 설정 (탭 순환, 대소문자 무시, 히스토리 검색 등) |
