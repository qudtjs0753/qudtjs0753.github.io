---
title: "Neovim 생산성 도구 세팅 — 검색, 파일 트리, 테마, 키맵, 자동명령"
date: 2026-02-26 07:33:00 +0900
categories: [DevEnv]
tags: [neovim, lazy-loading]
---

[이전 글](/posts/neovim-c-dev-setup/)에서 Neovim의 핵심 기능(LSP, 자동완성, 구문 강조)을 설정했다. 이번 글에서는 **실제로 코딩할 때 편한 환경**을 만드는 나머지 설정을 정리한다.

## 최종 디렉토리 구조

```
~/dotfiles/.config/nvim/
├── init.lua                    # 진입점
└── lua/
    ├── config/
    │   ├── options.lua          # 에디터 기본설정
    │   ├── lazy.lua             # lazy.nvim 부트스트랩
    │   ├── keymaps.lua          # 글로벌 키바인딩       ← 이번 글
    │   └── autocmds.lua         # 자동명령              ← 이번 글
    └── plugins/
        ├── lsp.lua              # LSP (이전 글)
        ├── cmp.lua              # 자동완성 (이전 글)
        ├── treesitter.lua       # 구문 강조 (이전 글)
        ├── telescope.lua        # 퍼지 파인더           ← 이번 글
        ├── explorer.lua         # 파일 트리             ← 이번 글
        ├── ui.lua               # 테마 + 상태바         ← 이번 글
        ├── editor.lua           # 편집 보조 유틸리티     ← 이번 글
        ├── gitsigns.lua         # git 변경 표시         ← 이번 글
        └── markdown.lua         # Markdown 렌더링       ← 이번 글
```

### 파일 간 관계

```
init.lua
  ├─ require → config/options.lua    vim 기본 옵션 (leader key 등)
  ├─ require → config/lazy.lua       lazy.nvim 부트스트랩
  │                └─ plugins/ 디렉토리 자동 스캔
  │                     ├─ telescope.lua    (검색)
  │                     ├─ explorer.lua     (파일 트리)
  │                     ├─ ui.lua           (테마 + 상태바)
  │                     ├─ editor.lua       (편집 보조)
  │                     ├─ gitsigns.lua     (git 변경 표시)
  │                     └─ markdown.lua     (Markdown 렌더링)
  ├─ require → config/keymaps.lua    플러그인 무관 단축키
  └─ require → config/autocmds.lua   이벤트 기반 자동 동작
```

`config/` 파일들은 Neovim 자체 기능을 설정하고, `plugins/` 파일들은 플러그인별 설정을 담당한다. 새 플러그인 추가 = `plugins/`에 파일 1개 추가.

---

## 1. 퍼지 파인더 — telescope.lua

파일명, 코드 내용, 열린 버퍼를 실시간으로 검색하는 UI. `grep`이나 `find` 대신 에디터 안에서 바로 검색할 수 있다.

```lua
return {
  "nvim-telescope/telescope.nvim",
  dependencies = { "nvim-lua/plenary.nvim" },
  keys = {
    { "<leader>ff", function() require("telescope.builtin").find_files() end, desc = "파일명 검색" },
    { "<leader>fg", function() require("telescope.builtin").live_grep() end, desc = "코드 내용 검색" },
    { "<leader>fb", function() require("telescope.builtin").buffers() end, desc = "열린 버퍼 목록" },
  },

  opts = {
    defaults = {
      file_ignore_patterns = { "%.git/objects/" },
    },
    pickers = {
      find_files = {
        hidden = true,
      },
    },
  },
}
```

### 키바인딩

| 키 | 동작 |
|---|---|
| `Space f f` | 프로젝트 내 파일명 검색 |
| `Space f g` | 프로젝트 내 코드 내용 검색 (ripgrep 필요) |
| `Space f b` | 현재 열린 버퍼 목록 |

### opts 설정

