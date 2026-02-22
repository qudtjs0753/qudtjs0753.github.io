---
title: "Ansible로 PC 환경 세팅 자동화하기"
date: 2026-02-21 21:00:00 +0900
categories: [DevEnv]
tags: [ansible, dotfiles]
---

기존 `install.sh` 기반 dotfiles 관리에서 Ansible playbook으로 전환한 과정을 정리했다.
symlink뿐 아니라 패키지 설치, 프로그램 설치, GNOME 설정, pre-commit hook까지 한 파일로 자동화한다.

> symlink 원리와 저장소 구조는 [이전 글](/posts/dotfiles-symlink-management/)에서 다뤘으므로 생략한다.

---

## 최종 결과

```bash
ansible-playbook setup.yml --ask-become-pass
```

이 한 줄이면 새 PC 환경 세팅이 끝난다.

playbook이 처리하는 작업은 다음과 같다.

| 단계 | 작업 | 수동 시 필요한 명령 |
|------|------|---------------------|
| 1 | apt 패키지 설치 | `sudo apt install ...` |
| 2 | 프로그램 설치 (VS Code, Claude Code, Starship) | `snap install`, `curl \| sh` |
| 3 | dotfile symlink 생성 | `ln -s` 반복 |
| 4 | GNOME 확장 활성화 | `gnome-extensions enable ...` |
| 5 | GNOME dconf 설정 로드 | `dconf load ...` |
| 6 | pre-commit hook 설치 | `git config core.hooksPath hooks` |

---

## Ansible이란

2012년 Michael DeHaan이 만들었다. 당시 서버 설정 자동화 도구로 Puppet, Chef가 있었지만, 둘 다 관리 대상 서버에 에이전트를 설치해야 했고 설정이 복잡했다. 여기서 에이전트란 관리 대상 서버에 상주하면서 중앙 서버의 명령을 받아 실행하는 데몬 프로세스다. 서버가 100대면 100대 전부에 에이전트를 깔고 관리해야 한다. Ansible은 에이전트 없이 SSH만으로 동작하고, YAML로 작성하는 단순한 구조를 택했다. 2015년에 Red Hat이 인수했고, 현재는 사실상 업계 표준이다.

이름은 SF 소설에서 따왔다. Ursula K. Le Guin의 작품에 나오는 "ansible"은 빛보다 빠른 초광속 통신 장치인데, 수많은 서버에 명령을 동시에 전달하는 도구의 성격과 맞아떨어진다.

원래 원격 서버 수십 대를 한꺼번에 설정할 때 쓰지만, `hosts: localhost`로 지정하면 내 PC에도 사용할 수 있다.

설치는 한 줄이면 된다.

```bash
sudo apt install ansible-core
```

### 핵심 개념

| 개념 | 설명 | bash 대응 |
|------|------|-----------|
| **playbook** | YAML로 작성하는 작업 목록. "이 순서대로 실행해라"라는 선언적 명세 | 쉘 스크립트 파일 |
| **task** | 하나의 작업 단위. `name`으로 설명하고 모듈로 실행 | 스크립트 안의 명령 한 줄 |
| **module** | Ansible이 제공하는 기능 단위 (`apt`, `file`, `copy`, `shell` 등) | `apt install`, `ln -s`, `cp` |
| **loop** | 리스트를 순회하며 태스크 반복 | `for item in "${list[@]}"` |
| **when** | 조건부 실행 | `if [ 조건 ]` |
| **become: true** | sudo 권한 상승 | `sudo` |
| **register** | 태스크 결과를 변수에 저장해서 다음 태스크에서 참조 | `result=$(명령)` |

이 개념들이 아래 코드에서 어떻게 쓰이는지 살펴보자.

---

## setup.yml 핵심 분석

전체 코드를 나열하는 대신, 핵심 패턴 위주로 설명한다.

### 변수 정의

```yaml
vars:
  dotfiles_dir: "{% raw %}{{ playbook_dir }}{% endraw %}"
  home_dir: "{% raw %}{{ lookup('env', 'HOME') }}{% endraw %}"
  dotfiles:
    - .bashrc
    - .gitconfig
    - .vimrc
    - .inputrc
    - .config/starship.toml
    - .claude/commands/blog.md
    - .claude/settings.json
    - .tmux.conf
```

`playbook_dir`은 Ansible 빌트인 변수로, `setup.yml`이 위치한 디렉토리의 절대 경로다.
쉘 스크립트에서 `DOTFILES_DIR="$(cd "$(dirname "$0")" && pwd)"`로 했던 것과 같은 역할이다.

