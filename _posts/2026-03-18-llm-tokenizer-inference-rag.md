---
title: "LLM의 토크나이저, 추론, 그리고 RAG"
date: 2026-03-18 21:00:00 +0900
categories: [AI]
tags: [llm, tokenizer, bpe, rag, nlp]
---

## 요약

| 주제 | 핵심 내용 |
|------|----------|
| BPE 토크나이제이션 | 빈번한 문자 쌍을 반복 병합하여 서브워드 단위를 만드는 알고리즘 |
| 토크나이저 파일 | `vocab.json`(토큰→ID 매핑) + `merges.txt`(병합 규칙)로 결정론적 동작 |
| LLM 추론 | 다음 토큰의 확률 분포를 계산하여 순차 생성 (검색이 아닌 생성) |
| RAG | 외부 검색 결과를 프롬프트에 추가하여 LLM 입력을 풍부하게 만드는 파이프라인 |

BPE 서브워드 토크나이제이션부터 LLM의 답변 생성 원리, RAG 아키텍처까지 정리한 노트다.

---

## 1. BPE (Byte Pair Encoding) 서브워드 토크나이제이션

### 1.1 배경: 희귀 단어 문제

NMT(Neural Machine Translation) 모델은 고정된 어휘(vocabulary)를 사용한다. 이때 두 가지 근본적인 문제가 발생한다.

- **OOV(Out-of-Vocabulary) 문제**: 어휘에 없는 단어는 `<UNK>` 토큰으로 처리되어 번역 품질이 급격히 떨어진다.
- **어휘 크기 vs 연산 비용 트레이드오프**: 어휘 크기를 무작정 늘리면 softmax 연산 비용이 폭증한다.

특히 독일어, 터키어 같은 교착어·합성어가 많은 언어에서 이 문제가 심각하다. 예를 들어 "Abwasserbehandlungsanlage"(하수처리시설) 같은 합성어는 통째로 어휘에 넣기 어렵지만, "Abwasser" + "behandlungs" + "anlage"로 분리하면 각각은 흔한 단어가 된다.

### 1.2 핵심 아이디어: BPE를 서브워드 분절에 적용

BPE는 원래 데이터 압축 알고리즘이다. Sennrich et al. (2016)은 이를 단어 분절(segmentation)에 응용했다.

**알고리즘 동작 방식:**

1. 학습 코퍼스의 모든 단어를 개별 문자(character) 시퀀스로 초기화한다.
2. 가장 빈번하게 인접 등장하는 문자 쌍(bigram)을 찾는다.
3. 그 쌍을 하나의 새로운 단위로 병합(merge)한다.
4. 2~3을 원하는 병합 횟수(hyperparameter)만큼 반복한다.

예를 들어 "low", "lower", "newest", "widest"가 코퍼스에 있다면, 빈도가 높은 문자 쌍부터 순차적으로 병합하면서 `l o w` → `lo w` → `low`, `e s t` → `es t` → `est` 같은 서브워드 단위가 자연스럽게 형성된다.

**병합 횟수가 핵심 하이퍼파라미터다.** 적게 하면 문자 수준에 가까워지고(어휘 작음, 시퀀스 길어짐), 많이 하면 전체 단어 수준에 가까워진다(어휘 큼, 시퀀스 짧아짐).

### 1.3 Joint BPE vs. Separate BPE

- **Separate BPE**: 각 언어에서 독립적으로 BPE를 학습한다.
- **Joint BPE**: source와 target 코퍼스를 합쳐서 하나의 BPE를 학습한다.

Joint BPE가 더 효과적인데, 같은 단어(예: 고유명사)가 양쪽 언어에서 동일한 서브워드로 분절되어 모델이 복사(copy) 전략을 학습하기 쉬워지기 때문이다.

### 1.4 실험 결과와 영향

