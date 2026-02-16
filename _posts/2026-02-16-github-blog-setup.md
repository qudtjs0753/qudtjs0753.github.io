---
title: "GitHub 블로그 만들기"
date: 2026-02-16 15:00:00 +0900
categories: [Blogging, Tutorial]
tags: [github, jekyll, chirpy, ruby, blog]
---

Claude writing. for test.
Jekyll + Chirpy 테마를 사용하여 GitHub Pages 블로그를 만드는 전체 과정을 정리한 글이다.
각 단계에서 무엇을 설치했고, 왜 필요한지, 어떤 역할을 하는지 상세하게 기록했다.

---

## Step 1: Ruby 환경 설치

### 설치한 것들 요약

| 설치 항목 | 무엇인가 | 왜 설치했나 |
|-----------|---------|------------|
| 시스템 빌드 패키지 | C 라이브러리 및 컴파일 도구 | Ruby를 소스에서 컴파일하기 위해 |
| rbenv | Ruby 버전 관리 도구 | 원하는 Ruby 버전을 설치/관리하기 위해 |
| Ruby 3.3.6 | 프로그래밍 언어 | Jekyll이 Ruby로 만들어졌기 때문에 |

### 1. 시스템 빌드 패키지

Ruby는 C 언어로 작성되어 있어서, 소스 코드에서 컴파일할 때 여러 시스템 라이브러리가 필요하다.

#### 설치한 패키지들

| 패키지 | 역할 | 언제 사용되나 |
|--------|------|-------------|
| **build-essential** | C/C++ 컴파일러(gcc, g++)와 make 도구 | Ruby 소스 코드를 컴파일할 때 |
| **autoconf** | 빌드 설정 스크립트를 자동 생성 | Ruby의 `./configure` 단계에서 |
| **bison** | 파서(parser) 생성기 | Ruby가 코드를 해석하는 파서를 빌드할 때 |
| **libssl-dev** | OpenSSL 암호화 라이브러리의 개발 헤더 | `gem install`로 패키지를 HTTPS로 다운로드할 때 |
| **libreadline-dev** | 터미널 입력 편집 라이브러리 | Ruby의 `irb` (대화형 콘솔)에서 방향키, 히스토리 사용 시 |
| **zlib1g-dev** | 데이터 압축/해제 라이브러리 | gem 파일(압축됨)을 설치할 때 |
| **libyaml-dev** | YAML 파싱 라이브러리 | Jekyll의 `_config.yml`, 포스트의 Front Matter를 읽을 때 |
| **libffi-dev** | Foreign Function Interface 라이브러리 | Ruby에서 C 함수를 호출할 때 (다양한 gem에서 사용) |
| **libgdbm-dev** | GNU 키-값 데이터베이스 라이브러리 | Ruby의 내장 dbm 모듈에서 |
| **libncurses-dev** | 터미널 UI 제어 라이브러리 | 터미널에서 색상, 커서 이동 처리 |

> `-dev`가 붙은 패키지는 **개발용 헤더 파일**을 포함한다. 런타임 라이브러리는 이미 시스템에 있을 수 있지만, 소스 컴파일 시에는 헤더 파일(`.h`)이 필요하다.
{: .prompt-info }

#### 설치 명령어

```bash
sudo apt-get update
sudo apt-get install -y git curl libssl-dev libreadline-dev zlib1g-dev \
  autoconf bison build-essential libyaml-dev libffi-dev libgdbm-dev \
  libncurses5-dev libsqlite3-dev
```

### 2. rbenv

#### 무엇인가
Ruby 버전 관리 도구. Python의 `pyenv`, Node.js의 `nvm`에 해당한다.

#### 왜 설치했나
- 시스템 기본 Ruby를 건드리지 않고, 원하는 버전(3.3.6)을 독립적으로 설치하기 위해
- 프로젝트마다 다른 Ruby 버전을 사용할 수 있게 하기 위해

