---
title: "AI Harness 이해하기 — 두 가지 관점에서 본 LLM의 통제와 평가"
date: 2026-03-14 10:30:00 +0900
categories: [AI]
tags: [llm, evaluation, agent, harness, tool-use, rag]
---

## 들어가며: Harness란 무엇인가?

"Harness"라는 단어는 본래 **말의 고삐나 마구(馬具)**를 뜻한다. AI 맥락에서 이 용어가 새로운 의미로 쓰이고 있다. 바로 **"강력하지만 제어하기 어려운 LLM의 능력을 실제로 활용 가능하게 만드는 모든 것"**을 뜻한다.

하지만 "Harness"에는 두 가지 관점이 있다:
1. **통합/실용 관점** — LLM을 실제 시스템에 연결하고 제어하는 것
2. **측정/검증 관점** — LLM의 성능을 평가하고 개선하는 것

이 글에서는 이 두 관점을 명확히 구분하고, 각각의 의미와 사례를 정리한다.

---

## 관점 1: 통합/실용 관점의 Harness

### LLM의 한계, Harness의 역할

LLM은 강력하지만 그 자체만으로는 실제 업무를 처리할 수 없다. LLM의 한계와 그를 보완하는 Harness의 솔루션은 다음과 같다:

| LLM의 한계 | Harness 솔루션 | 설명 |
|-----------|---------------|------|
| **Context 길이 제한** | Memory 관리 | 대화 히스토리를 효율적으로 저장/로드 |
| **환각(Hallucination)** | RAG (정보 검색) | 실시간 데이터베이스/인터넷 검색으로 사실 기반 제공 |
| **외부 시스템 접근 불가** | Tools / MCP | 파일 시스템, 데이터베이스, API 접근 |
| **제어 어려움** | 구조화된 프롬프트 | 역할 정의, 명확한 지시, 출력 형식 지정 |

### 실제 예: Claude Code

**Claude Code는 Claude 모델을 위한 Harness**다. 다음 기능들이 LLM을 실제 에이전트로 변환한다:

```
Claude (LLM)
    ↓
┌─────────────────────────────────────┐
│  Claude Code (Harness)              │
├─────────────────────────────────────┤
│ 📁 파일 시스템 접근 (Read/Write)    │
│ 🖥️  터미널 명령 실행 (Bash)        │
│ 🔗 MCP(Model Context Protocol) 연결 │
│ 🔍 검색 도구 (Grep, Glob)           │
│ 🌐 웹 조회 (WebFetch, WebSearch)    │
│ 📊 Git 명령 (커밋, PR 생성 등)      │
└─────────────────────────────────────┘
    ↓
실제 프로젝트 수정, 코드 리뷰, 배포 가능
```

**Harness 없으면:**
- Claude는 당신의 코드를 읽을 수 없다 ❌
- 파일을 수정할 수 없다 ❌
- 터미널 명령을 실행할 수 없다 ❌
- 결과적으로 "대화만" 가능 (ChatGPT 수준)

**Harness 있으면:**
- 실제 파일을 직접 읽고 수정한다 ✅
- 테스트를 자동으로 실행한다 ✅
- 변경 사항을 Git에 커밋한다 ✅
- **실제 에이전트로 작동** ✅

### 다른 사례들

**OpenAI Codex Harness**
```
프롬프트 (코드 생성 요청)
    ↓
Codex 모델 실행
    ↓
생성된 코드를 JSON 형식으로 출력
    ↓
Tools, MCP 연결로 빌드/테스트 실행
    ↓
점진적 개선 및 반복
```

---

## 관점 2: 측정/검증 관점의 Harness

### Harness = 평가 인프라

Harness는 또한 **모델의 성능을 체계적으로 측정하는 자동화 시스템**을 의미한다.

```
테스트 케이스 1000개 준비
    ↓
모델에 입력하고 출력 수집
    ↓
┌─────────────────────────────────┐
│ 평가 Harness 실행               │
├─────────────────────────────────┤
│ 1️⃣  채점 (Grader)              │
│    → 답이 맞는가? 정확도?       │
│                                 │
│ 2️⃣  비교 (Comparator)          │
│    → v1.0 vs v1.1 어느 게 낫나? │
│                                 │
│ 3️⃣  분석 (Analyzer)            │
│    → 어떤 카테고리가 약한가?    │
└─────────────────────────────────┘
    ↓
보고서: 정확도 95%, 응답시간 1.2초
```

### 평가 Harness의 구성 요소

| 요소 | 설명 | 예시 |
|------|------|------|
| **Test Cases** | 평가할 입력 데이터 | 1000개의 고객 질문 |
| **Expected Output** | 기준 답변 | "긍정적 감정 분류" |
| **Metrics** | 측정 지표 | 정확도, F1 스코어, 응답시간 |
| **Grading Logic** | 자동 채점 규칙 | "정확히 일치"? "부분 점수"? |
| **Logging** | 결과 기록 | 각 테스트의 성공/실패, 점수 |

### 실제 예: EleutherAI LM Evaluation Harness

EleutherAI의 오픈소스 도구는 60개 이상의 벤치마크를 포함한다:

```yaml
# tasks.yaml - 평가 작업 정의
tasks:
  - name: "reading_comprehension"
    dataset: "squad.json"
    prompt_template: "질문: {question}\n단락: {context}\n답:"
    metric: ["exact_match", "f1_score"]

  - name: "sentiment_classification"
    dataset: "sst2.json"
    prompt_template: "{text}\n감정: [긍정/부정]"
    metric: ["accuracy"]
```