WMT 번역 태스크(영→독, 영→러)에서 BPE 기반 서브워드 분절이 기존 방식보다 BLEU 점수가 유의미하게 높았다. 이후 서브워드 토크나이제이션은 NLP의 사실상 표준이 되었다. GPT 시리즈는 BPE를, BERT는 WordPiece를, T5·LLaMA 등은 SentencePiece(Unigram LM)를 사용하는 등 구체적인 알고리즘은 다르지만, "빈번한 부분 문자열을 병합하여 서브워드를 만든다"는 핵심 아이디어는 공통이다.

---

## 2. 사전학습 모델의 재사용과 Vocabulary

### 2.1 모델과 토크나이저는 반드시 세트로 사용해야 한다

사전학습된 언어 모델을 재사용할 때 모델 가중치와 토크나이저(vocabulary + merge rules)는 반드시 함께 가져와야 한다.

모델의 embedding layer는 각 토큰 ID에 대응하는 벡터를 학습한다. 예를 들어 토큰 ID 1523이 "play"에 대응한다면, embedding matrix의 1523번 행은 "play"의 의미를 인코딩한 벡터다. 다른 토크나이저를 가져다 쓰면 ID 1523이 전혀 다른 토큰을 가리키게 되어 모델이 학습한 모든 표현이 깨진다. output layer의 softmax도 동일한 vocabulary 크기와 매핑에 의존한다.

### 2.2 실제 배포 구조

Hugging Face 같은 허브에서 모델을 다운로드하면 항상 두 가지가 함께 온다.

| 구성 요소 | 파일 예시 | 역할 |
|-----------|----------|------|
| 모델 가중치 | `pytorch_model.bin`, `model.safetensors` | 신경망 파라미터 |
| 토크나이저 (BPE 기반) | `vocab.json` + `merges.txt` | 토큰↔ID 매핑 + BPE 병합 규칙 |
| 토크나이저 (SentencePiece 기반) | `tokenizer.model` | 통합 토크나이저 모델 |

코드에서도 항상 같은 체크포인트에서 둘을 로드한다:

```python
from transformers import AutoTokenizer, AutoModel

tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")
model = AutoModel.from_pretrained("bert-base-uncased")
```

### 2.3 Fine-tuning 시 Vocabulary 확장

Fine-tuning할 때도 기본적으로 원래 토크나이저를 그대로 쓴다. 새 도메인에 특화된 단어가 필요하면 `add_tokens()`로 토큰을 추가하고 embedding layer를 resize할 수 있다:

```python
tokenizer.add_tokens(["[VIN]", "[PART_NUMBER]"])
model.resize_token_embeddings(len(tokenizer))
```

새 토큰에 대한 embedding 벡터는 랜덤 초기화되고, fine-tuning 과정에서 학습된다. 기존 vocabulary와 그에 대응하는 embedding은 그대로 유지된다.

---

## 3. 토크나이저 파일의 실제 구조

### 3.1 토크나이저는 학습이 필요 없다 (추론 시)

토크나이저 학습은 모델 학습 이전에 별도의 과정으로 한 번만 수행된다.

1. 대규모 코퍼스에서 문자 쌍의 빈도를 세고 merge를 반복 → `merges.txt` 생성
2. 최종적으로 만들어진 모든 서브워드에 ID를 부여 → `vocab.json` 생성
3. 이 토크나이저로 코퍼스를 토큰화한 뒤, 그 결과로 모델을 학습

이후 토크나이저는 고정(frozen)된다. 추론 시에는 `vocab.json` + `merges.txt`만으로 순수한 룰 기반 변환기로 동작한다. 신경망이 아니라 단순한 문자열 처리 알고리즘이다.

### 3.2 vocab.json — 토큰-ID 매핑 테이블

```json
{
  "!": 0,
  "a": 64,
  "b": 65,
  "Ġthe": 262,
  "Ġand": 290,
  "er": 263,
  "ing": 278,
  "tion": 653,
  "Ġplay": 711,
  "Ġplayer": 2137,
  "Ġunbelievable": 22082,
  "<|endoftext|>": 50256
}
```

GPT-2 기준 50,257개의 토큰이 존재한다. `Ġ`는 단어 앞의 공백을 나타내는 특수 문자다. 개별 문자(바이트 레벨 256개)가 먼저 오고, BPE merge로 만들어진 서브워드들이 뒤에 위치한다.