#### 어떻게 동작하나
1. `~/.rbenv/versions/` 아래에 각 Ruby 버전을 설치한다
2. `rbenv global 3.3.6` 명령으로 기본 버전을 지정한다
3. 쉘의 `PATH`를 조작하여 `ruby` 명령이 rbenv가 관리하는 버전을 가리키게 한다

#### 설치 경로
```
~/.rbenv/
├── bin/           # rbenv 실행 파일
├── versions/      # 설치된 Ruby 버전들
│   └── 3.3.6/    # Ruby 3.3.6 설치 경로
├── shims/         # ruby, gem 등의 심링크 (PATH에 등록됨)
└── plugins/
    └── ruby-build/  # Ruby를 빌드하는 플러그인
```

#### 설치 명령어

```bash
# rbenv 설치
curl -fsSL https://github.com/rbenv/rbenv-installer/raw/HEAD/bin/rbenv-installer | bash

# 쉘에 등록
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
source ~/.bashrc
```

### 3. Ruby 3.3.6

#### 무엇인가
프로그래밍 언어. 웹 프레임워크(Ruby on Rails), 정적 사이트 생성기(Jekyll) 등의 생태계를 가지고 있다.

#### 왜 설치했나
**Jekyll이 Ruby로 만들어졌기 때문이다.** Jekyll을 설치하고 실행하려면 Ruby가 반드시 필요하다.

#### 전체 의존 관계
```
Ruby (프로그래밍 언어)
 └── gem (Ruby의 패키지 관리자, Ruby에 내장)
      ├── Jekyll (정적 사이트 생성기)
      ├── Bundler (프로젝트별 의존성 관리)
      └── jekyll-theme-chirpy (블로그 테마)
```

#### 언제 사용되나
- `gem install`로 Jekyll, Bundler를 설치할 때
- `bundle install`로 프로젝트 의존성을 설치할 때
- `bundle exec jekyll serve`로 블로그를 로컬에서 미리볼 때
- `bundle exec jekyll build`로 블로그를 HTML로 빌드할 때

#### 설치 명령어

```bash
rbenv install 3.3.6
rbenv global 3.3.6
```

#### 설치 확인

```bash
ruby --version          # => ruby 3.3.6
rbenv --version         # => rbenv 1.3.2-...
which ruby              # => ~/.rbenv/shims/ruby
```

---

## Step 2: Jekyll과 Bundler 설치

### 설치한 것들 요약

| 설치 항목 | 무엇인가 | 왜 설치했나 |
|-----------|---------|------------|
| Jekyll 4.4.1 | 정적 사이트 생성기 | Markdown → HTML 웹사이트 변환 |
| Bundler 4.0.6 | Ruby 의존성 관리 도구 | 프로젝트별 gem 버전 관리 |

### 1. Jekyll

#### 무엇인가
**정적 사이트 생성기(Static Site Generator)**. Markdown(`.md`) 파일과 템플릿을 조합하여 순수 HTML/CSS/JS로 된 웹사이트를 만들어주는 도구이다.

#### 정적 사이트 vs 동적 사이트

| | 정적 사이트 (Jekyll) | 동적 사이트 (WordPress 등) |
|---|---|---|
| **서버** | 필요 없음 (HTML 파일만 호스팅) | 서버 + DB 필요 |
| **속도** | 빠름 (미리 생성된 HTML) | 요청마다 페이지 생성 |
| **비용** | 무료 (GitHub Pages) | 서버 비용 발생 |
| **보안** | 공격 표면 없음 | DB 인젝션 등 위험 |
| **작성** | Markdown으로 작성 | 웹 에디터 사용 |

#### 동작 방식
```
[Markdown 파일] + [테마 템플릿] + [설정(_config.yml)]
                     ↓ Jekyll 빌드
              [HTML/CSS/JS 웹사이트] (_site/ 폴더)
```

#### 주요 명령어
```bash
bundle exec jekyll serve    # 로컬 개발 서버 실행 (http://127.0.0.1:4000)
bundle exec jekyll build    # HTML로 빌드만 수행 (_site/ 폴더에 출력)
```