설치와 실행:
```bash
git clone https://github.com/EleutherAI/lm-evaluation-harness
cd lm-evaluation-harness
pip install -e .

# 기본 평가 실행
lm_eval --model claude --tasks mmlu,gsm8k --num_fewshot 5
```

### Anthropic Skill-Creator의 평가 파이프라인

Anthropic은 스킬 개발 과정에 평가를 통합했다:

```
┌─────────────────────────────────────────┐
│ 1. CREATE (스킬 작성)                   │
│    → 함수 또는 워크플로우 정의         │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ 2. EVAL (평가 실행)                     │
│    ├─ Executor: 스킬 실행               │
│    ├─ Grader: 출력 검증                 │
│    ├─ Comparator: 버전 비교             │
│    └─ Analyzer: 패턴 분석               │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ 3. IMPROVE (개선)                       │
│    → 약한 부분 식별 & 수정             │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ 4. BENCHMARK (최종 비교)                │
│    → 어느 버전이 최고인가?             │
└─────────────────────────────────────────┘
```

---

## 두 관점의 관계

흥미로운 점은, 이 두 관점이 **별개가 아니라 상호보완적**이라는 것이다:

```
Harness (통합)로 실제 작업 수행
    ↓
작업 결과 수집
    ↓
Harness (평가)로 성능 측정
    ↓
개선점 발견
    ↓
다시 작업 수행 (반복)
```

예를 들어:
1. **Claude Code (통합 Harness)**로 실제 코드 리뷰 수행
2. 리뷰 결과를 데이터 수집
3. **평가 Harness**로 "리뷰 품질이 얼마나 좋은가?" 측정
4. 모델 또는 프롬프트 개선
5. 다시 반복

---

## 당신이 Harness를 구축할 때 고르는 길

### Path 1: LLM을 실제로 쓸 수 있게 하기 (통합)

**목표:** Claude/GPT를 단순 챗봇에서 벗어나 실제 일을 하는 에이전트로

**필요한 것:**
- Tools 정의 (파일 접근, API 호출, 데이터베이스 조회)
- RAG 시스템 (정확한 정보 제공)
- Memory 관리 (긴 대화 유지)
- 구조화된 프롬프트 (지시사항 명확화)

**예시:**
```python
# Claude를 실제 작업 에이전트로 만들기
tools = [
    {
        "name": "read_file",
        "description": "파일 내용 읽기",
        "input_schema": {...}
    },
    {
        "name": "execute_sql",
        "description": "데이터베이스 쿼리 실행",
        "input_schema": {...}
    }
]

response = client.messages.create(
    model="claude-opus-4-6",
    tools=tools,
    messages=[...]
)
```

### Path 2: LLM의 성능을 평가하기 (측정)

**목표:** 모델의 정확도, 속도, 비용 추적하기

**필요한 것:**
- 평가 데이터셋 준비
- 자동 채점 로직
- 메트릭 정의
- 시간에 따른 추적 (v1.0 → v1.1 → v2.0)

**예시:**
```python
# 간단한 평가 Harness
eval_cases = [
    {"question": "한국 수도는?", "expected": "서울"},
    {"question": "2+2는?", "expected": "4"},
]

for case in eval_cases:
    response = claude.message(case["question"])
    is_correct = response == case["expected"]
    results.append({"case": case, "correct": is_correct})

accuracy = sum(r["correct"] for r in results) / len(results)
print(f"정확도: {accuracy * 100}%")
```

---

## 핵심 정리

| 관점 | Harness의 역할 | 초점 | 기업 사례 |
|------|---------------|------|---------|
| **통합/실용** | LLM을 실제 시스템에 연결하고 제어 | "어떻게 쓸 것인가?" | Claude Code, Codex |
| **측정/검증** | 모델 성능을 체계적으로 평가 | "얼마나 잘하는가?" | Skill-Creator, Eval Harness |

---

## 다음 단계: 직접 해보기

이 글에서 다룬 두 관점을 실제로 경험하는 가장 빠른 경로는 **skill을 직접 만들고 평가하는 것**이다.

### Step 1: 통합 Harness — skill 구축

목표: Claude가 실제 작업을 수행하도록 Tool을 연결하는 skill 하나를 만든다.

개발 사이클:
1. **PRD 작성** — 이 skill이 어떤 문제를 해결하는가?
2. **요구사항 정의** — skill이 다뤄야 할 입력/출력/엣지케이스
3. **DB Entity 설계** (상태를 저장해야 할 경우)
4. **구현** — 테스트케이스 구성 → 테스트코드 작성 → 구현 → 리팩토링

### Step 2: 평가 Harness — skill-creator eval

skill을 만들었으면, Claude Code에서 skill-creator로 평가한다:

```
/skill-creator
→ Eval 모드 선택
→ 테스트 프롬프트 + 기대 결과(assertions) 정의
→ 병렬 에이전트가 자동 채점
→ HTML 리포트 확인 (통과율, 토큰, 시간)
```

### Step 3: 반복 개선

Improve 모드로 실패 패턴 분석 → skill 수정 → Benchmark로 버전 비교.
이 사이클을 반복하면서 실제로 작동하는 에이전트를 만들어간다.

> skill-creator 공식 가이드: [Improving skill-creator](https://claude.ai/blog/improving-skill-creator)

---

## 참고 자료

- [Anthropic Blog: Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- [OpenAI: Harness Engineering](https://openai.com/index/harness-engineering/)
- [EleutherAI LM Evaluation Harness GitHub](https://github.com/EleutherAI/lm-evaluation-harness)
- [Toss Tech Blog: Software 3.0 Era](https://toss.tech/article/software-3-0-era)
