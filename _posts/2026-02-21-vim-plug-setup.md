---
title: "Vim 플러그인 매니저 vim-plug 설치 및 설정"
date: 2026-02-21 12:00:00 +0900
categories: [Vim]
tags: [vim-plug]
---

Vim에서 플러그인을 쉽게 관리할 수 있게 해주는 **vim-plug** 설치 및 설정 방법을 정리한다.

---

## vim-plug란?

Vim/Neovim용 **플러그인 매니저**이다. 플러그인을 설치, 업데이트, 삭제하는 과정을 명령어 한 줄로 처리할 수 있다.

### 왜 플러그인 매니저가 필요한가

| | 수동 설치 | vim-plug 사용 |
|---|---|---|
| **설치** | git clone + 경로 직접 관리 | `:PlugInstall` 한 줄 |
| **업데이트** | 각 플러그인 폴더에서 git pull | `:PlugUpdate` 한 줄 |
| **삭제** | 폴더 직접 삭제 + 설정 정리 | `.vimrc`에서 제거 후 `:PlugClean` |
| **빌드 단계** | 플러그인마다 다른 방식 | `do` 옵션으로 자동화 |

---

## 1. vim-plug 설치

```bash
curl -fLo ~/.vim/autoload/plug.vim --create-dirs \
  https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim
```

이 명령이 하는 일:
- `~/.vim/autoload/` 디렉토리를 생성하고
- `plug.vim` 파일을 다운로드하여 넣는다
- Vim의 `autoload` 메커니즘에 의해 자동으로 로드된다

---

## 2. .vimrc 설정

`~/.vimrc` 파일에 아래 내용을 추가한다. 파일이 없으면 새로 만든다.

```vim
call plug#begin('~/.vim/plugged')

" 여기에 플러그인을 추가한다
" Plug '저장소/플러그인이름'

call plug#end()
```

### 설정 구조 설명

```vim
call plug#begin('~/.vim/plugged')   " 플러그인 설치 경로 지정, 시작
Plug '...'                           " 설치할 플러그인 선언
Plug '...'
call plug#end()                      " 끝
```

- `plug#begin()` ~ `plug#end()` 사이에 `Plug` 명령으로 플러그인을 선언한다
- `'~/.vim/plugged'`는 플러그인이 설치될 디렉토리 경로이다

---

## 3. 플러그인 관리 명령어

`.vimrc`에 플러그인을 추가한 후 Vim을 열고:

| 명령어 | 동작 |
|--------|------|
| `:PlugInstall` | 새로 추가한 플러그인 설치 |
| `:PlugUpdate` | 설치된 플러그인 업데이트 |
| `:PlugClean` | `.vimrc`에서 제거된 플러그인 삭제 |
| `:PlugStatus` | 플러그인 상태 확인 |

### 플러그인 추가 워크플로우

```
1. ~/.vimrc에 Plug '...' 추가
2. Vim 열기
3. :PlugInstall 실행
4. 완료
```

### 플러그인 제거 워크플로우

```
1. ~/.vimrc에서 Plug '...' 줄 삭제
2. Vim 열기
3. :PlugClean 실행 (파일 삭제 확인)
4. 완료
```

---

## 4. 플러그인 추가 예시: Markdown Preview

브라우저에서 마크다운을 실시간 미리보기할 수 있는 플러그인이다.

```vim
call plug#begin('~/.vim/plugged')

Plug 'iamcco/markdown-preview.nvim', { 'do': { -> mkdp#util#install() } }

call plug#end()
```

- `{ 'do': ... }` — 설치 후 자동으로 빌드 스크립트를 실행하는 옵션이다
- 이 markdown 플러그인은 Node.js 기반이므로 **Node.js가 설치되어 있어야** 한다
```
사용법 참고
:MarkdownPreview        " 브라우저에서 미리보기 시작
:MarkdownPreviewStop    " 미리보기 중지
```

---

## 5. .vimrc 저장 시 자동 반영

`.vimrc`를 수정한 뒤 매번 Vim을 재시작하거나 `:source ~/.vimrc`를 직접 입력하는 건 번거롭다.

아래 설정을 추가하면 `.vimrc`를 저장(`:w`)할 때마다 **자동으로 source**된다.

```vim
autocmd BufWritePost ~/.vimrc source ~/.vimrc
```

### 설정 설명

| 요소 | 의미 |
|------|------|
| `autocmd` | 특정 이벤트 발생 시 명령을 자동 실행 |
| `BufWritePost` | 파일이 저장된 직후 발생하는 이벤트 |
| `~/.vimrc` | 이 파일에 대해서만 동작하도록 제한 |
| `source ~/.vimrc` | 실행할 명령 — `.vimrc`를 다시 읽어들인다 |

### 적용된 전체 .vimrc 예시

```vim
call plug#begin('~/.vim/plugged')

Plug 'iamcco/markdown-preview.nvim', { 'do': { -> mkdp#util#install() } }

call plug#end()

autocmd BufWritePost ~/.vimrc source ~/.vimrc
```

이제 플러그인을 추가하거나 설정을 변경한 뒤 `:w`만 누르면 바로 반영된다.

### 주의: source는 Vim 내부 명령이다

터미널에서 `source ~/.vimrc`를 실행하면 오류가 발생한다.

```bash
$ source ~/.vimrc
bash: syntax error near unexpected token `plug#begin'
```

`source`는 셸과 Vim 양쪽에 존재하는 명령이지만 동작이 다르다.

| 환경 | `source` 동작 |
|------|---------------|
| **Bash** | 셸 스크립트(`.bashrc` 등)를 현재 셸에서 실행 |
| **Vim** | Vim 스크립트(`.vimrc` 등)를 Vim 내부에서 실행 |

`.vimrc`는 Vim 스크립트 문법으로 작성되어 있으므로, Bash가 해석하면 문법 오류가 난다. `.vimrc`를 다시 읽어들이려면 반드시 **Vim 안에서** `:source ~/.vimrc`를 사용해야 한다.

---

## 참고

- [vim-plug GitHub](https://github.com/junegunn/vim-plug)
- [markdown-preview.nvim GitHub](https://github.com/iamcco/markdown-preview.nvim)