### 3.3 merges.txt — BPE 병합 규칙

```
#version: 0.2
Ġ t
Ġ a
h e
i n
r e
o n
Ġt he
e r
Ġ s
a t
in g
```

위에서부터 순서대로 적용하는 병합 규칙이다. 이 파일의 순서가 곧 BPE의 병합 우선순위다.

### 3.4 토큰화 동작 예시

```
입력: "unbelievable"

1. merges.txt의 규칙을 위에서부터 순서대로 적용
2. "u n b e l i e v a b l e"
   → "un" + "bel" + "iev" + "able"
   → "unbeliev" + "able"
3. vocab.json에서 각 서브워드의 ID를 조회
```

GPU도 필요 없고, 학습도 필요 없고, 두 파일의 룰만으로 결정론적(deterministic)으로 동작한다. 같은 입력이 들어오면 항상 같은 토큰 시퀀스가 나온다.

### 3.5 토큰 ID에서 모델 입력으로

토크나이저의 출력은 정수 배열이다:

```
"BPE가 뭐야?" → [BPE, 가, Ġ뭐, 야, ?] → [18078, 3456, 12847, 9021, 30]
```

이 정수들이 모델의 embedding layer를 통과하면 각각 고차원 벡터(예: 768차원)로 변환된다. 이 벡터 시퀀스가 Transformer의 실제 입력이 된다.

여기까지가 **결정론적 영역**이다. 같은 텍스트는 항상 같은 토큰 ID, 같은 embedding 벡터를 만든다. 사실 Transformer의 forward pass — attention과 FFN 연산 — 도 같은 입력이면 같은 확률 분포를 출력하므로 결정론적이다. 비결정론적이 되는 지점은 그 확률 분포에서 다음 토큰을 **샘플링**하는 단계다. 다음 섹션에서 이 과정을 자세히 다룬다.

---

## 4. LLM은 어떻게 질문에 답변하는가

### 4.1 "찾는다"는 표현은 부정확하다

LLM은 데이터베이스에서 무언가를 검색(retrieve)하는 것이 아니다. 학습 과정에서 본 텍스트의 패턴이 가중치(weight)에 녹아들어 있고, 추론 시에는 그 가중치를 통해 다음 토큰의 확률 분포를 계산할 뿐이다.

### 4.2 학습 때 일어나는 일

학습 코퍼스에 BPE에 관한 문장들이 있었다면, 모델은 다음 토큰 예측(next token prediction)을 반복한다:

```
입력: "BPE stands for"       → 정답: "Byte"
입력: "BPE stands for Byte"  → 정답: "Pair"
입력: "BPE stands for Byte Pair" → 정답: "Encoding"
```

이 과정에서 수십억 개의 파라미터가 조금씩 업데이트된다. "BPE"라는 토큰 뒤에 "Byte", "Pair", "Encoding", "algorithm", "subword", "merge" 같은 토큰이 올 확률이 높다는 통계적 관계가 가중치에 분산 저장된다.

### 4.3 추론 때 일어나는 일

사용자가 "BPE가 뭐야?"라고 질문하면:

```
입력 토큰: [BPE, 가, Ġ뭐, 야, ?]
```

이 토큰 시퀀스가 Transformer의 레이어를 통과하면서 attention 메커니즘이 "BPE"와 "뭐야"의 관계를 포착하고, 최종 레이어의 출력이 softmax를 거쳐 다음 토큰의 확률 분포를 만든다:

```
P("Byte") = 0.15
P("서브워드") = 0.12
P("BPE") = 0.08
P("It") = 0.03
... 전체 vocabulary의 각 토큰에 대한 확률
```

여기서 하나를 샘플링하고, 그 토큰을 다시 입력에 붙여서 또 다음 토큰을 예측하고, 이를 EOS 토큰이 나올 때까지 반복한다.

### 4.4 Hallucination의 구조적 원인