### 2. Bundler

#### 무엇인가
Ruby 프로젝트의 **의존성(gem) 관리 도구**. 다른 언어의 비슷한 도구들:

| 언어 | 패키지 관리자 | 의존성 관리 도구 | 잠금 파일 |
|------|-------------|----------------|----------|
| Ruby | gem | **Bundler** | Gemfile.lock |
| Python | pip | pip + venv | requirements.txt |
| Node.js | npm | npm | package-lock.json |
| Java | Maven | Maven | pom.xml |

#### 핵심 파일 2개

**Gemfile** — "이 프로젝트에 필요한 gem 목록"
```ruby
gem "jekyll-theme-chirpy", "~> 7.4"   # Chirpy 테마
gem "html-proofer", "~> 5.2"          # HTML 검증 도구
```

**Gemfile.lock** — "실제로 설치된 정확한 버전 기록"
- 팀원 모두가 동일한 버전을 사용하도록 보장
- git에 커밋하여 버전을 고정

#### `bundle exec`를 붙이는 이유
`jekyll serve` 대신 `bundle exec jekyll serve`를 사용하는 이유:
- `bundle exec`는 **Gemfile.lock에 기록된 정확한 버전**의 gem을 사용하도록 보장한다
- 시스템에 여러 버전의 Jekyll이 설치되어 있어도, 프로젝트에 맞는 버전이 실행됨

#### 설치 명령어

```bash
gem install jekyll bundler
```

### 전체 도구 체인 관계도

```
rbenv (Ruby 버전 관리)
 └── Ruby 3.3.6 (프로그래밍 언어)
      └── gem (패키지 관리자, Ruby 내장)
           ├── Jekyll 4.4.1 (정적 사이트 생성기)
           └── Bundler 4.0.6 (의존성 관리)
                └── bundle install로 프로젝트 gem 설치
                     ├── jekyll-theme-chirpy (블로그 테마)
                     ├── jekyll-seo-tag (SEO 최적화)
                     ├── jekyll-sitemap (사이트맵 생성)
                     └── ... 기타 의존 gem들
```

---

## Step 3: Chirpy 템플릿으로 블로그 프로젝트 생성

### 이번 단계에서 한 것 요약

| 작업 | 무엇을 했나 | 왜 했나 |
|------|-----------|--------|
| git clone | chirpy-starter 템플릿 다운로드 | 블로그 프로젝트의 뼈대를 가져오기 위해 |
| rm -rf .git && git init | git 히스토리 초기화 | 내 블로그의 새 히스토리를 시작하기 위해 |
| bundle install | 프로젝트 의존성 설치 | Gemfile에 정의된 gem들을 설치하기 위해 |

### 1. chirpy-starter 템플릿

#### 무엇인가
Chirpy 테마 개발자(cotes2020)가 제공하는 **블로그 시작용 템플릿**이다.

#### 왜 chirpy-starter를 사용하나 (vs 전체 테마 fork)

| 방법 | chirpy-starter (우리가 선택) | chirpy 전체 fork |
|------|---------------------------|-----------------|
| **포함 내용** | 설정 파일만 (테마는 gem으로) | 테마 코드 전체 |
| **파일 수** | ~20개 | ~200개 |
| **테마 업데이트** | `bundle update`로 간편 | git merge 필요 (충돌 위험) |
| **커스터마이징** | 제한적 (설정 위주) | 자유롭게 수정 가능 |
| **권장 대상** | 초보자, 빠른 시작 | 테마를 깊게 수정할 사람 |

#### gem 기반 테마란?
테마의 레이아웃, CSS, JavaScript 등 핵심 코드가 **Ruby gem(라이브러리)으로 패키징**되어 있다.
`Gemfile`에 `gem "jekyll-theme-chirpy"`를 선언하면, `bundle install` 시 자동으로 테마가 설치된다.

