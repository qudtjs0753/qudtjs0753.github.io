---
title: "dotfiles 심볼릭 링크로 관리하기"
date: 2026-02-21 16:00:00 +0900
categories: [DevEnv]
tags: [dotfiles, symlink]
---

개발 환경 설정 파일(dotfiles)을 Git 저장소와 심볼릭 링크로 관리하는 방법을 정리했다.<br/>
`git clone` + `install.sh` 한 줄로 동일한 환경을 구성할 수 있도록 설정해보았다.</br>
github link : [qudtjs0753/dotfiles](https://github.com/qudtjs0753/dotfiles)

---

## dotfiles가 뭔가

리눅스/맥 환경에서 `.`으로 시작하는 설정 파일들을 통칭한다.
홈 디렉토리에 흩어져 있는 이 파일들을 한 곳에 모아 Git으로 버전 관리하면 여러 장점이 있다.

| 장점 | 설명 |
|------|------|
| **이력 관리** | 설정 변경 히스토리를 Git으로 추적 |
| **환경 재현** | 새 PC에서 clone 한 번이면 동일 환경 구축 |
| **공유** | GitHub에 올려서 다른 사람과 설정 공유 가능 |
| **백업** | 원격 저장소가 곧 백업 |

---

## 저장소 구조

홈 디렉토리 경로를 그대로 미러링하는 **flat 구조**를 사용했다.
이렇게 하면 심볼릭 링크 로직이 단순해진다.

```
dotfiles/
├── .bashrc
├── .claude/
│   └── commands/
│       └── blog.md
├── .gitconfig
├── .gitignore
├── .vimrc
├── install.sh
└── README.md
```

관리 대상 파일은 다음과 같다.

| 파일 | 역할 |
|------|------|
| `.bashrc` | NVM, direnv hook, 환경변수 설정 |
| `.gitconfig` | Git 사용자 이름/이메일 |
| `.vimrc` | vim-plug, 한국어 인코딩, 검색 설정 |
| `.claude/commands/blog.md` | Claude Code `/blog` 커맨드 |

---

## install.sh 코드 분석

핵심인 설치 스크립트를 한 줄씩 살펴본다.

### 전체 코드

```bash
#!/usr/bin/env bash
set -euo pipefail

DOTFILES_DIR="$(cd "$(dirname "$0")" && pwd)"

files=(
  .bashrc
  .gitconfig
  .vimrc
  .claude/commands/blog.md
)

for file in "${files[@]}"; do
  src="$DOTFILES_DIR/$file"
  dest="$HOME/$file"

  # Skip if symlink already points to the correct target
  if [ -L "$dest" ] && [ "$(readlink "$dest")" = "$src" ]; then
    echo "OK: $dest -> $src (already linked)"
    continue
  fi

  # Create parent directory if needed
  mkdir -p "$(dirname "$dest")"

  # Back up existing file (not symlink) before replacing
  if [ -e "$dest" ] && [ ! -L "$dest" ]; then
    echo "Backing up $dest -> ${dest}.bak"
    mv "$dest" "${dest}.bak"
  elif [ -L "$dest" ]; then
    echo "Removing old symlink $dest"
    rm "$dest"
  fi

  ln -s "$src" "$dest"
  echo "Linked: $dest -> $src"
done

echo "Done!"
```

### 핵심 포인트별 설명

#### `set -euo pipefail` — 안전 장치

```bash
set -euo pipefail
```

쉘 스크립트의 방어적 코딩 3종 세트다.

| 옵션 | 의미 |
|------|------|
| `-e` | 명령어 하나라도 실패하면 즉시 종료 |
| `-u` | 정의되지 않은 변수 사용 시 에러 |
| `-o pipefail` | 파이프라인 중간 명령어 실패도 감지 |

이 세 줄 없이 스크립트를 작성하면, 오류가 발생해도 조용히 넘어가서 디버깅이 어려워진다.

#### `DOTFILES_DIR` — 절대 경로 계산

```bash
DOTFILES_DIR="$(cd "$(dirname "$0")" && pwd)"
```

스크립트가 **어디에서 실행되든** dotfiles 저장소의 절대 경로를 얻는다.
`$0`은 실행된 스크립트 자기 자신이고, `dirname`으로 디렉토리를 추출한 뒤, `cd + pwd`로 절대 경로로 변환한다.

#### idempotent 설계 — 멱등성

```bash
if [ -L "$dest" ] && [ "$(readlink "$dest")" = "$src" ]; then
  echo "OK: $dest -> $src (already linked)"
  continue
fi
```

이미 올바른 심볼릭 링크가 걸려 있으면 건너뛴다.
스크립트를 여러 번 실행해도 결과가 동일하다 — 이것이 **멱등성(idempotency)**.

실제로 두 번째 실행 시 출력:

```
OK: /home/user/.bashrc -> /home/user/dotfiles/.bashrc (already linked)
OK: /home/user/.gitconfig -> /home/user/dotfiles/.gitconfig (already linked)
OK: /home/user/.vimrc -> /home/user/dotfiles/.vimrc (already linked)
OK: /home/user/.claude/commands/blog.md -> /home/user/dotfiles/.claude/commands/blog.md (already linked)
Done!
```

#### 기존 파일 백업

```bash
if [ -e "$dest" ] && [ ! -L "$dest" ]; then
  echo "Backing up $dest -> ${dest}.bak"
  mv "$dest" "${dest}.bak"
elif [ -L "$dest" ]; then
  echo "Removing old symlink $dest"
  rm "$dest"
fi
```

기존 설정 파일이 **일반 파일**이면 `.bak`으로 백업한 뒤 교체한다.
잘못된 **심볼릭 링크**면 제거 후 새로 건다.

이렇게 하면 기존 설정을 잃어버릴 걱정이 없다.

#### 중첩 디렉토리 처리

```bash
mkdir -p "$(dirname "$dest")"
```

`.claude/commands/blog.md`처럼 중첩된 경로의 경우, 상위 디렉토리가 없을 수 있다.
`mkdir -p`가 이를 자동으로 생성해준다.

---

## 심볼릭 링크의 동작 원리

심볼릭 링크는 **원본 파일을 가리키는 바로가기**다.
`~/.bashrc`를 편집하면 실제로 `~/dotfiles/.bashrc`가 수정된다.

```
~/.bashrc              -> ~/dotfiles/.bashrc
~/.gitconfig           -> ~/dotfiles/.gitconfig
~/.vimrc               -> ~/dotfiles/.vimrc
~/.claude/commands/blog.md -> ~/dotfiles/.claude/commands/blog.md
```

즉, 홈 디렉토리에서 설정을 수정해도 dotfiles 저장소 안의 파일이 바뀌므로 `git diff`로 변경 사항을 확인하고 커밋할 수 있다.

---

## 새 PC에서 환경 세팅하기

```bash
git clone git@github.com:qudtjs0753/dotfiles.git ~/dotfiles
cd ~/dotfiles
chmod +x install.sh
./install.sh
```

이 네 줄이면 끝이다.

---

## 관리 대상에서 제외할 파일

모든 dotfile을 관리할 필요는 없다. 민감하거나 자동 생성되는 파일은 제외해야 한다.

| 제외 대상 | 이유 |
|-----------|------|
| `.bash_history` | 명령어 히스토리 — 민감할 수 있음 |
| `.ssh/` | SSH 키 — 절대 공개 저장소에 올리면 안 됨 |
| `.viminfo` | Vim이 자동 생성 |
| `.claude.json` | 인증 정보 포함 가능 |

`.gitignore`도 잊지 말자.

```
.DS_Store
*.swp
*.bak
```

`*.bak`을 제외해두면 백업 파일이 실수로 커밋되는 것을 방지한다.

---

## 새 설정 파일 추가하는 법

1. dotfiles 저장소에 파일 복사
2. `install.sh`의 `files` 배열에 경로 추가
3. `./install.sh` 실행
4. `git add`, `git commit`, `git push`

예를 들어 `.tmux.conf`를 추가하려면:

```bash
cp ~/.tmux.conf ~/dotfiles/.tmux.conf
```

```bash
files=(
  .bashrc
  .gitconfig
  .vimrc
  .claude/commands/blog.md
  .tmux.conf            # 추가
)
```

---

## 정리

dotfiles 관리의 핵심은 단순함에 있다.

1. 홈 디렉토리 구조를 미러링하는 저장소 만들기
2. 심볼릭 링크로 원본과 연결
3. `install.sh`로 자동화, 멱등성 보장
4. Git으로 버전 관리

복잡한 도구 없이 `bash`, `ln -s`, `git`만으로 충분하다.