{% raw %}`{{ }}`{% endraw %} 문법은 Ansible 자체 문법이 아니라 Python 템플릿 엔진인 Jinja2의 문법이다. Ansible이 Jinja2를 내장해서 사용한다. {% raw %}`{{ 변수명 }}`{% endraw %}은 값을 출력하고, {% raw %}`{% 제어문 %}`{% endraw %}은 조건·반복을 처리한다. 이 글에서는 {% raw %}`{{ }}`{% endraw %}만 쓰인다.

### apt 패키지 설치

```yaml
- name: Install required apt packages
  become: true
  ansible.builtin.apt:
    name:
      - gnome-shell-extension-manager
      - gnome-tweaks
      - dconf-cli
    state: present
    update_cache: true
```

가장 단순한 형태의 태스크다.
`become: true`로 sudo 권한을 얻고, `apt` 모듈로 패키지를 설치한다.
`state: present`는 "설치되어 있어야 한다"는 선언이다 — 이미 설치되어 있으면 건너뛴다.
`update_cache: true`는 설치 전에 `sudo apt update`를 자동 실행한다.

### 프로그램 설치 — snap, curl installer

snap이나 curl installer로 설치하는 프로그램들도 Ansible로 자동화할 수 있다.

#### VS Code — snap

```yaml
- name: Check if snapd is available
  ansible.builtin.stat:
    path: /run/snapd.socket
  register: snapd_check

- name: Install VS Code via snap
  become: true
  ansible.builtin.command:
    cmd: snap install code --classic
  when: snapd_check.stat.exists
```

먼저 `stat`으로 snapd 소켓이 존재하는지 확인한다.
Docker 컨테이너처럼 snapd가 없는 환경에서는 `when` 조건에 의해 자동으로 skip된다.

#### Claude Code, Starship — curl installer + creates

```yaml
- name: Install Claude Code
  ansible.builtin.shell:
    cmd: curl -fsSL https://claude.ai/install.sh | bash
    creates: "{% raw %}{{ home_dir }}{% endraw %}/.local/bin/claude"

- name: Install Starship prompt
  ansible.builtin.shell:
    cmd: curl -sS https://starship.rs/install.sh | sh -s -- -y
    creates: "{% raw %}{{ home_dir }}{% endraw %}/.local/bin/starship"
```

둘 다 공식 설치 스크립트를 curl로 받아 실행하는 패턴이다.
핵심은 `creates` 파라미터다 — 지정한 경로에 파일이 이미 존재하면 태스크를 건너뛴다.
이것만으로 멱등성이 보장된다.

### loop + when — 디렉토리 생성

프로그램 설치가 기본 설정 파일을 생성할 수 있으므로, dotfile symlink는 프로그램 설치 이후에 만든다. 이렇게 하면 우리 설정이 기본 설정을 덮어쓴다.

```yaml
- name: Ensure parent directories exist
  ansible.builtin.file:
    path: "{% raw %}{{ home_dir }}/{{ item | dirname }}{% endraw %}"
    state: directory
    mode: "0755"
  loop: "{% raw %}{{ dotfiles }}{% endraw %}"
  when: (item | dirname) != ""
```

`dotfiles` 리스트를 `loop`로 순회하면서, 각 항목의 부모 디렉토리를 생성한다.
`.config/starship.toml`은 `~/.config/` 디렉토리가 필요하지만, `.bashrc`는 홈 디렉토리에 바로 위치하므로 `when` 조건으로 건너뛴다.

bash로 쓰면 이런 코드다.

```bash
for file in "${dotfiles[@]}"; do
  dir=$(dirname "$file")
  if [ -n "$dir" ]; then
    mkdir -p "$HOME/$dir"
  fi
done
```

### register — stat 결과를 다음 태스크로 전달

```yaml
- name: Check existing dotfiles
  ansible.builtin.stat:
    path: "{% raw %}{{ home_dir }}/{{ item }}{% endraw %}"
  loop: "{% raw %}{{ dotfiles }}{% endraw %}"
  register: dotfile_stats

- name: Back up existing files (not symlinks)
  ansible.builtin.command:
    cmd: mv "{% raw %}{{ home_dir }}/{{ item.item }}{% endraw %}" "{% raw %}{{ home_dir }}/{{ item.item }}{% endraw %}.bak"
  loop: "{% raw %}{{ dotfile_stats.results }}{% endraw %}"
  when: item.stat.exists and not item.stat.islnk
```

`stat` 모듈로 각 파일의 상태(존재 여부, symlink 여부)를 조사하고, `register`로 결과를 `dotfile_stats`에 저장한다.
다음 태스크에서 `dotfile_stats.results`를 `loop`하면서, 일반 파일이 존재하면 `.bak`으로 백업한다.
이미 symlink인 경우에는 `when` 조건에서 걸러진다.