우리 프로젝트에는 테마 코드가 보이지 않지만, 빌드 시 gem에서 가져와서 사용한다:
```
프로젝트 파일 (우리가 관리)     +     gem 파일 (Bundler가 관리)
├── _config.yml                     ├── _layouts/
├── _posts/                         ├── _includes/
├── _tabs/                          ├── _sass/
└── assets/img/                     └── assets/js/
         ↓                                  ↓
              Jekyll이 합쳐서 → _site/ (최종 HTML)
```

### 2. 실행한 명령어 상세

#### 2-1. 템플릿 다운로드
```bash
git clone https://github.com/cotes2020/chirpy-starter.git qudtjs0753.github.io
```
- `git clone`: 원격 저장소를 복제
- `qudtjs0753.github.io`: 복제할 폴더 이름 (GitHub Pages 규칙: `<username>.github.io`)

#### 2-2. git 히스토리 초기화
```bash
cd qudtjs0753.github.io
rm -rf .git    # 템플릿의 git 히스토리 삭제
git init        # 새로운 git 저장소 시작
```
- 템플릿의 커밋 히스토리는 우리 블로그와 무관하므로 삭제
- `git init`으로 깨끗한 상태에서 내 블로그의 히스토리를 시작

#### 2-3. 의존성 설치
```bash
bundle install
```
- `Gemfile`에 정의된 모든 gem을 설치
- 설치된 gem 총 64개 (jekyll-theme-chirpy, jekyll-seo-tag, jekyll-sitemap 등)

### 3. 프로젝트 구조

```
qudtjs0753.github.io/
├── _config.yml              # 블로그 전체 설정 (제목, URL, 언어 등)
├── _posts/                  # 블로그 글을 넣는 폴더
│   └── 2026-02-16-hello-world.md
├── _tabs/                   # 네비게이션 페이지
│   ├── about.md             #   "About" 페이지
│   ├── archives.md          #   "Archives" (글 목록)
│   ├── categories.md        #   "Categories" (카테고리별)
│   └── tags.md              #   "Tags" (태그별)
├── _data/                   # 데이터 파일
│   ├── contact.yml          #   사이드바 연락처 아이콘 설정
│   └── share.yml            #   글 하단 공유 버튼 설정
├── _plugins/                # Jekyll 플러그인
│   └── posts-lastmod-hook.rb  # 포스트 수정일 자동 감지
├── assets/img/              # 이미지 저장 폴더
├── Gemfile                  # Ruby 의존성 목록
├── Gemfile.lock             # 의존성 정확한 버전 잠금
├── index.html               # 메인 페이지 (테마가 처리)
├── .github/
│   └── workflows/
│       └── pages-deploy.yml # GitHub Actions 배포 워크플로우
├── .gitignore               # git에서 제외할 파일 목록
└── tools/
    ├── run.sh               # 로컬 실행 스크립트
    └── test.sh              # 테스트 스크립트
```

#### 각 폴더/파일의 역할

| 경로 | 역할 | 언제 수정하나 |
|------|------|-------------|
| `_config.yml` | 블로그 전체 설정 | 블로그 설정을 바꿀 때 |
| `_posts/` | 블로그 글 저장소 | 새 글을 작성할 때마다 |
| `_tabs/` | 네비게이션 페이지 | About 페이지 수정 등 |
| `_data/` | 사이드바, 공유 버튼 설정 | UI 설정을 바꿀 때 |
| `assets/img/` | 이미지 파일 | 포스트에 이미지 추가 시 |
| `Gemfile` | gem 의존성 | 새 플러그인 추가 시 |
| `.github/workflows/` | 자동 배포 설정 | 거의 수정하지 않음 |

### GitHub Pages 저장소 이름 규칙

| 저장소 이름 | URL | 비고 |
|------------|-----|------|
| `qudtjs0753.github.io` | `https://qudtjs0753.github.io` | 사용자 사이트 (1개만 가능) |
| `my-project` | `https://qudtjs0753.github.io/my-project` | 프로젝트 사이트 (여러 개 가능) |

우리는 사용자 사이트를 만들었으므로 저장소 이름이 반드시 `qudtjs0753.github.io`여야 한다.

