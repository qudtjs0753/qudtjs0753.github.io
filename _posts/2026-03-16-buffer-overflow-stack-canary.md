---
title: "버퍼 오버플로우와 스택 카나리 — C/C++에서의 메모리 보안"
date: 2026-03-16 15:30:00 +0900
categories: [Programming]
tags: [security, memory, stack, c++]
---

알고리즘 문제를 풀다가 `char direction[3]`에 4바이트를 쓰는 실수를 했다.
결국 `direction[10]`으로 수정해서 통과했지만, 왜 작은 배열 크기가 문제였는지, 그리고 현대 컴파일러가 어떻게 이런 공격을 막는지 정리하게 되었다.

---

## 스택 메모리의 구조

먼저 함수가 실행될 때 스택이 어떻게 구성되는지 이해해야 한다.

### 함수 호출과 스택 프레임

함수 호출 시 CPU는 다음과 같이 동작한다:

```
높은 주소 (함수 시작)
┌──────────────────┐
│ [Return Address] │  ← 함수 반환 주소
├──────────────────┤
│ [RBP]            │  ← 이전 함수의 Base Pointer
├──────────────────┤
│ 로컬 변수들      │  ← 현재 함수의 변수들
├──────────────────┤
│ (오버플로우 위험)│
└──────────────────┘
낮은 주소 (RSP - Stack Pointer)
```

**RBP (Register Base Pointer)**는 함수의 기준점이다.
- 함수 시작 시 설정되어 함수 종료까지 고정
- 컴파일러가 각 변수의 오프셋(RBP - N)을 계산해서 접근
- LIFO 구조지만, 함수 내에서는 오프셋으로 자유롭게 접근 가능

### 변수 선언 순서와 메모리 배치

```cpp
void solve() {
    int query;           // RBP - 4
    int queryPos;        // RBP - 8
    int queryVal;        // RBP - 12
    char direction[3];   // RBP - 15 (가장 낮은 주소)
}
```

스택은 **높은 주소에서 낮은 주소로 자란다** (downward growth).
따라서 마지막에 선언된 변수가 가장 낮은 주소에 위치한다.

---

## 버퍼 오버플로우는 어떻게 발생하는가?

### 문제의 코드

```cpp
char direction[3];
scanf("%s", direction);  // "row" 또는 "col" 입력
```

C의 문자열은 항상 null terminator(`'\0'`)로 끝나야 하므로:
- `direction[0]` = `'r'` (입력값)
- `direction[1]` = `'o'` (입력값)
- `direction[2]` = `'w'` (입력값)
- `direction[3]` = `'\0'` ← **scanf가 자동으로 추가! 배열 범위를 벗어남!**

"row"는 3글자지만, C 문자열로는 `"row\0"` = 4바이트가 필요합니다.

### scanf는 경계 검사를 하지 않는다

```cpp
scanf("%s", direction);
// scanf는 입력 크기를 제한하지 않고 계속 메모리에 쓴다!
```

`scanf`는 문자열 길이를 검사하지 않으므로, 배열 크기를 초과해서 메모리에 계속 쓴다.

### 실제 메모리 배치와 침범

```
높은 주소
┌──────────────────┐
│ query            │
├──────────────────┤
│ queryPos         │
├──────────────────┤
│ queryVal         │
├──────────────────┤
│ direction[0]='r' │
│ direction[1]='o' │
│ direction[2]='w' │
├──────────────────┤
│ direction[3]='\0'│ ← 배열 범위 초과!
├──────────────────┤
│ [Stack Canary]   │ ← 손상됨!
└──────────────────┘
낮은 주소
```

direction[3]의 `'\0'`은 direction 바로 아래(낮은 주소)의 **스택 카나리를 침범**합니다.

---

## 스택 카나리(Stack Canary)란?

### 이름의 유래

**광산의 카나리새:**
- 과거 광부들이 독성 가스를 감지하기 위해 카나리새를 광산에 보냈다
- 카나리가 죽으면 = 위험 신호
- 마찬가지로 스택 카나리가 손상되면 = 오버플로우 신호

### 동작 원리

현대 컴파일러(gcc, clang)는 기본적으로 스택 보호를 활성화한다.

```
높은 주소
┌──────────────────────┐
│ [RBP]                │
├──────────────────────┤
│ query                │
├──────────────────────┤
│ queryPos             │
├──────────────────────┤
│ queryVal             │
├──────────────────────┤
│ direction[3]         │
├──────────────────────┤
│ [Stack Canary]       │ ← 무작위 값 (0xDEADBEEF 같은)
│                      │   컴파일러가 자동 삽입!
├──────────────────────┤
│ [Return Address]     │
└──────────────────────┘
낮은 주소
```

### 함수 종료 시 검증

```cpp
}  // 함수 끝남
   // 1. Stack Canary 값 확인
   // 2. 변했으면 → "stack smashing detected!"
   // 3. 프로그램 강제 종료
   // 4. 안 변했으면 → 정상 종료
```

---

## 실제 테스트 — 버퍼 오버플로우 감지

### 극단적인 오버플로우 (카나리 손상 발생)

다음 코드로 테스트했다:

```cpp
#include <stdio.h>
#include <string.h>

int main() {
    char buffer[5];
    int important = 0xDEADBEEF;

    printf("Before: important = 0x%X\n", important);
    strcpy(buffer, "0123456789");  // 11바이트를 5바이트 버퍼에
    printf("After: important = 0x%X\n", important);

    return 0;
}
```

**실행 결과:**
```
*** stack smashing detected ***: terminated
(core dumped)
```

카나리가 손상되어 프로그램이 강제 종료된다.

