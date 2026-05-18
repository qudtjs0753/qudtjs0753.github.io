---
title: "HuggingFace snapshot_download의 cache_dir 구조 이해하기"
date: 2026-05-18 21:00:00 +0900
categories: [AI]
tags: [huggingface, snapshot_download, cache, symlink, lerobot]
---

HuggingFace Hub에서 모델이나 데이터셋을 다운로드할 때 `snapshot_download()`를 사용한다.
이 함수에는 `cache_dir`와 `local_dir` 두 가지 저장 방식이 있는데,
LeRobot을 비롯한 대부분의 ML 프레임워크가 `cache_dir` 방식을 권장한다.
왜 이런 구조가 필요한지, 어떻게 동작하는지 정리한다.

---

## 왜 이 구조가 필요한가

ML 모델 파일은 수십 GB에 달한다. 단순히 버전별로 파일을 복사해 저장하면
동일한 가중치 파일이 버전마다 중복 저장되어 디스크가 낭비된다.
이 문제를 해결하기 위해 HuggingFace는 **Git의 object store** 설계를 차용했다.

---

## Git object store와의 대응

Git도 내부적으로 모든 파일을 SHA1 해시명으로 `.git/objects/`에 저장하고,
워킹 트리는 그 위에 올라오는 "뷰"다.
HuggingFace cache_dir은 이 구조를 그대로 가져왔다.

| Git | HuggingFace cache_dir | 역할 |
|---|---|---|
| `.git/objects/` | `blobs/` | 실제 파일 데이터 저장 |
| `.git/refs/heads/`, `.git/refs/tags/` | `refs/` | 이름 → commit 해시 매핑 |
| working tree (checkout 결과) | `snapshots/<hash>/` | 특정 시점의 파일 목록 |

Git에서 브랜치나 태그는 commit 해시를 담은 텍스트 파일일 뿐이고,
실제 데이터는 objects에 있다. HuggingFace도 동일하다.

---

## 전체 디렉토리 구조

```
~/.cache/huggingface/lerobot/hub/
└── datasets--lerobot--pusht/        ← repo_id (/ → -- 치환, repo_type 접두사)
    ├── blobs/
    │   ├── a3f2c1...                ← v1 config.json 내용
    │   ├── 9e8b7a...                ← model.safetensors 내용 (버전 공유)
    │   ├── f4d3e2...                ← v2 config.json 내용
    │   └── ...
    ├── refs/
    │   ├── main                     ← 텍스트 파일, 내용: "a1b2c3d..."
    │   └── v2.0                     ← 텍스트 파일, 내용: "f9e8d7c..."
    └── snapshots/
        ├── a1b2c3d.../              ← v1 (= main) commit
        │   ├── config.json          → ../../../blobs/a3f2c1... (symlink)
        │   └── model.safetensors    → ../../../blobs/9e8b7a... (symlink)
        └── f9e8d7c.../              ← v2.0 commit
            ├── config.json          → ../../../blobs/f4d3e2... (symlink, 새 내용)
            └── model.safetensors    → ../../../blobs/9e8b7a... (symlink, 동일 blob)
```

---

## 세 가지 구성 요소

### refs/ — 이름 → commit 해시 매핑

`main`, `v1.0`, `v2.0` 같은 이름이 어떤 commit을 가리키는지 기록한다.
중요한 점은 **파일별 해시가 아니라 commit 해시 하나**만 저장한다는 것이다.

```
refs/v2.0  →  "f9e8d7c..."   ← 이 commit 시점 전체를 가리킴
```

어떤 파일이 그 commit에 포함되는지, 각 파일의 내용이 무엇인지는
refs가 아니라 snapshots가 알고 있다.

### snapshots/ — 특정 시점의 파일 목록 (symlink 껍데기)

각 디렉토리명이 commit 해시다. 안에는 **원래 파일명으로 된 symlink만** 있고 실제 데이터는 없다.

```
snapshots/f9e8d7c.../config.json  →  ../../../blobs/f4d3e2...
```

여기서 핵심은 symlink 방향이다. **snapshot → blob** 방향이지, blob → snapshot이 아니다.
파일의 이름과 경로 정보는 snapshot이 들고 있고, 실제 바이트는 blob이 들고 있다.