---

## Step 4: 블로그 기본 설정 (`_config.yml`)

### 변경한 설정 요약

| 항목 | 기본값 (템플릿) | 변경한 값 | 역할 |
|------|---------------|----------|------|
| `lang` | `en` | `ko` | 블로그 UI 언어 |
| `timezone` | (비어있음) | `Asia/Seoul` | 포스트 시간 기준 |
| `title` | `Chirpy` | `가보자` | 블로그 제목 |
| `tagline` | `A text-focused Jekyll theme` | `무의식적 기록` | 부제목 |
| `description` | `A minimal, responsive...` | `아무거나 다쓰는 곳` | SEO 설명 |
| `url` | (비어있음) | `https://qudtjs0753.github.io` | 블로그 주소 |
| `github.username` | `github_username` | `qudtjs0753` | GitHub 유저명 |
| `social.name` | `your_full_name` | `kbs` | 저자 이름 |

### `_config.yml`이란?

Jekyll의 **전역 설정 파일**이다. YAML(Yet Another Markup Language) 형식으로 작성되며, 블로그의 모든 기본 동작을 제어한다.

#### YAML 기본 문법
```yaml
# 키-값 쌍
title: 가보자

# 중첩 구조
github:
  username: qudtjs0753

# 여러 줄 텍스트 (>- : 줄바꿈을 공백으로 변환, 마지막 줄바꿈 제거)
description: >-
  아무거나 다쓰는 곳

# 리스트
social:
  links:
    - https://github.com/qudtjs0753
```

### 설정 항목 상세 설명

#### `lang: ko`
- 블로그 UI의 언어를 설정한다
- `ko`로 설정하면 "Categories" → "카테고리", "Tags" → "태그" 등으로 표시
- Chirpy 테마의 `_data/locales/ko.yml` 파일에 번역이 정의되어 있음

#### `timezone: Asia/Seoul`
- 포스트의 날짜/시간 계산에 사용되는 시간대
- 이 설정이 없으면 서버(GitHub Actions)의 시간대(UTC)를 사용
- `Asia/Seoul`은 UTC+9 (한국 표준시)

#### `title: 가보자`
- 사이드바 상단과 브라우저 탭에 표시되는 **블로그 제목**
- SEO의 `og:site_name`으로도 사용됨

#### `tagline: 무의식적 기록`
- 사이드바에서 제목 아래에 표시되는 **부제목**

#### `description: 아무거나 다쓰는 곳`
- 검색 엔진에 노출되는 **블로그 설명** (SEO용)
- HTML의 `<meta name="description">` 태그에 들어감
- Google 검색 결과에서 블로그 이름 아래에 표시되는 텍스트

#### `url: "https://qudtjs0753.github.io"`
- 블로그의 **전체 주소**
- 사이트맵, RSS 피드, canonical URL 등에 사용
- 끝에 `/`를 붙이지 않는다

#### `github.username: qudtjs0753`
- 사이드바에 GitHub 아이콘 링크로 표시됨
- 클릭하면 `https://github.com/qudtjs0753`으로 이동

#### `social.name: kbs`
- 포스트의 **기본 저자 이름**으로 표시
- 페이지 하단 copyright에도 사용됨

### 건드리지 않은 설정들

#### 나중에 필요시 설정할 것들
| 항목 | 역할 | 언제 설정하나 |
|------|------|-------------|
| `avatar` | 사이드바 프로필 사진 | 프로필 이미지를 추가할 때 |
| `social.email` | 이메일 주소 | 공개할 이메일이 있을 때 |
| `comments.provider` | 댓글 시스템 | 댓글 기능을 추가할 때 |
| `analytics.google.id` | Google Analytics | 방문자 분석이 필요할 때 |
| `theme_mode` | 다크/라이트 모드 | 기본 테마를 고정하고 싶을 때 |

