---
title: "Neovim Keymap 설정: noremap의 중요성 — 재귀 매핑 문제 해결하기"
date: 2026-03-12 21:30:00 +0900
categories: [Vim]
tags: [neovim, keymap, noremap]
---

Neovim에서 키 매핑을 설정할 때, `noremap` 옵션을 빼먹으면 예상치 못한 버그가 발생할 수 있다. 실제로 내 Neovim 설정에서 `gg`와 `G` 명령어를 커스텀하려고 했을 때, `noremap = true` 옵션을 빼먹었다가 재귀 매핑 루프 문제에 직면했다. 이 글은 그 경험을 정리한 것이다. 특히 기존 명령어를 기반으로 새로운 매핑을 만들 때 **재귀 매핑(recursive mapping)** 문제가 생길 수 있으며, 이를 해결하는 방법을 다룬다.

---

## 문제: 재귀 매핑으로 인한 무한 루프

### 상황 1: 기존 명령어를 조합한 매핑

Neovim의 기본 명령어 `gg`와 `G`를 다음과 같이 설정했다고 해보자:

```lua
local map = vim.keymap.set

-- noremap 없음 (위험!)
map("n", "gg", "gg0", { desc = "첫 줄의 맨 처음으로 이동" })
map("n", "G", "G$", { desc = "마지막 줄의 맨 마지막으로 이동" })
```

### 무한 루프가 발생하는 실행 과정

1. **사용자가 `gg` 입력**
2. Neovim이 매핑 검사: "gg가 매핑되어 있나?" → 예! `gg0`으로 실행
3. **`gg0`을 실행하려고 함**
4. 문자열 내의 `gg` 다시 확인: "gg가 매핑되어 있나?" → 예! 또 `gg0`으로 실행
5. 3번으로 돌아가서 **무한 반복...**
6. **Neovim 멈춤** (또는 오류 발생)

### 발생하는 오류 메시지

```
E169: Command too recursive
```

또는 사용자는 다음과 같은 증상을 경험한다:
- Neovim이 반응하지 않음 (먹통)
- CPU 사용률이 갑자기 치솟음
- Ctrl+C로 강제 종료해야 함

---

## 재귀 매핑이 발생하는 다른 사례들

### 예시 1: jk를 ESC로 매핑했을 때

**잘못된 코드:**
```lua
-- noremap 없음
map("i", "j", "k", { desc = "아래로 이동" })
map("i", "jk", "<Esc>", { desc = "Insert 모드 탈출" })
```

**실행 과정:**
1. Insert 모드에서 `jk` 입력
2. `jk`는 `<Esc>`로 매핑됨
3. 하지만 `j`도 매핑되어 있다!
4. `jk`를 풀어쓰면 `j` + `k` → `k` + `k` → 기대하지 않은 동작 발생

### 예시 2: 다른 매핑을 기반으로 한 매핑

**잘못된 코드:**
```lua
map("n", "w", "b", { desc = "뒤로 이동" })
map("n", "W", "w", { desc = "앞으로 이동" })
```

**실행 과정:**
1. `W` 입력
2. `W`는 `w`로 매핑됨
3. 하지만 `w`도 매핑되어 있음 (`b`로)
4. 결과: `W` → `w` → `b` (예상과 다름!)

---

## 해결책: noremap = true 추가하기

### noremap 옵션의 위치

`{ }`로 감싼 옵션 테이블에서 어디든 추가하면 된다:

```lua
local map = vim.keymap.set

-- 옵션 1: noremap만 지정
map("n", "gg", "gg0", { noremap = true })

-- 옵션 2: desc와 함께 사용
map("n", "gg", "gg0", { noremap = true, desc = "첫 줄의 맨 처음으로 이동" })

-- 옵션 3: desc가 먼저 와도 됨 (순서 상관없음)
map("n", "G", "G$", { desc = "마지막 줄의 맨 마지막으로 이동", noremap = true })

-- 옵션 4: 다른 옵션들과 함께
map("n", "jk", "<Esc>", {
  noremap = true,
  silent = true,
  desc = "Insert 모드 탈출"
})
```

### noremap = true일 때 동작

1. 사용자가 `gg` 입력
2. Neovim이 매핑된 `gg0`을 실행
3. `gg0` 내부의 `gg`는 **원본 명령어로 실행** (매핑 재확인 안 함!)
4. `gg` 명령어 실행 → 첫 줄의 첫 non-blank 문자로 이동
5. `0` 명령어 실행 → 현재 줄의 column 0으로 이동
6. **안전하게 완료** ✅

---

## 올바른 설정 예시들

### 상황별 올바른 설정

| 상황 | 잘못된 코드 | 올바른 코드 | noremap 필요? |
|------|---------|---------|--------|
| **기존 명령어 조합** | `map("n", "gg", "gg0")` | `map("n", "gg", "gg0", { noremap = true })` | ✅ 필수 |
| Insert 모드 탈출 | `map("i", "jk", "<Esc>")` | `map("i", "jk", "<Esc>", { noremap = true })` | ✅ 필수 |
| 창 이동 | `map("n", "<C-h>", "<C-w>h")` | `map("n", "<C-h>", "<C-w>h", { noremap = true })` | ✅ 필수 |
| 플러그인 명령어 | `map("n", "<leader>f", "Telescope find_files")` | `map("n", "<leader>f", "<cmd>Telescope find_files<cr>", { noremap = true })` | ✅ 필수 |
| Lua 함수 호출 | `map("n", "gd", function() ... end)` | `map("n", "gd", function() ... end, { noremap = true })` | ✅ 필수 |