### 작은 오버플로우 (카나리 미손상, 다른 값 손상)

`char direction[3]`에 단 4바이트(null terminator 포함)만 쓰는 경우를 살펴보자.

**로컬 환경에서의 테스트 결과:**

다양한 컴파일 옵션(`-O2`, `-O3`, `-Ofast`, `-static`, `-march=native` 등)으로 테스트했지만,
`direction[3]`과 `direction[10]` 모두 동일한 결과를 출력했다.

**Baekjoon에서의 제출 결과:**

- `direction[3]` → Wrong Answer (2회 제출)
- `direction[10]` → Accepted

**원인 추정 (확정 불가능):**

1. **메모리 배치의 차이**
   - 로컬에서는 `direction` 이후 메모리가 여유롭거나 중요하지 않은 값이 있을 수 있음
   - Baekjoon 서버에서는 `direction` 바로 옆(낮은 주소)에 중요한 변수나 제어 정보가 있을 수 있음

2. **다른 libc/gcc 버전**
   - `scanf`의 내부 동작이 다를 수 있음
   - 컴파일러 최적화 전략이 다를 수 있음

3. **특수한 컴파일 옵션**
   - 로컬에서 시도하지 못한 설정이 있을 수 있음

**결론:**

정확한 원인은 Baekjoon 서버의 정확한 OS, gcc/g++ 버전, libc, 컴파일 옵션을 알지 못하는 이상 확정할 수 없다.
하지만 **`direction[3]이 버퍼 오버플로우를 일으킨다는 사실은 변하지 않으며**,
환경에 따라 결과가 달라질 수 있다는 것이 **오버플로우의 위험성을 잘 보여준다.**

---

## 공격자 관점: 왜 카나리가 필요한가?

### 카나리 없는 시대 (1990년대)

```
공격 목표: return address 변조
           ↓ (임의 코드 실행)

buffer overflow
    ↓
direction[3]
    ↓
direction[4]
    ↓
... (계속 오버플로우)
    ↓
[Return Address] 변조 가능! ✓
```

### 카나리가 있는 현대

```
buffer overflow
    ↓
direction[3]
    ↓
[Stack Canary] ← 무작위값, 예측 불가능
                 손상됨 → 감지! → 프로그램 종료 ✗
```

공격자는 카나리를 알아야 return address에 접근할 수 있는데,
카나리가 **무작위값**이라 브루트포스 불가능하다.

---

## 안전한 코드 작성

### 좋은 예: 안전한 배열 크기

```cpp
char direction[10];  // 충분한 크기
scanf("%s", direction);  // 안전
```

### 더 좋은 예: 크기 제한

```cpp
char direction[10];
scanf("%9s", direction);  // 최대 9글자 + null terminator
```

### 또는 안전한 함수 사용

```cpp
#include <stdio.h>

char direction[10];
fgets(direction, sizeof(direction), stdin);
// 자동으로 크기 제한, null terminator 추가
```

| 함수 | 안전성 | 비고 |
|------|-------|------|
| `scanf("%s")` | ❌ 위험 | 크기 제한 없음 |
| `scanf("%9s")` | ✓ 안전 | 크기 제한 가능 |
| `fgets()` | ✓ 안전 | 권장 |
| `strcpy()` | ❌ 위험 | 크기 제한 없음 |
| `strncpy()` | ✓ 안전 | 크기 제한 가능 |

---

## 컴파일러 설정

### 카나리는 기본으로 활성화됨

```bash
# 기본값: 카나리 활성화
g++ -o program program.cpp

# 명시적으로 비활성화 (위험!)
g++ -fno-stack-protector -o program program.cpp

# 강력한 보호
g++ -fstack-protector-all -o program program.cpp
```

**중요:** 현대 gcc/g++ (특히 Ubuntu)는 기본적으로 `-fstack-protector`가 활성화되어 있습니다.
따라서 일반적인 `g++ -o program program.cpp` 명령으로 컴파일해도 스택 카나리 보호가 자동으로 적용됩니다.

- `-fstack-protector`: 위험한 함수만 보호 (기본값)
- `-fstack-protector-all`: 모든 함수 보호 (성능 저하)
- `-fno-stack-protector`: 보호 비활성화 (권장하지 않음)

---

## 정리

| 개념 | 설명 |
|------|------|
| **버퍼 오버플로우** | 배열 범위를 벗어난 메모리 쓰기 |
| **스택 카나리** | 메모리 손상을 감지하는 보호값 |
| **Return Address** | 함수 종료 후 돌아갈 주소 |
| **RBP** | 함수 내 변수의 기준점 |
| **scanf의 위험성** | 입력 크기를 제한하지 않음 |

**핵심:**
- 배열 크기는 충분히 할당하기
- `scanf` 대신 `fgets` 사용하기
- `strcpy` 대신 `strncpy` 사용하기
- 현대 컴파일러의 카나리가 대부분의 공격을 막아주지만, 안전한 코딩이 기본

---

## 참고

이 글은 Baekjoon 21035번 문제를 풀며 발견한 버퍼 오버플로우 버그와,
이를 통해 학습한 스택 메모리 구조와 현대 보안 메커니즘을 정리한 것이다.

특히 인상적인 점은, 로컬에서는 재현되지 않은 오류가 온라인 저지에서는 발생했다는 것이다.
이는 **버퍼 오버플로우의 결과가 환경에 따라 다를 수 있다**는 중요한 교훈을 보여준다.
따라서 "지금 당장 crash가 나지 않으니까 안전하다"는 생각은 위험하며,
정확한 배열 크기 선언과 안전한 입력 함수 사용이 필수불가결하다.
