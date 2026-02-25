---
title: "Neovim에 C 개발환경 구축하기 — LSP, 자동완성, 구문 강조"
date: 2026-02-25 20:44:00 +0900
categories: [DevEnv]
tags: [vim]
---

**Neovim + Lua 설정**을 통해 C언어 개발 환경 구축한 내용 정리. LSP(Language Server Protocol)를 통한 코드 분석, 자동완성, AST 기반 구문 강조까지 설정. 왜 C냐? 그냥 알고리즘 공부하는데 심심해서.. 왜 VIM이냐? 단축키도 많이 쓰고 그냥 멋있어보여서 ㅋㅋ..

## 최종 구성 요약

| 기능 | 플러그인 | 역할 |
|------|----------|------|
| 플러그인 관리 | lazy.nvim | 플러그인 매니저 (자동 로드) |
| LSP 서버 관리 | mason.nvim + mason-lspconfig | clangd 자동 설치/연결 |
| LSP 설정 | nvim-lspconfig | 서버별 설정 + 키바인딩 |
| 자동완성 | blink.cmp | LSP/버퍼/경로/스니펫 통합 완성 |
| 구문 강조 | nvim-treesitter | AST 기반 정확한 하이라이팅 |

```
~/dotfiles/.config/nvim/
├── init.lua                 # 진입점 (require 4줄)
└── lua/
    ├── config/
    │   ├── options.lua      # 에디터 기본설정
    │   └── lazy.lua         # lazy.nvim 부트스트랩
    └── plugins/
        ├── lsp.lua          # mason + lspconfig + 키바인딩
        ├── cmp.lua          # blink.cmp 자동완성
        └── treesitter.lua   # 구문 강조
```

---

## 1. Neovim 설치

Ubuntu 24.04 기본 apt 패키지는 v0.9.5로 오래된 버전이다. PPA를 추가하여 최신 버전을 설치한다.

```bash
sudo add-apt-repository ppa:neovim-ppa/unstable -y
sudo apt update
sudo apt install neovim
```

```bash
nvim --version
# NVIM v0.12.0-dev
```

---

## 2. 디렉토리 구조 생성

```bash
mkdir -p ~/dotfiles/.config/nvim/lua/config
mkdir -p ~/dotfiles/.config/nvim/lua/plugins
```

Neovim의 Lua 설정은 `lua/` 디렉토리 아래에 모듈로 구성한다. `require("config.options")`는 `lua/config/options.lua` 파일을 로드한다.

---

## 3. 진입점 — init.lua

```lua
require("config.options")
require("config.lazy")
require("config.keymaps")
require("config.autocmds")
```

Neovim이 시작되면 `init.lua`를 실행한다. 각 모듈을 순서대로 로드하며, **options → lazy(플러그인) → keymaps → autocmds** 순서가 중요하다. 플러그인이 로드되기 전에 기본 옵션이 먼저 설정되어야 한다.

---

## 4. 에디터 기본설정 — options.lua

```lua
-- Leader key: 단축키 조합의 시작 키를 스페이스바로 설정
-- lazy.nvim이 키맵을 등록하기 전에 먼저 설정해야 함
vim.g.mapleader = " "
vim.g.maplocalleader = " "

-- ── 기존 .vimrc에서 포팅 ──────────────────────────────
vim.opt.number = true                          -- 줄 번호 표시
vim.opt.title = true                           -- 터미널 타이틀에 파일명 표시
vim.opt.ignorecase = true                      -- 검색 시 대소문자 무시
vim.opt.smartcase = true                       -- 대문자 포함 시 대소문자 구분 검색
vim.opt.hlsearch = true                        -- 검색 결과 하이라이트
vim.opt.incsearch = true                       -- 타이핑하면서 실시간 검색
vim.opt.fileencodings = { "utf-8", "euc-kr" }  -- 파일 열 때 인코딩 자동 감지 순서

-- ── 개발용 추가 설정 ──────────────────────────────────
vim.opt.relativenumber = true   -- 현재 줄 기준 상대 줄 번호 (5j, 12k 같은 이동에 유용)
vim.opt.tabstop = 4             -- 탭 문자의 표시 너비 (4칸)
vim.opt.shiftwidth = 4          -- 자동 들여쓰기 너비 (4칸)
vim.opt.expandtab = true        -- 탭 키 입력 시 스페이스로 변환
vim.opt.smartindent = true      -- 새 줄에서 이전 줄의 들여쓰기를 자동 유지
vim.opt.wrap = false            -- 긴 줄 자동 줄바꿈 끄기 (가로 스크롤)
vim.opt.cursorline = true       -- 현재 커서가 있는 줄 하이라이트
vim.opt.signcolumn = "yes"      -- 좌측 사인 열 항상 표시 (git 변경, LSP 경고 등)
vim.opt.termguicolors = true    -- 24비트 트루컬러 지원 (gruvbox 테마에 필요)
vim.opt.scrolloff = 8           -- 커서 위아래로 항상 8줄 여백 유지
vim.opt.updatetime = 250        -- CursorHold 이벤트 대기 시간 (ms), LSP 호버 반응 속도에 영향
vim.opt.undofile = true         -- 파일을 닫았다 열어도 undo 기록 유지
vim.opt.splitright = true       -- 수직 분할 시 새 창이 오른쪽에 열림
vim.opt.splitbelow = true       -- 수평 분할 시 새 창이 아래에 열림
```