이 2단계를 거친 후, symlink를 생성한다.

```yaml
- name: Create dotfile symlinks
  ansible.builtin.file:
    src: "{% raw %}{{ dotfiles_dir }}/{{ item }}{% endraw %}"
    dest: "{% raw %}{{ home_dir }}/{{ item }}{% endraw %}"
    state: link
    force: true
  loop: "{% raw %}{{ dotfiles }}{% endraw %}"
```

`state: link` + `force: true`로 심볼릭 링크를 생성한다. 기존 symlink가 있어도 덮어쓰므로 몇 번을 실행해도 결과가 같다(멱등성).

---

## GNOME 확장 관리

### 확장 목록 추출

현재 활성화된 GNOME 확장 목록은 다음 명령으로 추출할 수 있다.

```bash
gnome-extensions list --enabled
```

이 결과를 `setup.yml`의 `gnome_extensions` 변수에 넣으면 된다.

```yaml
gnome_extensions:
  - tiling-assistant@ubuntu.com
  - search-light@icedman.github.com
  - ubuntu-dock@ubuntu.com
  - ubuntu-appindicators@ubuntu.com
```

### 확장 활성화 + dconf 설정 로드

```yaml
- name: Enable GNOME extensions
  ansible.builtin.command:
    cmd: gnome-extensions enable "{% raw %}{{ item }}{% endraw %}"
  loop: "{% raw %}{{ gnome_extensions }}{% endraw %}"
  failed_when: false
  tags: [gnome]

- name: Load GNOME dconf settings
  ansible.builtin.shell:
    cmd: dconf load /org/gnome/ < "{% raw %}{{ dotfiles_dir }}{% endraw %}/gnome-settings.dconf"
  tags: [gnome]
```

`failed_when: false`로 설정해서 확장이 아직 설치되지 않은 경우에도 playbook이 중단되지 않는다.
확장 자체는 GNOME Extensions 앱이나 브라우저에서 미리 설치해둬야 한다.

dconf 설정은 키보드 단축키, 테마, 창 동작 등 GNOME 데스크톱 설정을 `gnome-settings.dconf` 파일에서 일괄 로드한다.
설정을 내보낼 때는 `dconf dump /org/gnome/ > gnome-settings.dconf`를 사용한다.

`tags: [gnome]`을 붙여두면, GUI가 없는 환경(Docker 컨테이너 등)에서 `--skip-tags gnome`으로 이 태스크들만 건너뛸 수 있다.

---

## install.sh → Ansible 변환 포인트

| bash | Ansible | 설명 |
|------|---------|------|
| `files=(.bashrc .vimrc ...)` | `dotfiles:` YAML 리스트 | 배열 → 변수 |
| `for file in "${files[@]}"` | `loop: "{% raw %}{{ dotfiles }}{% endraw %}"` | 반복문 |
| `if [ -e "$dest" ] && [ ! -L "$dest" ]` | `when: item.stat.exists and not item.stat.islnk` | 조건 분기 |
| `sudo apt install ...` | `become: true` + `apt` 모듈 | 권한 상승 |
| `DOTFILES_DIR="$(cd ... && pwd)"` | `playbook_dir` 빌트인 변수 | 경로 계산 |

쉘 스크립트에서 직접 구현해야 했던 멱등성, 에러 처리, 조건 분기를 Ansible이 모듈 수준에서 제공한다.

---

## pre-commit hook으로 동기화 보장

dotfiles 저장소에 파일을 추가하고 `setup.yml`의 `dotfiles` 목록에 넣는 걸 깜빡하면, 새 PC에서 해당 파일의 symlink가 생성되지 않는다.
이를 방지하기 위해 pre-commit hook을 사용한다.

hook은 두 가지를 검사한다.

| 검사 | 의미 | 메시지 |
|------|------|--------|
| git 추적 파일 ∉ `dotfiles` 목록 | 파일을 추가했는데 목록 갱신을 빠뜨림 | `+ 파일명` |
| `dotfiles` 목록 ∉ 실제 파일 | 파일을 삭제했는데 목록에서 안 뺌 | `- 파일명` |

어느 쪽이든 불일치가 있으면 커밋이 차단된다.
`README.md`, `setup.yml` 같은 인프라 파일은 symlink 대상이 아니므로 hook의 `EXCLUDE` 배열로 검사에서 제외한다.

### hook 관리: core.hooksPath

Git hook을 팀(또는 저장소)과 공유하는 방법은 여러 가지다.