---

## vim.keymap.set의 주요 옵션

```lua
map("n", "key", "result", {
  -- 매핑 재귀 금지 (대부분 true로 설정)
  noremap = true,

  -- 매핑 재귀 허용 (noremap과 반대, 거의 안 씀)
  remap = false,

  -- 명령 실행 메시지 숨기기
  silent = true,

  -- 매핑 설명 (which-key 플러그인에서 표시)
  desc = "설명",

  -- 특정 버퍼에만 적용
  buffer = 0,  -- 0 = 현재 버퍼
})
```

---

## 실제 사용 사례

### 내 keymaps.lua의 예시

```lua
local map = vim.keymap.set

-- ── 창 이동: Ctrl + h/j/k/l로 분할 창 간 이동 ──
map("n", "<C-h>", "<C-w>h", { noremap = true, desc = "왼쪽 창으로 이동" })
map("n", "<C-j>", "<C-w>j", { noremap = true, desc = "아래쪽 창으로 이동" })
map("n", "<C-k>", "<C-w>k", { noremap = true, desc = "위쪽 창으로 이동" })
map("n", "<C-l>", "<C-w>l", { noremap = true, desc = "오른쪽 창으로 이동" })

-- ── ESC로 검색 하이라이트 끄기 ──
map("n", "<Esc>", "<cmd>nohlsearch<cr>", { noremap = true, desc = "검색 하이라이트 해제" })

-- ── 파일 처음/끝으로 이동 (column 지정) ──
map("n", "gg", "gg0", { noremap = true, desc = "첫 줄의 맨 처음으로 이동" })
map("n", "G", "G$", { noremap = true, desc = "마지막 줄의 맨 마지막으로 이동" })
```

---

## Vimscript와의 비교

### Vimscript에서는 자동으로 처리됨

Vimscript를 사용하면 명령어 이름 자체에 `noremap` 개념이 포함되어 있다:

```vim
" Vimscript (Vim 전통 방식)
nnoremap gg gg0           " nn = Normal + noremap (자동으로 재귀 매핑 금지)
inoremap jk <Esc>        " in = Insert + noremap (자동으로 재귀 매핑 금지)
vnoremap J :m '>+1<cr>gv=gv  " vn = Visual + noremap (자동으로 재귀 매핑 금지)

" 반면, noremap 없으면 위험
map gg gg0         " ❌ 재귀 매핑 허용 (위험! gg가 무한 루프됨)
nmap gg gg0        " ❌ Normal 모드지만 여전히 위험
```

Vimscript에서는 명령어 이름으로 이미 `noremap = true`인지 아닌지가 결정되어 있다!

### Lua API에서는 명시적으로 지정해야 함

Lua의 `vim.keymap.set()` 함수는 모든 경우에 동일한 API이므로, `noremap` 여부를 **직접 옵션으로 써줘야** 한다:

```lua
local map = vim.keymap.set

-- 기본값은 noremap = false (재귀 매핑 활성화 - 위험!)
map("n", "gg", "gg0")

-- noremap = true를 명시적으로 지정해야 안전!
map("n", "gg", "gg0", { noremap = true })
```

**"명시적으로"의 의미:**
- Vimscript: 명령어 이름이 자동으로 결정함 (`nnoremap` vs `map`)
- Lua: 개발자가 직접 옵션을 써줘야 함 (`{ noremap = true }`)

따라서 Lua를 쓸 때는 **매번 `{ noremap = true }`를 직접 작성**해야 한다!

---

## 핵심 정리

| 항목 | 설명 |
|------|------|
| **noremap = true의 의미** | "매핑 내부의 키들을 다시 매핑하지 마" |
| **기본값** | `noremap = false` (재귀 매핑 활성화) |
| **사용 시기** | 거의 항상 (특별한 경우만 false) |
| **오류 증상** | `E169: Command too recursive` 또는 Neovim 먹통 |
| **해결책** | `{ noremap = true }` 추가 |
| **위치** | 마지막 매개변수 `{}` 테이블 안에 어디든 |

---

## 결론

> **기본 규칙: 특별한 이유가 없으면 항상 `noremap = true`를 사용하자.**

Neovim Lua 설정에서 키 매핑을 할 때는:

```lua
local map = vim.keymap.set

-- 항상 이런 식으로 작성
map("n", "키", "실행", { noremap = true, desc = "설명" })
```

이 패턴을 기억하면 재귀 매핑 문제로 인한 예상치 못한 오류를 방지할 수 있다!

---

## 참고 자료

- [Neovim Documentation: vim.keymap.set](https://neovim.io/doc/user/lua.html#vim.keymap.set)
- Vim 튜토리얼: `:help map-modes`