| 옵션 | 위치 | 역할 |
|------|------|------|
| `file_ignore_patterns` | `defaults` | 모든 picker에서 지정 패턴을 제외. Lua 패턴이므로 `.`은 `%.`으로 이스케이프 |
| `hidden` | `pickers.find_files` | `.config/` 같은 숨김 파일/폴더를 검색 결과에 포함 |

`hidden = true`를 켜면 `.git/` 내부 파일도 노출되므로, `file_ignore_patterns`로 `.git/objects/`를 제외한다.

### 왜 이렇게 설정했나

- `keys = { ... }`로 키바인딩을 지정하면, **해당 키를 누를 때까지 플러그인을 로드하지 않는다** (lazy loading). Neovim 시작 시간에 영향을 주지 않는다.
- `function() require("telescope.builtin").find_files() end`처럼 함수로 감싸는 이유: 키를 누를 때까지 telescope 모듈 로드를 지연시키기 위해서다.
- `plenary.nvim`은 Neovim Lua 플러그인의 "표준 라이브러리" 같은 존재로, 비동기 I/O 등 유틸 함수를 제공한다.
- `live_grep`은 내부적으로 **ripgrep**을 사용하므로 `sudo apt install ripgrep`이 필요하다.

### Neovim nightly 사용 시 주의

Neovim 0.12(nightly)에서 `vim.treesitter.language.ft_to_lang()`이 제거되었다. Telescope 버전을 `tag = "0.1.8"`처럼 고정하면 구버전이 이 함수를 호출하면서 에러가 발생한다:

```
attempt to call field 'ft_to_lang' (a nil value)
```

`tag` 줄을 제거하고 `:Lazy sync`로 최신 버전을 받으면 해결된다. `tag`를 고정하면 `:Lazy sync`를 해도 해당 태그에 머무르기 때문에 업데이트가 되지 않는다.
---

## 2. 파일 트리 — explorer.lua

VS Code의 좌측 사이드바처럼 디렉토리 구조를 보여주는 패널.

```lua
return {
  "nvim-tree/nvim-tree.lua",
  dependencies = { "nvim-tree/nvim-web-devicons" },
  keys = {
    { "<leader>e", "<cmd>NvimTreeToggle<cr>", desc = "파일 트리 토글" },
  },
  opts = {
    disable_netrw = true,
    filters = { dotfiles = false },
    git = { enable = true },
    renderer = {
      icons = {
        show = {
          file = true,
          folder = true,
          git = true,
        },
      },
      highlight_git = true,
    },
  },
}
```

### 주요 설정

| 옵션 | 값 | 의미 |
|------|---|------|
| `disable_netrw` | `true` | Vim 내장 파일 탐색기를 끄고 nvim-tree가 대체 |
| `filters.dotfiles` | `false` | `.gitignore` 같은 숨김 파일도 표시 |
| `git.enable` | `true` | 파일 옆에 git 상태 아이콘(✗, ✓ 등) 표시 |
| `highlight_git` | `true` | git 상태에 따라 파일명 색상 변경 |

`nvim-web-devicons`가 파일 종류별 아이콘(, ,  등)을 보여준다. 터미널에 **Nerd Font**가 설치되어 있어야 아이콘이 깨지지 않는다.

---

## 3. 테마 + 상태바 — ui.lua

### gruvbox 테마

Starship 프롬프트에서 이미 gruvbox_dark를 사용하고 있어서, Neovim도 gruvbox로 통일했다.

```lua
{
  "ellisonleao/gruvbox.nvim",
  priority = 1000,
  config = function()
    require("gruvbox").setup({
      contrast = "hard",
    })
    vim.cmd.colorscheme("gruvbox")
  end,
},
```

`priority = 1000`은 lazy.nvim에게 "이 플러그인을 가장 먼저 로드해라"라는 뜻이다. 테마가 늦게 로드되면 화면이 잠깐 기본 색상으로 번쩍이는 현상(flash)이 생긴다.