| 방법 | 특징 | 적합한 경우 |
|------|------|-------------|
| [Husky](https://typicode.github.io/husky/) | npm 패키지. `package.json` 기반 | Node.js 프로젝트 |
| [Lefthook](https://github.com/evilmartians/lefthook) | Go 바이너리. YAML 설정, 빠름 | 다국어 프로젝트 |
| [pre-commit (Python)](https://pre-commit.com/) | pip 패키지. 플러그인 생태계 | Python 프로젝트, linter 조합 |
| **`core.hooksPath`** | Git 2.9+ 내장. 별도 설치 없음 | 특정 생태계에 속하지 않는 저장소 |

dotfiles 저장소는 Node.js도 Python도 아니다. 패키지 매니저가 없는 환경에서 외부 도구를 설치하는 것은 오버엔지니어링이다.
`core.hooksPath`는 Git 자체 기능이므로 별도 의존성이 없다.

```bash
git config core.hooksPath hooks
```

이 한 줄이면 Git이 `.git/hooks/` 대신 `hooks/` 디렉토리를 직접 참조한다.
hook 파일을 수정하면 즉시 반영되므로, 복사를 깜빡하는 문제가 원천적으로 사라진다.

setup.yml의 마지막 태스크도 이에 맞춰 `copy`에서 `git config`로 변경했다.

```yaml
- name: Set git hooksPath
  ansible.builtin.command:
    cmd: git config core.hooksPath hooks
    chdir: "{% raw %}{{ dotfiles_dir }}{% endraw %}"
```

---

## Docker로 playbook 검증

playbook을 수정한 뒤, 실제로 새 PC에서 잘 동작하는지 어떻게 확인할까?

Ansible에는 `ping` 모듈이 있다.

```bash
ansible localhost -m ping
```

이 명령은 Ansible이 대상 호스트에 연결할 수 있는지만 확인한다. 원격 서버 수십 대를 관리할 때는 의미 있지만, `hosts: localhost`인 우리 playbook에서는 항상 성공하므로 검증이 되지 않는다.

우리가 확인해야 하는 건 "연결 가능한가"가 아니라 **"playbook을 실행했을 때 원하는 상태가 만들어지는가"**다.
Docker 컨테이너로 클린 Ubuntu 환경을 만들고 playbook을 실행하면 된다.

### Dockerfile

저장소에 포함된 `Dockerfile`로 검증한다.

```dockerfile
FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y ansible-core git curl && rm -rf /var/lib/apt/lists/*

WORKDIR /dotfiles
COPY . /dotfiles/

RUN git init && \
    ansible-playbook setup.yml --skip-tags gnome
```

`--skip-tags gnome`으로 GUI가 필요한 GNOME 태스크는 건너뛴다.
snapd가 없으므로 VS Code 설치도 `when` 조건에 의해 자동 skip된다.

### 실행

```bash
docker build --no-cache -t dotfiles-test .
```

빌드가 성공하면 playbook이 에러 없이 완료된 것이다.
`--no-cache`를 붙여야 매번 처음부터 실행한다.

### 검증 항목

| 검증 대상 | 확인 방법 |
|-----------|-----------|
| apt 패키지 | `dpkg -l gnome-tweaks dconf-cli gnome-shell-extension-manager` |
| Starship 설치 | `test -f ~/.local/bin/starship` |
| dotfile symlinks | `test -L ~/.bashrc` 등 각 파일이 symlink인지 확인 |
| symlink 타겟 | `readlink ~/.bashrc` → dotfiles 디렉토리 내 파일을 가리키는지 |
| pre-commit hook | `git config core.hooksPath` → `hooks` |
| 실행 순서 | ansible 출력에서 apt → programs → symlinks → hook 순서인지 |

컨테이너에 들어가서 직접 확인할 수도 있다.

```bash
docker run --rm -it dotfiles-test bash
```

검증이 끝나면 이미지를 삭제한다.

```bash
docker rmi dotfiles-test
```

---

## 정리

| 관점 | install.sh | Ansible playbook |
|------|-----------|-----------------|
| symlink 관리 | O | O |
| 패키지 설치 | X (별도 수동) | O (`apt` 모듈) |
| 프로그램 설치 | X (별도 수동) | O (`snap`, `curl` installer) |
| GNOME 설정 | X (별도 수동) | O (`dconf load`) |
| 확장 활성화 | X (별도 수동) | O (`gnome-extensions enable`) |
| hook 설치 | X (별도 수동) | O (`core.hooksPath`) |
| 멱등성 | 직접 구현 | 모듈이 보장 |

`install.sh`는 symlink 하나에 집중하는 데 좋았지만, 환경 전체를 재현하려면 결국 여러 스크립트와 수동 작업이 필요했다.
Ansible playbook 하나로 통합하면서 `ansible-playbook setup.yml --ask-become-pass` 한 줄이면 되는 구조가 됐다.