어딘가에 "BPE란 Byte Pair Encoding의 약자로..."라는 문서가 저장되어 있고 그걸 꺼내오는 것이 아니다. 학습 때 본 수많은 문장의 패턴이 수십억 개 파라미터에 분산되어 압축 저장된 것이고, 추론 시에는 그 파라미터들의 행렬 연산 결과로 그럴듯한 토큰 시퀀스가 생성되는 것이다. "찾아서 읽어주는" 게 아니라 "그럴듯한 다음 토큰을 생성하는" 것이기 때문에, 패턴 조합이 현실과 다른 결과를 낼 수 있다.

---

## 5. RAG (Retrieval-Augmented Generation)

### 5.1 순수 LLM의 한계

```
사용자 질문 → [LLM] → 파라미터에서 생성 → 답변
```

모델 가중치에 압축 저장된 패턴만으로 답변을 생성한다. 학습 때 못 본 내용이나 최신 정보는 답할 수 없고, 본 내용도 부정확하게 재구성할 수 있다.

### 5.2 RAG를 추가하면

```
사용자 질문 → [Retriever가 관련 문서 검색] → 질문 + 검색된 문서를 합쳐서 → [LLM] → 답변
```

핵심은 LLM에 입력되는 프롬프트 자체가 달라진다는 것이다. LLM의 내부 구조를 바꾸는 게 아니라, 입력을 풍부하게 만들어주는 외부 파이프라인을 추가하는 것이다.

### 5.3 RAG 동작 과정

**Step 1 — 질문을 벡터로 변환**

"BPE가 뭐야?"를 embedding 모델(답변을 생성하는 LLM과는 별개의 모델)에 넣어서 벡터(예: 768차원 float 배열)로 만든다.

**Step 2 — 벡터 DB에서 유사한 문서 검색**

미리 chunking해둔 문서들의 벡터와 코사인 유사도를 비교해서 가장 관련 높은 chunk를 가져온다.

**Step 3 — 프롬프트 조립**

검색된 문서를 질문과 합쳐서 LLM에 넘긴다:

```
아래 참고 자료를 바탕으로 질문에 답하세요.

[참고 자료]
"BPE(Byte Pair Encoding)는 원래 데이터 압축 알고리즘으로,
Sennrich et al.(2016)이 이를 서브워드 분절에 적용했다.
가장 빈번한 문자 쌍을 반복적으로 병합하여..."

[질문]
BPE가 뭐야?
```

**Step 4 — LLM이 답변 생성**

LLM 입장에서는 더 긴 입력 시퀀스가 들어온 것뿐이다. 동작 원리는 동일하게 다음 토큰 예측이다. 하지만 context window 안에 정확한 정보가 들어있으니, attention 메커니즘이 참고 자료의 관련 부분에 높은 가중치를 주고, 결과적으로 훨씬 정확한 토큰 시퀀스를 생성하게 된다.

### 5.4 RAG의 본질

모델 파라미터는 그대로이고, 입력을 풍부하게 만들어주는 외부 파이프라인을 추가하는 것이다. LLM은 여전히 다음 토큰을 예측할 뿐인데, 예측의 근거가 되는 context가 더 정확하고 풍부해진다.

### 5.5 RAG에도 한계는 있다

RAG를 붙인다고 hallucination이 완전히 사라지는 것은 아니다.

- **검색 실패**: Retriever가 관련 없는 문서를 가져오면, LLM은 잘못된 context를 근거로 답변을 생성한다.
- **검색 무시**: LLM이 검색된 문서를 무시하고 파라미터에 저장된 패턴(parametric memory)에 의존할 수 있다.
- **잘못된 조합**: 검색된 여러 chunk의 내용을 잘못 해석하거나 조합할 수 있다.

RAG는 hallucination의 가능성을 줄여주는 강력한 전략이지만, 근본적인 해결책은 아니다.

---

## 참고 자료

- Sennrich, R., Haddow, B., & Birch, A. (2016). [Neural Machine Translation of Rare Words with Subword Units](https://aclanthology.org/P16-1162/). ACL 2016.
- Lewis, P., et al. (2020). [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401). NeurIPS 2020.
