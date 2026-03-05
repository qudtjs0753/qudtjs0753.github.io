---
title: "Stateless RNN vs Stateful RNN: 차이점과 PyTorch 구현"
date: 2026-03-05 15:30:00 +0900
categories: [DeepLearning]
tags: [rnn]
---

## 요약

| | Stateless RNN | Stateful RNN |
|--|--------------|-------------|
| 배치 간 hidden state | 매번 0으로 초기화 | 이전 배치에서 이어받음 |
| 배치 순서 | 상관없음 (셔플 가능) | 반드시 순서 보장 |
| 구현 난이도 | 간단 (기본값) | 수동 관리 필요 |
| 장기 문맥 | 배치 내부만 | 배치 경계를 넘어 유지 |
| 대표 용도 | 문장 분류, 감성 분석 | 긴 시계열, 텍스트 생성 |

---

## Stateless와 Stateful의 구분

이 구분은 **배치(batch) 간에 hidden state를 유지하느냐 초기화하느냐**의 차이다.

### Stateless RNN

배치가 바뀔 때마다 **hidden state를 0으로 초기화**한다.

```
배치 1: [x1, x2, x3] -> h0=0에서 시작 -> h3 출력
배치 2: [x4, x5, x6] -> h0=0에서 시작 (h3 버림) -> h6 출력
```

각 배치가 **독립적인 시퀀스**로 처리된다. 배치 1에서 학습한 문맥이 배치 2로 전달되지 않는다.

### Stateful RNN

배치가 바뀌어도 **이전 배치의 마지막 hidden state를 다음 배치의 초기값으로 이어받는다.**

```
배치 1: [x1, x2, x3] -> h0=0에서 시작 -> h3 출력
배치 2: [x4, x5, x6] -> h0=h3에서 시작 -> h6 출력 (문맥 연속)
```

하나의 긴 시퀀스를 여러 배치로 잘라 넣어도 **문맥이 끊기지 않는다.**

---

## 왜 Stateful이 필요한가: 배치 하나에 시퀀스 전체가 안 들어갈 때

GPU 메모리에는 한계가 있어서, 긴 시퀀스를 한 번에 넣을 수 없는 경우가 생긴다.

```
전체 시계열: [x1, x2, x3, ..., x1000]  <- 한 번에 못 넣음

잘라서 넣기:
  배치 1: [x1 ~ x200]
  배치 2: [x201 ~ x400]
  배치 3: [x401 ~ x600]
  ...
```

### Stateless일 때

```
배치 1: [x1~x200]   -> h0=0에서 시작 -> x200까지의 문맥 학습
배치 2: [x201~x400] -> h0=0에서 시작 -> x1~x200의 정보 완전히 소실
```

배치 2 입장에서는 x201이 **시퀀스의 첫 번째 데이터**인 것처럼 처리된다. x1~x200에서 쌓인 패턴(추세, 문맥)을 전혀 모른다.

### Stateful일 때

```
배치 1: [x1~x200]   -> h0=0에서 시작 -> h200에 x1~x200 문맥 압축
배치 2: [x201~x400] -> h0=h200에서 시작 -> x1~x200 문맥 위에서 계속 학습
```

배치 2가 시작할 때 **이전 200스텝의 정보가 hidden state에 남아있다.** 그래서 x201을 처리할 때 앞선 흐름을 참고할 수 있다.

### 구체적 예시: 주가 예측

1년치 일별 주가(365일)를 100일씩 잘라 넣는다고 하자.

```
배치 1: 1~100일   -> 상승 추세 학습
배치 2: 101~200일 -> ???
```

**Stateless**: 배치 2는 101일째부터 갑자기 시작. 이전 100일이 상승이었는지 하락이었는지 모른다.

**Stateful**: 배치 2는 "지금까지 상승 추세였다"는 정보를 h100에서 이어받는다. 101일째 데이터를 그 추세 위에서 해석할 수 있다.

---

## 배치 크기를 크게 하면 되지 않을까?

이론적으로는 시퀀스 전체를 한 배치에 넣으면 Stateful이 필요 없다. 하지만 현실적 제약이 있다.

| 제약 | 설명 |
|------|------|
| GPU 메모리 | 시퀀스 길이에 비례해서 메모리 사용량 증가. RNN은 각 스텝의 중간 계산을 역전파용으로 저장해야 함 |
| 학습 속도 | 시퀀스가 길수록 역전파(BPTT) 경로가 길어져 기울기 소실/폭발 심화 |

**Stateful은 이 두 문제를 우회하는 실용적 해법**이다. 짧게 잘라서 메모리는 아끼면서, hidden state 전달로 긴 문맥은 유지한다.

---

## PyTorch 구현

### Stateless RNN