### lualine 상태바

하단에 현재 상태 정보를 표시하는 바. 6개 섹션(A~C 왼쪽, X~Z 오른쪽)으로 나뉜다.

```lua
{
  "nvim-lualine/lualine.nvim",
  dependencies = { "nvim-tree/nvim-web-devicons" },
  opts = {
    options = {
      theme = "gruvbox",
    },
    sections = {
      lualine_a = { "location" },
      lualine_b = { "branch" },
      lualine_c = { "filename" },
      lualine_x = { "filetype" },
      lualine_y = { "progress" },
      lualine_z = { "mode" },
    },
  },
},
```

```
┌──────────────────────────────────────────────────┐
│ location │ branch │ filename    filetype │ % │ MODE │
└──────────────────────────────────────────────────┘
```

---

## 4. 편집 보조 — editor.lua

자주 쓰는 편집 기능 3개를 하나의 파일에 묶었다.

```lua
return {
  {
    "windwp/nvim-autopairs",
    event = "InsertEnter",
    opts = {},
  },
  {
    "numToStr/Comment.nvim",
    event = "BufReadPost",
    opts = {},
  },
  {
    "folke/which-key.nvim",
    event = "VeryLazy",
    opts = {},
  },
}
```

| 플러그인 | 역할 | 사용법 |
|----------|------|--------|
| nvim-autopairs | 괄호 자동 완성 | `(` 입력 → `()` 자동 생성 |
| Comment.nvim | 주석 토글 | `gcc`로 현재 줄 주석 처리/해제 |
| which-key.nvim | 키바인딩 가이드 | `Space` 누르면 등록된 키 목록 팝업 |

### lazy loading의 event 패턴

3개 플러그인의 `event` 값이 제각각인데, 이것이 lazy loading의 핵심이다:

| event | 의미 | 해당 플러그인 |
|-------|------|-------------|
| `InsertEnter` | Insert 모드 진입 시 | autopairs (타이핑할 때만 필요) |
| `BufReadPost` | 파일을 열었을 때 | Comment |
| `VeryLazy` | 모든 플러그인 로드 후 | which-key (다른 플러그인의 키 정보를 모아야 함) |

**필요한 시점에만 로드**하여 Neovim 시작 시간을 최소화한다.

---

## 5. Git 변경 표시 — gitsigns.lua

VS Code에서 파일을 수정하면 에디터 왼쪽에 초록/파랑/빨강 막대가 자동으로 표시된다. gitsigns.nvim으로 Neovim에서도 동일한 경험을 할 수 있다. 별도 파일로 분리하여 관리한다.

```lua
return {
  "lewis6991/gitsigns.nvim",
  event = { "BufReadPre", "BufNewFile" },
  opts = {
    attach_to_untracked = true,
  },
}
```

### 주요 옵션

| 옵션 | 기본값 | 설명 |
|------|--------|------|
| `attach_to_untracked` | `false` | `true`로 설정하면 git에 아직 추가하지 않은 새 파일도 전체 초록색으로 표시한다 |

### 기본 제공 명령어

| 명령어 | 동작 |
|--------|------|
| `:Gitsigns next_hunk` | 다음 변경 블록으로 이동 |
| `:Gitsigns prev_hunk` | 이전 변경 블록으로 이동 |
| `:Gitsigns preview_hunk` | 현재 변경 내용 미리보기 |
| `:Gitsigns blame_line` | 현재 줄의 git blame 표시 |

### 표시가 안 나올 때 확인할 것

gitsigns는 **git 레포 안의 파일**에서만 동작한다. 표시가 안 나오면:

1. `:pwd`로 현재 디렉토리가 git 레포 안인지 확인
2. 열고 있는 파일에 실제 변경사항이 있는지 확인 (커밋 대비 diff가 있어야 표시됨)
3. `:lua print(vim.inspect(vim.b.gitsigns_status_dict))`로 gitsigns가 버퍼에 정상 연결됐는지 확인