### Vim → Neovim 설정 변환 규칙

| Vim (.vimrc) | Neovim (Lua) | 설명 |
|---|---|---|
| `set number` | `vim.opt.number = true` | 불리언 옵션 |
| `set tabstop=4` | `vim.opt.tabstop = 4` | 숫자 옵션 |
| `set fileencodings=utf-8,euc-kr` | `vim.opt.fileencodings = { "utf-8", "euc-kr" }` | 리스트 옵션 |
| `let mapleader = " "` | `vim.g.mapleader = " "` | 글로벌 변수 |

---

## 5. 플러그인 매니저 — lazy.nvim

```lua
-- lazy.nvim 부트스트랩: 없으면 자동으로 git clone
local lazypath = vim.fn.stdpath("data") .. "/lazy/lazy.nvim"
if not vim.uv.fs_stat(lazypath) then
  vim.fn.system({
    "git", "clone", "--filter=blob:none",
    "https://github.com/folke/lazy.nvim.git",
    "--branch=stable",
    lazypath,
  })
end
vim.opt.rtp:prepend(lazypath)

-- plugins/ 디렉토리의 모든 파일을 자동으로 플러그인 스펙으로 로드
require("lazy").setup("plugins")
```

기존 vim-plug와의 차이:

| | vim-plug | lazy.nvim |
|---|---|---|
| **설치** | curl로 수동 다운로드 | 부트스트랩 코드가 자동 설치 |
| **플러그인 선언** | `.vimrc`에 `Plug '...'` 나열 | `plugins/` 폴더에 파일 1개 = 플러그인 1개 |
| **로딩** | 전부 즉시 로드 | 이벤트 기반 지연 로드 (lazy loading) |
| **추가/삭제** | `.vimrc` 수정 → `:PlugInstall` | 파일 추가/삭제 → 자동 반영 |

`require("lazy").setup("plugins")` 한 줄로 `lua/plugins/` 디렉토리의 모든 `.lua` 파일을 플러그인 스펙으로 인식한다. 새 플러그인 추가 = 파일 1개 추가.

---

## 6. 구문 강조 — treesitter.lua

```lua
return {
  -- AST 기반 구문 강조 엔진: 정규식보다 정확한 하이라이팅 제공
  "nvim-treesitter/nvim-treesitter",
  build = ":TSUpdate",  -- 설치/업데이트 시 파서 바이너리 자동 컴파일
  event = "BufReadPost", -- 파일을 열 때 로드 (lazy loading)
  opts = {
    -- 자동 설치할 언어 파서 목록 (새 언어 추가 시 여기에 한 줄 추가)
    ensure_installed = { "c", "lua", "vim", "vimdoc", "query", "markdown" },
    highlight = {
      enable = true,  -- treesitter 기반 구문 강조 활성화
    },
    indent = {
      enable = true,  -- treesitter 기반 자동 들여쓰기
    },
  },
}
```

### Treesitter란?

Vim의 기본 구문 강조는 **정규식** 기반이다. `int`라는 문자열이 있으면 맥락에 관계없이 키워드로 색칠한다. Treesitter는 코드를 **AST(Abstract Syntax Tree)**로 파싱하여 "이 `int`는 타입 선언이다", "이 `int`는 변수명 일부다"를 구분할 수 있다.

`ensure_installed`에 나열한 언어의 파서는 **nvim-treesitter 플러그인**이 Neovim 최초 실행 시 자동으로 다운로드+컴파일한다. 파서 바이너리는 `~/.local/share/nvim/lazy/nvim-treesitter/parser/` 에 저장된다. Python 추가 시 이 리스트에 `"python"` 한 줄 추가하면 된다.

---

## 7. LSP 설정 — lsp.lua

LSP(Language Server Protocol)는 에디터와 언어 분석 서버 사이의 **표준 프로토콜**이다. Neovim(클라이언트)이 clangd(서버)에 "이 함수의 정의가 어디야?"라고 물으면, clangd가 위치를 알려주는 방식이다.
LSP에 대해서는 추후 정리해볼 예정. 재밌는 주제다.