OS가 symlink를 투명하게 따라가므로 코드 입장에서는 일반 파일과 동일하게 접근된다.

```bash
$ ls -la snapshots/f9e8d7c.../
config.json -> ../../../blobs/f4d3e2c...       # symlink
model.safetensors -> ../../../blobs/9e8b7a...  # symlink
```

### blobs/ — 실제 데이터 창고

파일 내용의 SHA256 해시가 파일명이다. 파일 형식을 알지 못하는 **순수한 바이트 덩어리**다.
파일명(`.json`, `.safetensors`)도 없고 어느 모델에 속하는지도 모른다.
그 정보는 snapshot의 symlink 이름이 가지고 있다.

파일 내용이 같으면 해시가 같으므로 여러 버전에서 동일한 파일은 blob 하나를 공유한다.

```
v1의 model.safetensors 내용 SHA256 = 9e8b7a...
v2의 model.safetensors 내용 SHA256 = 9e8b7a...  ← 동일
→ blobs/9e8b7a... 하나만 존재, 두 snapshot이 같은 blob을 가리킴
```

---

## 모델 로드 시 상호작용 흐름

`snapshot_download(repo_id="lerobot/pusht", revision="v2.0")`을 호출하면
내부적으로 다음 순서로 동작한다.

```
1. refs/v2.0 읽기
        ↓
   "f9e8d7c..."  (commit 해시 획득)
        ↓
2. snapshots/f9e8d7c.../  경로 반환
        ↓
3. Python 코드가 이 경로를 root로 사용
        ↓
4. root / "config.json" 열기
        ↓  OS가 symlink 추적
   blobs/f4d3e2...  실제 바이트 읽기
```

LeRobot에서는 이 반환 경로를 바로 `root`로 사용한다.

```python
# lerobot/datasets/lerobot_dataset.py
self.meta.root = Path(
    snapshot_download(
        self.repo_id,
        repo_type="dataset",
        revision=self.revision,
        cache_dir=HF_LEROBOT_HUB_CACHE,  # ~/.cache/huggingface/lerobot/hub/
    )
)
# self.meta.root = ~/.cache/.../snapshots/f9e8d7c.../
```

이후 코드는 `root / "meta/info.json"`, `root / "data/episode_000.parquet"` 처럼
평범한 파일 경로로 데이터에 접근한다. symlink인지 신경 쓸 필요가 없다.

---

## local_dir 방식과 비교

| | `cache_dir` | `local_dir` |
|---|---|---|
| 파일 저장 | `blobs/`에 해시명으로 1개 | 지정 경로에 실제 파일 복사 |
| 중복 제거 | symlink로 공유 | 버전마다 별도 복사 |
| Revision 격리 | `snapshots/<hash>/`로 자동 격리 | 없음 (덮어씀) |
| 반환값 | `snapshots/<hash>/` 경로 | `None` |
| legacy 감지 | 해당 없음 | `.cache/huggingface/download/` 생성 |

`local_dir` 방식을 사용하면 `.cache/huggingface/download/` 디렉토리가 생기는데,
LeRobot의 `has_legacy_hub_download_metadata()`가 이를 감지해 재다운로드를 유도한다.

---

## 주의할 점

**blob은 삭제되지 않는다.**
어떤 snapshot에서라도 참조 중인 blob은 남아 있다.
버전이 쌓일수록 캐시가 커지는 이유다. 명시적으로 revision을 삭제해야 정리된다.

```python
from huggingface_hub import scan_cache_dir
scan_cache_dir().delete_revisions("a1b2c3d...")
# 이 snapshot만 참조하던 blob도 함께 정리 가능
```

**10GB 파일을 수정하면 10GB blob이 추가된다.**
blob은 파일 단위 저장이라 1바이트만 바뀌어도 파일 전체가 새 blob이 된다.
HuggingFace가 대용량 모델을 여러 shard로 나누는 이유가 여기 있다.

```
model-00001-of-00004.safetensors  ← 레이어 1~n
model-00002-of-00004.safetensors  ← 레이어 n+1~m
...
```

특정 레이어만 파인튜닝했다면 해당 shard만 새 blob이 생기고 나머지는 공유된다.