`nil`이 나오면 git 레포를 인식하지 못한 것이고, `gitdir`, `head`, `root` 등이 나오면 정상이다. 변경사항이 없는 파일이라 표시할 것이 없는 경우이니 수정된 파일을 열어본다.

---

## 6. Markdown 렌더링 — markdown.lua

render-markdown.nvim은 Markdown 파일을 **버퍼 안에서 인라인 렌더링**해주는 플러그인이다. 제목에 배경색이 입혀지고, 코드 블록에 테두리가 생겨 가독성이 크게 올라간다.

```lua
return {
  "MeanderingProgrammer/render-markdown.nvim",
  ft = "markdown",
  dependencies = {
    "nvim-treesitter/nvim-treesitter",
  },
  opts = {
    heading = {
      icons = { '# ', '## ', '### ', '#### ', '##### ', '###### ' },
      width = 'block',
    },
    code = {
      width = 'block',
      border = 'thick',
    },
  },
}
```

### 사전 준비

Treesitter의 `markdown`과 `markdown_inline` 파서가 필요하다:

```vim
:TSInstall markdown markdown_inline
```

이미 `treesitter.lua`의 `ensure_installed`에 `"markdown"`이 있다면 `markdown_inline`만 추가하면 된다.

### 주요 옵션

| 옵션 | 값 | 설명 |
|------|---|------|
| `ft` | `"markdown"` | Markdown 파일을 열 때만 로드 (lazy loading) |
| `heading.icons` | `{ '# ', '## ', ... }` | 헤딩 앞에 표시할 텍스트. 기본값은 이모지인데, 텍스트 기호로 변경했다 |
| `heading.width` | `'block'` | 헤딩 배경색을 텍스트 길이만큼만 표시 (`'full'`이면 줄 끝까지) |
| `code.width` | `'block'` | 코드 블록 배경을 코드 길이에 맞춤 |
| `code.border` | `'thick'` | 코드 블록 위아래에 두꺼운 구분선 표시 |

---

## 7. 글로벌 키바인딩 — keymaps.lua

플러그인과 무관하게 **Neovim 자체 기능**에 대한 단축키를 등록한다. 플러그인 전용 키바인딩(telescope의 `Space ff` 등)은 각 플러그인 파일에 있다.

```lua
local map = vim.keymap.set

-- 창 이동: Ctrl + h/j/k/l로 분할 창 간 이동
map("n", "<C-h>", "<C-w>h", { desc = "왼쪽 창으로 이동" })
map("n", "<C-j>", "<C-w>j", { desc = "아래쪽 창으로 이동" })
map("n", "<C-k>", "<C-w>k", { desc = "위쪽 창으로 이동" })
map("n", "<C-l>", "<C-w>l", { desc = "오른쪽 창으로 이동" })

-- 버퍼 이동: Tab/Shift+Tab으로 열린 파일 간 전환
map("n", "<Tab>", "<cmd>bnext<cr>", { desc = "다음 버퍼" })
map("n", "<S-Tab>", "<cmd>bprevious<cr>", { desc = "이전 버퍼" })

-- ESC로 검색 하이라이트 끄기
map("n", "<Esc>", "<cmd>nohlsearch<cr>", { desc = "검색 하이라이트 해제" })

-- Visual 모드에서 선택 영역 위아래 이동
map("v", "J", ":m '>+1<cr>gv=gv", { desc = "선택 영역 아래로 이동" })
map("v", "K", ":m '<-2<cr>gv=gv", { desc = "선택 영역 위로 이동" })
```

### 키바인딩 배치 원칙

`vim.keymap.set`의 첫 번째 인자가 **모드**를 결정한다:

| 모드 | 의미 |
|------|------|
| `"n"` | Normal 모드 — 기본 상태, 명령을 입력하는 모드 |
| `"v"` | Visual 모드 — 텍스트를 선택한 상태 |
| `"i"` | Insert 모드 — 텍스트를 입력하는 상태 |