```lua
return {
  {
    -- LSP 서버 자동 설치/관리 (바이너리를 ~/.local/share/nvim/mason에 보관)
    "mason-org/mason.nvim",
    opts = {},
  },
  {
    -- mason이 설치한 서버를 자동으로 vim.lsp.enable() 해주는 브릿지
    "mason-org/mason-lspconfig.nvim",
    dependencies = {
      "mason-org/mason.nvim",
      "neovim/nvim-lspconfig",  -- 각 서버의 기본 설정값(cmd, filetypes 등) 제공
    },
    opts = {
      -- 자동 설치할 LSP 서버 목록 (새 언어 추가 시 여기에 한 줄 추가)
      ensure_installed = { "clangd" },
    },
  },
  {
    -- LSP 키바인딩 + capabilities 설정
    "neovim/nvim-lspconfig",
    event = "BufReadPre",
    dependencies = { "saghen/blink.cmp" },

    config = function()
      -- blink.cmp이 제공하는 추가 완성 기능을 LSP에 알려줌
      local capabilities = require("blink.cmp").get_lsp_capabilities()

      -- ── 언어별 서버 설정 (새 언어 추가 시 이 패턴 복사) ──
      vim.lsp.config("clangd", {
        capabilities = capabilities,
      })
      vim.lsp.enable("clangd")

      -- LSP가 버퍼에 연결될 때 키바인딩 등록
      vim.api.nvim_create_autocmd("LspAttach", {
        callback = function(args)
          local map = function(keys, func, desc)
            vim.keymap.set("n", keys, func, { buffer = args.buf, desc = "LSP: " .. desc })
          end

          -- ── g 접두사: 이동 계열 ──
          map("gd", vim.lsp.buf.definition, "Go to Definition")
          map("gD", vim.lsp.buf.declaration, "Go to Declaration")
          map("gr", vim.lsp.buf.references, "Go to References")

          -- ── K: 문서 보기 (Vim 기본 K를 LSP 호버로 대체) ──
          map("K", vim.lsp.buf.hover, "Hover Documentation")

          -- ── <leader> 접두사: 리팩토링/액션 ──
          map("<leader>rn", vim.lsp.buf.rename, "Rename Symbol")
          map("<leader>ca", vim.lsp.buf.code_action, "Code Action")

          -- ── [ ] 접두사: 진단 메시지 이동 ──
          map("[d", vim.diagnostic.goto_prev, "Prev Diagnostic")
          map("]d", vim.diagnostic.goto_next, "Next Diagnostic")
          map("<leader>d", vim.diagnostic.open_float, "Line Diagnostic")
        end,
      })
    end,
  },
}
```

### 세 플러그인의 역할 구분

```
mason.nvim          clangd 바이너리를 ~/.local/share/nvim/mason/에 다운로드/설치
      ↓
mason-lspconfig     mason이 설치한 서버를 vim.lsp.enable()로 자동 활성화
      ↓
nvim-lspconfig      각 서버의 기본 설정값(cmd, filetypes, root_dir) 제공
```

mason-lspconfig의 `ensure_installed = { "clangd" }` 설정이 mason.nvim에 "clangd를 설치해라"라고 지시한다. mason이 바이너리를 다운로드하면, mason-lspconfig이 자동으로 `vim.lsp.enable("clangd")`를 호출하여 서버를 활성화한다.

### nvim-lspconfig이 제공하는 기본 설정값

LSP 서버(clangd, pyright 등)는 Neovim과 별도로 실행되는 **독립 프로세스**이다. Neovim이 서버를 실행하려면 최소 3가지를 알아야 한다:

| 설정 | 의미 | clangd 예시 |
|---|---|---|
| `cmd` | 서버를 실행하는 명령어 | `{ "clangd" }` |
| `filetypes` | 어떤 파일을 열면 서버를 시작할지 | `{ "c", "cpp", "objc" }` |
| `root_dir` | 프로젝트 루트를 어떻게 찾을지 | `compile_commands.json` 또는 `.git`이 있는 디렉토리 |

이걸 매 서버마다 직접 작성하면 번거롭다. nvim-lspconfig이 **수백 개 서버의 기본값을 미리 정의**해두었기 때문에, 우리는 `vim.lsp.config("clangd", { capabilities = ... })` 한 줄만 쓰면 나머지는 nvim-lspconfig이 채워준다.

### LSP 키바인딩 요약

| 키 | 동작 | Vim 관례 |
|---|---|---|
| `gd` | 정의로 이동 | g = go |
| `gD` | 선언으로 이동 | 대문자 = 변형 |
| `gr` | 참조 목록 | g = go, r = references |
| `K` | 문서 팝업 | Vim 기본 K의 LSP 버전 |
| `<leader>rn` | 이름 변경 | rn = rename |
| `<leader>ca` | 코드 액션 | ca = code action |
| `[d` / `]d` | 이전/다음 진단 | [ ] = 이전/다음 관례 |
| `<leader>d` | 진단 팝업 | d = diagnostic |