#### 수정하면 안 되는 것들 (155줄 이후)
```yaml
# ------------ The following options are not recommended to be modified --
kramdown:       # Markdown 렌더링 설정
collections:    # Jekyll 컬렉션 설정
defaults:       # 포스트/페이지 기본값
sass:           # CSS 컴파일 설정
compress_html:  # HTML 압축 설정
exclude:        # 빌드 제외 파일
jekyll-archives: # 아카이브 설정
```

> 이 항목들은 Chirpy 테마의 동작에 필요한 설정이므로, 수정하면 블로그가 깨질 수 있다.
{: .prompt-warning }

### 설정 변경 후 적용 방법

`_config.yml`을 수정한 후에는 **Jekyll 서버를 재시작**해야 한다:
```bash
# Ctrl+C로 서버 중지 후 다시 실행
bundle exec jekyll serve
```

> 일반 포스트 수정은 자동으로 반영되지만, `_config.yml`은 서버 시작 시에만 읽히기 때문에 재시작이 필요하다.
{: .prompt-info }

---

## Step 5: 첫 번째 포스트 작성

### 포스트 작성 규칙

#### 1. 파일 위치
모든 블로그 글은 **`_posts/` 폴더** 안에 넣어야 한다. Jekyll은 이 폴더만 블로그 글로 인식한다.

#### 2. 파일명 형식 (필수)
```
YYYY-MM-DD-제목.md
```

| 부분 | 설명 | 예시 |
|------|------|------|
| `YYYY-MM-DD` | 작성 날짜 | `2026-02-16` |
| `제목` | URL에 사용될 슬러그 | `hello-world` |
| `.md` | Markdown 확장자 | `.md` |

**예시**: `2026-02-16-hello-world.md` → URL: `/posts/hello-world/`

> 파일명의 날짜가 미래인 경우, 해당 날짜가 될 때까지 포스트가 표시되지 않는다. (개발 서버에서는 `--future` 플래그로 볼 수 있음)
{: .prompt-info }

#### 3. 파일명 작성 팁
- 영어 소문자와 하이픈(`-`) 사용 권장
- 공백 대신 하이픈 사용
- 한글도 가능하지만 URL이 인코딩되어 길어질 수 있음

```
좋은 예: 2026-02-16-java-spring-tutorial.md
나쁜 예: 2026-02-16-Java Spring Tutorial.md
```

### Front Matter

#### 무엇인가
파일 맨 위에 `---`로 감싼 **YAML 형식의 메타데이터**이다. Jekyll이 이 정보를 읽어서 포스트의 제목, 날짜, 카테고리 등을 설정한다.

#### 실제 작성한 Front Matter
```yaml
---
title: "첫 번째 블로그 포스트"
date: 2026-02-16 12:00:00 +0900
categories: [Blogging, Tutorial]
tags: [hello, jekyll, chirpy]
---
```

#### 각 항목 설명

**`title`** (필수)
- 브라우저와 블로그에 표시되는 **글 제목**
- 파일명의 제목과 다를 수 있음 (파일명은 URL용, title은 표시용)

**`date`** (필수)
- `YYYY-MM-DD HH:MM:SS +TTTT` 형식
- `+0900`은 **한국 시간(KST, UTC+9)**을 의미
- 이 값이 없으면 파일명의 날짜를 사용

**`categories`** (권장)
- **최대 2단계** 카테고리 지정 가능
- `[대분류, 소분류]` 구조
- URL에 반영됨: `/categories/blogging/`

**`tags`** (권장)
- 개수 제한 없음
- **소문자 권장** (대소문자가 다르면 별도 태그로 인식)

#### 선택적 Front Matter 항목

| 항목 | 기본값 | 설명 | 예시 |
|------|--------|------|------|
| `pin: true` | false | 홈 화면 상단에 고정 | 중요한 공지 글 |
| `toc: false` | true | 목차(Table of Contents) 비활성화 | 짧은 글 |
| `math: true` | false | LaTeX 수학 수식 활성화 | 알고리즘 글 |
| `mermaid: true` | false | Mermaid 다이어그램 활성화 | 설계 문서 |
| `image.path` | 없음 | 포스트 상단 미리보기 이미지 | 썸네일 |
| `comments: false` | true | 댓글 비활성화 | 댓글 불필요한 글 |

