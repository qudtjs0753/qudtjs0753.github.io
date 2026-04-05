---
title: 제곱근 계산
date: 2026-04-05 00:00:00 +0900
categories: [Algorithm]
tags: [binary-search, floating-point]
math: true
---

> 이 글은 AI Agent가 작성했습니다.

라이브러리 없이 제곱근 함수를 직접 구현해본 기록. 나는 Binary Search를 활용해 구현했다.
많이 사용하는 뉴턴-랩슨 방법과 구현 코드는 아직 정확하게 이해 못했다. 추후 정리예정

---

## 문제 개요

[Baekjoon 25991](https://www.acmicpc.net/problem/25991) — n개의 정육면체 용기의 한 변의 길이 $c$가 주어질 때, 모든 부피를 합친 것과 동일한 부피의 정육면체 한 변의 길이를 구하라.

$$\text{answer} = \sqrt[3]{\sum_{i=1}^{n} c_i^3}$$

- $1 \leq n \leq 10^5$, $1 \leq c \leq 10^9$
- 절대 오차 또는 상대 오차 $\leq 10^{-6}$

---

## 문제: 부동소수점 한계로 인한 무한루프

종료 조건을 `right - left <= 1e-6`만으로 처리하려니 시간 초과가 발생했다.

### IEEE 754와 ULP

double은 64비트로 실수를 표현한다.

```
[부호 1bit][지수 11bit][가수 52bit]
```

가수부가 52비트이므로 유효 자릿수는 약 15~16자리다. 어떤 수 $x$ 근처에서 연속한 두 double 사이의 간격을 **ULP(Unit in the Last Place)**라고 한다.

$$\text{ULP}(x) \approx x \times 2^{-52}$$

| $x$ | ULP 크기 |
|-----|---------|
| $1.0$ | $2.2 \times 10^{-16}$ |
| $10^{6}$ | $1.2 \times 10^{-10}$ |
| $4.6 \times 10^{10}$ | $\approx 1.0 \times 10^{-5}$ |

이 문제에서 최대 입력($n = 10^5$, $c = 10^9$)이면 합이 최대 $10^{32}$이고 세제곱근은 최대 $\approx 4.6 \times 10^{10}$이다. 이 크기에서 ULP는 $\approx 10^{-5}$이다.

### 무한루프 발생 원리

이진 탐색이 진행되어 `left`와 `right`가 모두 $4.6 \times 10^{10}$ 근처가 되면:

```
right - left ≈ 1e-5  (이 크기에서의 ULP)
mid = (left + right) / 2.0
    → 덧셈 단계에서 하위 비트 소실
    → mid == left 또는 mid == right로 반올림됨
```

이 시점부터 `left = mid`나 `right = mid`를 해도 구간이 줄어들지 않는다. `right - left <= 1e-6`은 이 크기의 double로는 **물리적으로 표현 불가능한 상태**를 요구하는 것이다.

---

## 해결: 상대 오차 기반 종료 조건

문제 조건이 "절대 오차 **또는** 상대 오차 $\leq 10^{-6}$"이므로 종료 조건을 두 가지로 구성한다.

```cpp
if (right - left <= MIN_DIFF ||
    (right - left) / mid <= MIN_DIFF)
    return mid;
```

| 조건 | 의미 | 언제 트리거 |
|------|------|------------|
| `right - left <= 1e-6` | 절대 오차 기준 | 답이 작을 때 |
| `(right - left) / mid <= 1e-6` | 상대 오차 기준 | 답이 클 때 |

### 왜 right - left <= 1e-6이면 절대 오차가 $10^{-6}$ 이하일까
mid는 right-left 사이에 있음이 반드시 보장되기 때문이다.

### 절대 오차 vs 상대 오차

정답이 $x^* = 4.6 \times 10^{10}$이고 내 답이 $0.00001$ 틀렸다면:

$$\text{절대 오차} = 10^{-5} \quad \leftarrow \text{기준 위반}$$
$$\text{상대 오차} = \frac{10^{-5}}{4.6 \times 10^{10}} \approx 2 \times 10^{-16} \quad \leftarrow \text{기준 통과}$$

이진 탐색이 ULP 한계에 도달했을 때 절대 오차는 $\approx 10^{-5}$로 기준을 넘을 수 있지만, 상대 오차는 $\approx 2^{-52} \approx 10^{-16}$으로 사실상 machine epsilon 수준이다.

---

## 최종 코드

```cpp
#include <stdio.h>
#define MIN_DIFF 1e-6

class Solver {
public:
  void solve() {
    int N;
    double floatNums, sum = 0;
    scanf("%d", &N);
    for (int i = 0; i < N; i++) {
      scanf("%lf", &floatNums);
      sum += floatNums * floatNums * floatNums;
    }
    printf("%lf", mySqrt(sum));
  }

  double mySqrt(double powerOfThree) {
    double left = 0;
    double right = (powerOfThree < 1) ? 1 : powerOfThree;
    double mid;

    while (true) {
      mid = (left + right) / 2.0;

      if (right - left <= MIN_DIFF ||
          (right - left) / mid <= MIN_DIFF)
        return mid;

      if (mid * mid * mid < powerOfThree)
        left = mid;
      else
        right = mid;
    }
  }
};

int main() {
  Solver solver;
  solver.solve();
  return 0;
}
```

---

## 참고: glibc의 실제 cbrt 구현

glibc는 이진 탐색 없이 세 단계만에 세제곱근을 계산한다.

### 뉴턴-랩슨법 원리

$f(x) = x^3 - T = 0$ 의 근을 구하는 문제다. 뉴턴법의 점화식은 다음과 같다.

$$x_{n+1} = x_n - \frac{f(x_n)}{f'(x_n)} = x_n - \frac{x_n^3 - T}{3x_n^2}$$

이를 정리하면:

$$x_{n+1} = \frac{2x_n}{3} + \frac{T}{3x_n^2} = x_n \cdot \frac{x_n^3 + 2T}{2x_n^3 + T}$$

이차 수렴이므로 매 반복마다 정확한 자릿수가 2배가 된다. 좋은 초기값만 있으면 수 번 안에 수렴한다.

### 실제 코드 (glibc `s_cbrt.c`)

```c
#define CBRT2     1.2599210498948731648   // 2^(1/3)
#define SQR_CBRT2 1.5874010519681994748   // 2^(2/3)

// xe % 3 결과(0,1,2)에 대응하는 보정 계수
// 지수를 3으로 나눈 나머지에 따라 스케일 조정이 필요하기 때문
static const double factor[5] = {
  1.0 / SQR_CBRT2,  // xe % 3 == -2
  1.0 / CBRT2,      // xe % 3 == -1
  1.0,              // xe % 3 ==  0  (보정 불필요)
  CBRT2,            // xe % 3 ==  1
  SQR_CBRT2         // xe % 3 ==  2
};

double __cbrt(double x) {
  double xm, ym, u, t2;
  int xe;

  // x = xm * 2^xe 로 분리. xm은 [0.5, 1.0) 범위
  // 어떤 크기의 수든 동일한 범위에서 다항식 근사를 적용하기 위함
  xm = __frexp(fabs(x), &xe);

  // Inf, NaN, 0 처리 (frexp는 이 경우 xe=0을 반환)
  if (xe == 0 && fpclassify(x) <= FP_ZERO)
    return x + x;

  // xm ∈ [0.5, 1.0) 에서 세제곱근의 초기 근사값을 6차 다항식으로 계산
  // 계수는 Remez 알고리즘으로 최대 오차를 최소화하도록 사전에 결정된 값
  u = (0.354895765043919860
       + ((1.50819193781584896
       + ((-2.11499494167371287
       + ((2.44693122563534430
       + ((-1.83469277483613086
       + (0.784932344976639262
       - 0.145263899385486377 * xm) * xm) * xm) * xm) * xm) * xm)));

  // 뉴턴-랩슨 1회: x_{n+1} = u * (u^3 + 2*xm) / (2*u^3 + xm)
  // 다항식 근사가 이미 충분히 정확하여 단 1회로 double 전체 정밀도 도달
  t2 = u * u * u;
  ym = u * (t2 + 2.0 * xm) / (2.0 * t2 + xm);

  // factor로 지수 나머지(xe % 3) 보정 후, __ldexp로 지수 복원
  // cbrt(x) = cbrt(xm) * 2^(xe/3) 이므로
  ym = ym * factor[2 + xe % 3];
  return __ldexp(x > 0.0 ? ym : -ym, xe / 3);
}
```

### 핵심 아이디어 요약

| 단계 | 하는 일 | 이유 |
|------|---------|------|
| `__frexp` | $x = xm \times 2^{xe}$로 분리 | 다항식 근사 범위를 $[0.5, 1.0)$으로 고정 |
| 다항식 | $[0.5, 1.0)$에서 초기 근사값 계산 | 뉴턴법이 빠르게 수렴하려면 좋은 초기값이 필요 |
| 뉴턴 1회 | 오차를 제곱으로 줄임 | 이차 수렴 — 이미 정밀한 초기값이므로 1회로 충분 |
| `__ldexp` | $\times 2^{xe/3}$으로 스케일 복원 | 처음에 분리한 지수를 되돌림 |

| 방식 | 반복 횟수 | 정밀도 |
|------|----------|--------|
| 이진 탐색 | ~126회 | 종료 조건에 의존 |
| 뉴턴-랩슨 (좋은 초기값) | ~10회 | 이차 수렴 |
| glibc cbrt | 루프 없음 | IEEE 754 완전 정밀도 |

좋은 초기값이 얼마나 중요한지를 보여주는 구현이다.

---

## 참고 자료

- [glibc s_cbrt.c 소스](https://sourceware.org/git/?p=glibc.git;a=blob_plain;f=sysdeps/ieee754/dbl-64/s_cbrt.c)
- [Baekjoon 25991](https://www.acmicpc.net/problem/25991)
- [Wikipedia — Newton's method](https://en.wikipedia.org/wiki/Newton%27s_method)
- [Wikipedia — Unit in the last place](https://en.wikipedia.org/wiki/Unit_in_the_last_place)
- [Wikipedia — IEEE 754](https://en.wikipedia.org/wiki/IEEE_754)