### 나중에 언어 추가하는 방법

Python을 추가하고 싶다면:

1. `lsp.lua`의 `ensure_installed`에 `"pyright"` 추가
2. `vim.lsp.config("pyright", { capabilities = capabilities })` 추가
3. `vim.lsp.enable("pyright")` 추가
4. `treesitter.lua`의 `ensure_installed`에 `"python"` 추가

키바인딩은 `LspAttach` autocmd로 공통 적용되므로 추가 작업 불필요.

---

## 8. 자동완성 — blink.cmp (cmp.lua)

blink.cmp은 타이핑할 때 드롭다운 메뉴로 완성 후보를 보여주는 **자동완성 엔진**이다. 후보는 여러 "소스"에서 수집된다:

| 소스 | LSP 필요 여부 | 하는 일 |
|------|:---:|------|
| `lsp` | O | LSP 서버에 질의하여 함수명, 변수명, 매크로 등을 받아옴 |
| `buffer` | X | 현재 열린 파일에 있는 단어를 수집해서 제안 |
| `path` | X | `./sr` 입력 시 `./src/` 같은 파일 경로를 제안 |
| `snippets` | X | `for` 입력 시 for 루프 템플릿으로 확장 |

LSP 서버가 없는 파일(마크다운, 설정 파일 등)에서도 buffer, path, snippets 완성은 동작한다.

```lua
return {
  -- 자동완성 엔진: LSP, 버퍼, 경로, 스니펫 소스가 모두 내장
  "saghen/blink.cmp",
  version = "1.*",  -- 릴리스 태그 기준으로 미리 빌드된 Rust 바이너리 사용

  ---@module 'blink.cmp'
  ---@type blink.cmp.Config
  opts = {
    -- 기본 키맵 프리셋 사용:
    --   Ctrl+n/p  후보 이동        Ctrl+b/f  문서 스크롤
    --   Ctrl+y    후보 확정        Ctrl+e    메뉴 닫기
    --   Ctrl+Space 메뉴 열기       Tab/S-Tab 스니펫 탭 정지점 이동
    keymap = { preset = "default" },

    completion = {
      -- 문서 팝업을 자동으로 띄울지 여부 (false면 Ctrl+Space로 수동 확인)
      documentation = { auto_show = true },
    },

    -- 완성 후보 소스 (내장 소스들, 순서 = 우선순위)
    sources = {
      default = { "lsp", "path", "snippets", "buffer" },
    },
  },
  opts_extend = { "sources.default" },
}
```

### 왜 nvim-cmp 대신 blink.cmp인가

| | nvim-cmp | blink.cmp |
|---|---|---|
| 소스 플러그인 | 별도 5개 설치 필요 | LSP, buffer, path, snippets **내장** |
| 퍼지 매칭 | Lua | Rust (오타 허용, 더 빠름) |
| 설정량 | ~50줄 | ~20줄 |
| 상태 | 유지보수 모드 | 활발한 개발 중 |

### 자동완성 키바인딩

| 키 | 동작 |
|---|---|
| `Ctrl+n` / `Ctrl+p` | 다음/이전 후보 이동 |
| `Ctrl+y` | 선택한 후보 확정 |
| `Ctrl+e` | 완성 메뉴 닫기 |
| `Ctrl+Space` | 수동으로 메뉴 열기 |
| `Ctrl+b` / `Ctrl+f` | 문서 팝업 위/아래 스크롤 |
| `Tab` / `Shift+Tab` | 스니펫 탭 정지점 이동 |

---

## 검증

```bash
# Neovim 버전 확인
nvim --version

# 최초 실행 — lazy.nvim이 플러그인 자동 설치
nvim

# 플러그인 매니저 UI 확인
:Lazy

# C 파일에서 LSP 연결 확인
:LspInfo

# Treesitter 설치 상태 확인
:TSInstallInfo
```

`.c` 파일을 열었을 때 구문 강조가 동작하고, `#inc` 입력 시 `#include` 제안이 뜨면 설정이 완료된 것이다.

---

## 참고

- [Neovim 공식 문서 — LSP](https://neovim.io/doc/user/lsp.html)
- [lazy.nvim GitHub](https://github.com/folke/lazy.nvim)
- [mason.nvim GitHub](https://github.com/mason-org/mason.nvim)
- [mason-lspconfig.nvim GitHub](https://github.com/mason-org/mason-lspconfig.nvim)
- [nvim-lspconfig GitHub](https://github.com/neovim/nvim-lspconfig)
- [blink.cmp GitHub](https://github.com/saghen/blink.cmp)
- [nvim-treesitter GitHub](https://github.com/nvim-treesitter/nvim-treesitter)
- [Neovim 0.11 LSP 변경사항](https://gpanders.com/blog/whats-new-in-neovim-0-11/)