키바인딩이 어디에 있는지 정리:

| 위치 | 내용 | 이유 |
|------|------|------|
| `keymaps.lua` | `Ctrl+h`, `Tab`, `Esc` 등 | Neovim 내장 기능 재매핑 |
| `telescope.lua` | `Space ff`, `Space fg` 등 | telescope 플러그인 전용 |
| `explorer.lua` | `Space e` | nvim-tree 플러그인 전용 |
| `lsp.lua` | `gd`, `K`, `Space rn` 등 | LSP 전용 (LspAttach 이벤트 시 등록) |

---

## 8. 자동명령 — autocmds.lua

특정 이벤트가 발생하면 자동으로 실행되는 동작을 정의한다. "X가 일어나면 Y를 해라" 패턴.

```lua
-- yank(복사) 시 하이라이트
vim.api.nvim_create_autocmd("TextYankPost", {
  callback = function()
    vim.hl.on_yank({ timeout = 300 })
  end,
})

-- treesitter 구문 강조 활성화
vim.api.nvim_create_autocmd("FileType", {
  callback = function()
    pcall(vim.treesitter.start)
  end,
})

-- 파일 저장 시 trailing whitespace 제거
vim.api.nvim_create_autocmd("BufWritePre", {
  pattern = "*",
  callback = function()
    local pos = vim.api.nvim_win_get_cursor(0)
    vim.cmd([[%s/\s\+$//e]])
    vim.api.nvim_win_set_cursor(0, pos)
  end,
})
```

### autocmd 이벤트 정리

| 이벤트 | 발생 시점 | 용도 |
|--------|----------|------|
| `TextYankPost` | `y`로 텍스트 복사 직후 | 복사한 영역을 0.3초간 강조 |
| `FileType` | 파일 타입이 결정될 때 | treesitter 하이라이팅 자동 적용 |
| `BufWritePre` | `:w`로 저장하기 직전 | 줄 끝 공백 자동 제거 |

`pcall(vim.treesitter.start)`에서 `pcall`은 "실행해보고 에러나면 무시"하는 래퍼다. 파서가 설치되지 않은 파일(`.txt` 등)을 열어도 에러 없이 넘어간다.

---

## 전체 플러그인 목록

| 플러그인 | 파일 | 역할 | lazy loading |
|----------|------|------|:---:|
| telescope.nvim | telescope.lua | 퍼지 검색 | keys |
| nvim-tree.lua | explorer.lua | 파일 트리 | keys |
| gruvbox.nvim | ui.lua | 색상 테마 | 즉시 (priority 1000) |
| lualine.nvim | ui.lua | 상태바 | 즉시 |
| nvim-autopairs | editor.lua | 괄호 자동 완성 | InsertEnter |
| Comment.nvim | editor.lua | 주석 토글 | BufReadPost |
| gitsigns.nvim | gitsigns.lua | git 변경 표시 | BufReadPre, BufNewFile |
| which-key.nvim | editor.lua | 키바인딩 가이드 | VeryLazy |
| render-markdown.nvim | markdown.lua | Markdown 인라인 렌더링 | ft (markdown) |

---

## 참고

- [telescope.nvim GitHub](https://github.com/nvim-telescope/telescope.nvim)
- [nvim-tree.lua GitHub](https://github.com/nvim-tree/nvim-tree.lua)
- [gruvbox.nvim GitHub](https://github.com/ellisonleao/gruvbox.nvim)
- [lualine.nvim GitHub](https://github.com/nvim-lualine/lualine.nvim)
- [which-key.nvim GitHub](https://github.com/folke/which-key.nvim)
- [gitsigns.nvim GitHub](https://github.com/lewis6991/gitsigns.nvim)
- [render-markdown.nvim GitHub](https://github.com/MeanderingProgrammer/render-markdown.nvim)