```python
import torch
import torch.nn as nn

class StatelessRNN(nn.Module):
    def __init__(self, input_size, hidden_size, output_size):
        super().__init__()
        self.rnn = nn.RNN(input_size, hidden_size, batch_first=True)
        self.fc = nn.Linear(hidden_size, output_size)

    def forward(self, x):
        # h0을 안 넘기면 자동으로 0으로 초기화
        out, _ = self.rnn(x)          # _ : 마지막 hidden state 버림
        out = self.fc(out[:, -1, :])   # 마지막 스텝만 사용
        return out

model = StatelessRNN(input_size=1, hidden_size=32, output_size=1)

# 매 배치마다 hidden state가 0에서 시작
for batch in dataloader:
    output = model(batch)  # 이전 배치의 기억 없음
```

핵심: `self.rnn(x)`에 h0을 안 넘기면 **매번 0으로 시작**한다. 이전 배치의 문맥은 완전히 사라진다.

### Stateful RNN

```python
import torch
import torch.nn as nn

class StatefulRNN(nn.Module):
    def __init__(self, input_size, hidden_size, output_size):
        super().__init__()
        self.hidden_size = hidden_size
        self.rnn = nn.RNN(input_size, hidden_size, batch_first=True)
        self.fc = nn.Linear(hidden_size, output_size)
        self.hidden = None  # hidden state 저장용

    def forward(self, x):
        # 이전 hidden state가 있으면 이어받고, 없으면 0에서 시작
        if self.hidden is not None:
            self.hidden = self.hidden.detach()  # 역전파 끊기 (메모리 절약)
        out, self.hidden = self.rnn(x, self.hidden)
        out = self.fc(out[:, -1, :])
        return out

    def reset_hidden(self):
        self.hidden = None

model = StatefulRNN(input_size=1, hidden_size=32, output_size=1)

for epoch in range(num_epochs):
    model.reset_hidden()  # 에폭 시작 시 초기화
    for batch in dataloader:
        output = model(batch)  # 이전 배치의 hidden state 이어받음
```

핵심: `self.hidden`에 이전 배치의 hidden state를 저장해두고, 다음 배치에서 `self.rnn(x, self.hidden)`으로 넘긴다.

### 차이가 드러나는 부분

```python
# Stateless - 매번 0에서 시작
out, _ = self.rnn(x)

# Stateful - 이전 state 이어받기
out, self.hidden = self.rnn(x, self.hidden)
```

딱 이 한 줄 차이다. hidden state를 **버리느냐 저장해서 다음에 넘기느냐**.

---

## detach()를 하는 이유

Stateful RNN 코드에서 눈에 띄는 한 줄이 있다.

```python
self.hidden = self.hidden.detach()
```

이걸 안 하면 역전파가 **이전 배치의 모든 연산까지 거슬러 올라간다.** 배치가 쌓일수록 계산 그래프가 무한히 커져서 메모리가 터진다. `detach()`로 **값은 유지하되 계산 그래프는 끊는다.**

```
detach() 없음: 배치3 -> 배치2 -> 배치1까지 역전파 (메모리 폭발)
detach() 있음: 배치3만 역전파 (hidden 값은 배치2에서 이어받음)
```

---

## Stateful도 해결 못 하는 문제: hidden state 병목

Stateful이든 Stateless든, hidden state는 **고정 크기 벡터**다.

```
h = [0.3, -0.1, 0.7, ..., 0.2]  <- 예: 256차원 고정
```

1000스텝의 정보든 10스텝의 정보든 이 고정 크기 벡터 하나에 압축해야 한다. 스텝이 길어질수록 **초반 정보는 점점 덮어씌워진다.**

```
x1 -> h1 (x1 정보 100%)
x1,x2 -> h2 (x1 정보 희석)
x1~x100 -> h100 (x1 정보 거의 소실)
x1~x1000 -> h1000 (x1 정보 사실상 없음)
```

이것은 인코더-디코더의 병목 문제와 **본질이 같다.** 둘 다 고정 크기 벡터 하나에 가변 길이 정보를 우겨넣는 구조적 한계이며, 이 병목을 깬 것이 **Attention**(2015, Bahdanau et al.)이고, 더 나아가 RNN 자체를 없애고 Attention만으로 시퀀스를 처리하는 **Transformer**로 이어졌다.

---

## 언제 어떤 걸 쓰는가

| 상황 | 선택 | 이유 |
|------|------|------|
| 문장 분류, 감성 분석 | **Stateless** | 각 문장이 독립적 |
| 긴 시계열 예측 (주가, 센서) | **Stateful** | 연속된 시간 흐름을 이어야 함 |
| 텍스트 생성 (긴 문서) | **Stateful** | 앞 문맥을 계속 참조 |
| 일반적인 경우 | **Stateless** | 기본값이고 관리가 쉬움 |

**Stateless로 충분한 경우가 대부분**이고, 하나의 긴 연속 시퀀스를 배치 단위로 잘라 처리해야 할 때만 Stateful이 필요하다.

---

## 참고 자료

- [PyTorch RNN Documentation](https://pytorch.org/docs/stable/generated/torch.nn.RNN.html)
- [Neural Machine Translation by Jointly Learning to Align and Translate (Bahdanau et al., 2015)](https://arxiv.org/abs/1409.0473)