### Chirpy 테마 전용 기능

#### 프롬프트 (알림 박스)
```markdown
> 이것은 팁입니다.
{: .prompt-tip }

> 이것은 정보입니다.
{: .prompt-info }

> 이것은 경고입니다.
{: .prompt-warning }

> 이것은 위험입니다.
{: .prompt-danger }
```

### 새 글 작성 워크플로우

```
1. _posts/ 폴더에 YYYY-MM-DD-제목.md 파일 생성
2. Front Matter 작성 (title, date, categories, tags)
3. Markdown으로 본문 작성
4. 로컬에서 확인: bundle exec jekyll serve
5. git add → git commit → git push
6. GitHub Actions가 자동으로 빌드 & 배포
```

---

## Step 6: 로컬에서 블로그 미리보기

### Jekyll 개발 서버

#### 실행 명령어
```bash
cd /home/helloworld/workspace/qudtjs0753.github.io
bundle exec jekyll serve
```

#### 이 명령이 하는 일
1. **빌드**: Markdown + 템플릿 → HTML/CSS/JS 생성 (`_site/` 폴더에 출력)
2. **서버 실행**: 생성된 파일을 `http://127.0.0.1:4000`에서 제공
3. **파일 감시**: 파일이 변경되면 자동으로 다시 빌드 (live reload)

#### 실행 시 출력 내용
```
Configuration file: /home/helloworld/workspace/qudtjs0753.github.io/_config.yml
            Source: /home/helloworld/workspace/qudtjs0753.github.io
       Destination: /home/helloworld/workspace/qudtjs0753.github.io/_site
 Incremental build: disabled. Enable with --incremental
      Generating...
                    done in 0.593 seconds.
 Auto-regeneration: enabled for '/home/helloworld/workspace/qudtjs0753.github.io'
    Server address: http://127.0.0.1:4000/
  Server running... press ctrl-c to stop.
```

### `_site/` 폴더

Jekyll이 빌드한 **최종 결과물**이 저장되는 폴더이다. GitHub Pages에 실제로 배포되는 것이 바로 이 폴더의 내용이다.

```
_site/
├── index.html          # 메인 페이지
├── posts/
│   └── hello-world/
│       └── index.html  # 블로그 포스트
├── categories/         # 카테고리 페이지
├── tags/               # 태그 페이지
├── assets/
│   ├── css/            # 컴파일된 CSS
│   └── js/             # JavaScript
├── sitemap.xml         # 사이트맵
└── feed.xml            # RSS 피드
```

> `_site/`는 `.gitignore`에 포함되어 있어서 git에 커밋되지 않는다. GitHub Actions에서 서버 측에서 빌드하기 때문이다.
{: .prompt-info }

### 유용한 서버 옵션

```bash
# 기본 실행
bundle exec jekyll serve

# 외부 네트워크에서도 접속 가능 (같은 WiFi 내 다른 기기에서 테스트)
bundle exec jekyll serve --host 0.0.0.0

# 미래 날짜의 포스트도 표시
bundle exec jekyll serve --future

# 초고(draft) 포스트도 표시
bundle exec jekyll serve --drafts

# 증분 빌드 (변경된 파일만 빌드, 더 빠름)
bundle exec jekyll serve --incremental

# 포트 변경
bundle exec jekyll serve --port 5000
```

### 자동 재빌드와 예외

**자동으로 반영되는 변경**
- `_posts/` 안의 파일 수정/추가
- `_tabs/` 안의 파일 수정
- `assets/` 안의 파일 변경

**서버 재시작이 필요한 변경**
- **`_config.yml`** 수정 → `Ctrl+C`로 중지 후 다시 `bundle exec jekyll serve`
- **`Gemfile`** 수정 → `bundle install` 후 서버 재시작

### 확인한 결과

빌드된 HTML에서 우리가 설정한 값들이 정상 반영되었음을 확인했다:

```html
<html lang="ko">                                    <!-- lang: ko -->
<meta property="og:title" content="가보자" />         <!-- title: 가보자 -->
<meta name="description" content="아무거나 다쓰는 곳" /> <!-- description -->
<meta property="og:site_name" content="가보자" />
"name":"kbs"                                         <!-- social.name: kbs -->
"sameAs":["https://github.com/qudtjs0753"]          <!-- social.links -->
```

---

## Step 7: GitHub에 배포

### 이번 단계에서 한 것 요약

| 작업 | 명령어 | 왜 했나 |
|------|--------|--------|
| Linux 플랫폼 추가 | `bundle lock --add-platform x86_64-linux` | GitHub Actions가 Linux에서 빌드하므로 |
| Git 초기 커밋 | `git add . && git commit` | 모든 파일을 버전 관리에 등록 |
| 브랜치 이름 변경 | `git branch -M main` | GitHub 기본 브랜치 이름에 맞추기 |
| 리모트 연결 | `git remote add origin ...` | 로컬 저장소와 GitHub 연결 |
| Push | `git push -u origin main` | GitHub에 코드 업로드 |
| GitHub Pages 설정 | Source → GitHub Actions | 자동 빌드/배포 활성화 |

### 1. Linux 플랫폼 추가

```bash
bundle lock --add-platform x86_64-linux
```

- `Gemfile.lock`에 플랫폼 정보가 기록되어 있는데, Linux 플랫폼이 빠져있으면 GitHub Actions에서 `bundle install`이 실패한다
- 이 명령으로 `Gemfile.lock`에 `x86_64-linux`를 추가하여, Linux 환경에서도 정상 동작하도록 보장

### 2. Git 커밋 및 Push

```bash
# 전체 파일 스테이징 (.gitignore에 정의된 파일은 자동 제외)
git add .

# 커밋 생성
git commit -m "Initial blog setup with Chirpy theme"

# 브랜치 이름을 main으로 변경 (GitHub 기본 브랜치에 맞춤)
git branch -M main

# GitHub 리모트 저장소 연결
git remote add origin https://github.com/qudtjs0753/qudtjs0753.github.io.git

# Push (-u: 이후 git push만 입력해도 origin main으로 push)
git push -u origin main
```

### 3. GitHub Pages 설정

GitHub Pages의 빌드 방식은 2가지가 있다:

| Source | 동작 방식 | 장점 | 단점 |
|--------|----------|------|------|
| **Deploy from a branch** | GitHub의 내장 Jekyll 빌드 | 설정 간단 | 플러그인 제한, 오래된 Jekyll 버전 |
| **GitHub Actions** (우리가 선택) | 워크플로우로 직접 빌드 | 최신 Jekyll, 모든 플러그인 사용 가능 | 워크플로우 파일 필요 |

> Chirpy 테마는 커스텀 플러그인을 사용하므로 **GitHub Actions**가 필수이다.
{: .prompt-warning }

설정 방법: GitHub 리포지토리 → **Settings** → **Pages** → **Source**를 **GitHub Actions**로 변경

#### 배포 흐름

```
Push to main
    ↓
GitHub Actions 시작
    ↓
1. Ubuntu 환경에서 Ruby 설치
2. bundle install (gem 설치)
3. bundle exec jekyll build (HTML 빌드)
4. 빌드 결과물을 GitHub Pages에 배포
    ↓
https://qudtjs0753.github.io 에서 접속 가능
```

### 4. 배포 결과 확인

```bash
gh run list --repo qudtjs0753/qudtjs0753.github.io --limit 5
```

```
completed  success  Initial blog setup with Chirpy theme  Build and Deploy         main  push     1m8s
completed  success  pages build and deployment            pages-build-deployment   main  dynamic  25s
```

- **Build and Deploy** (1분 8초): Chirpy의 워크플로우가 Jekyll 빌드 수행
- **pages-build-deployment** (25초): GitHub이 빌드 결과물을 Pages에 배포

