---
title: "LeRobot lekiwi 예제에 --resume 옵션 추가하기 (feat. Draccus 개념)"
date: 2026-03-29 00:00:00 +0900
categories: [AI]
tags: [lerobot, lekiwi, robotics, draccus, dataset]
---

> 이 글은 AI Agent가 작성했습니다.

## 왜 이 글을 쓰게 됐나

LeRobot의 `examples/lekiwi/record.py`로 데이터를 녹화하다가 중간에 끊기는 상황이 생겼다. 다시 이어서 녹화하고 싶은데, 이 예제 파일에는 `--resume` 같은 옵션이 **전혀 없다**.

공식 CLI인 `lerobot-record`에는 `--resume` 플래그가 있지만, 이걸 lekiwi에 쓰려면 또 다른 문제가 있다 (이것도 아래서 다룬다).


---

## Draccus란 무엇인가

먼저 배경 개념부터 짚고 가자.

LeRobot의 공식 스크립트들(`lerobot_record.py`, `lerobot_train.py` 등)은 **Draccus**라는 라이브러리로 CLI 인자를 파싱한다.

Draccus는 Python **dataclass를 그대로 CLI 인터페이스로** 만들어주는 라이브러리다. 핵심 아이디어는 단순하다: dataclass 필드 하나 = CLI 인자 하나.

### 기본 동작 방식

`@parser.wrap()` 데코레이터를 함수에 붙이면, 함수의 첫 번째 인자 타입(dataclass)을 보고 CLI 인자를 자동으로 파싱해서 주입해준다.

```python
# lerobot/scripts/lerobot_record.py (실제 코드)

@dataclass
class DatasetRecordConfig:
    repo_id: str          # --dataset.repo_id=...
    fps: int = 30         # --dataset.fps=30
    num_episodes: int = 50

@dataclass
class RecordConfig:
    robot: RobotConfig        # --robot.xxx=...
    dataset: DatasetRecordConfig  # --dataset.xxx=...
    resume: bool = False      # --resume=true

@parser.wrap()
def record(cfg: RecordConfig):
    # 이 함수가 호출될 때 cfg는 이미 CLI 인자로 채워져 있다
    if cfg.resume:
        dataset = LeRobotDataset.resume(cfg.dataset.repo_id, ...)
```

실제 CLI 사용 예:

```bash
lerobot-record \
    --robot.type=so100_follower \
    --robot.port=/dev/ttyACM0 \
    --dataset.repo_id=myname/my_dataset \
    --dataset.fps=30 \
    --resume=true
```

**중첩 dataclass**는 `.`으로 구분한다. `RecordConfig.dataset`이 `DatasetRecordConfig` 타입이므로 `--dataset.fps` 처럼 접근한다.

argparse와 비교하면 이렇다:

| | argparse | Draccus |
|---|---|---|
| 인자 정의 위치 | `add_argument()` 호출부 | dataclass 필드 선언부 |
| 기본값 관리 | `default=` 파라미터 | 필드 기본값 |
| 중첩 구조 | 직접 구현 필요 | `.`으로 자동 지원 |
| 타입 힌트 활용 | 수동 | 자동 |

### Draccus의 핵심: ChoiceRegistry

Draccus의 또 다른 핵심 기능은 **ChoiceRegistry**다. 로봇 종류가 여러 개일 때 `--robot.type=xxx` 하나로 올바른 config 클래스를 선택할 수 있게 해준다.

**문제 상황**: 로봇마다 설정이 다르다. SO100은 `port`가 필요하고, LeKiwi는 `remote_ip`가 필요하다. 이걸 하나의 CLI로 처리하려면?

**해결**: 부모 클래스를 `ChoiceRegistry`로 만들고, 각 로봇 config를 이름으로 등록한다.

```python
# 부모: ChoiceRegistry 상속
@dataclass(kw_only=True)
class RobotConfig(draccus.ChoiceRegistry, abc.ABC):
    id: str | None = None

# 자식: 이름으로 등록
@RobotConfig.register_subclass("so100_follower")
@dataclass
class SO100FollowerConfig(RobotConfig):
    port: str = "/dev/ttyACM0"

@RobotConfig.register_subclass("lekiwi_client")
@dataclass
class LeKiwiClientConfig(RobotConfig):
    remote_ip: str  # lekiwi 전용 필드
    port_zmq_cmd: int = 5555
```

CLI에서 `--robot.type=so100_follower`를 넘기면 Draccus가 자동으로 `SO100FollowerConfig`를 생성하고, `--robot.port=...`도 그 클래스의 필드에 매핑한다.

**핵심 조건**: 이 등록은 해당 모듈이 **import될 때** `@register_subclass` 데코레이터가 실행되면서 이루어진다. **import가 안 되면 등록도 안 된다.**

---

## lerobot-record CLI에서 lekiwi가 안 되는 이유

`lerobot-record`가 `--robot.type=lekiwi_client`를 파싱하려면, `LeKiwiClientConfig` 클래스가 **import되어 ChoiceRegistry에 등록**되어 있어야 한다.

그런데 `lerobot_record.py`의 import 목록을 보면:

```python
from lerobot.robots import (  # noqa: F401
    bi_so_follower,
    koch_follower,
    so_follower,
    ...
    # lekiwi 없음!
)
```

lekiwi 모듈이 import되지 않아서 `LeKiwiClientConfig`가 등록되지 않고, Draccus는 `lekiwi_client` 타입을 모른다.

게다가 구조적인 문제도 있다. lekiwi는 **arm 리더 + 키보드** 두 개의 teleop을 동시에 써야 하는데, `RecordConfig.teleop`은 단일 `TeleoperatorConfig | None`만 받는다. 리스트 형태를 지원하지 않는다.

이래서 lekiwi는 `examples/record.py` 같은 별도 스크립트를 쓰는 것이다.

참고: [PR#1865가 키워드 추가하는 내용인데 글 작성날짜 기준으로는 아직 merge 안된상태.](https://github.com/huggingface/lerobot/pull/1865)

---

## 예제 파일에 --resume 추가하기

`examples/lekiwi/record.py`에 `--resume`을 붙이는 방법은 간단하다. Draccus 없이 표준 `argparse`로 충분하다.

### Step 1: import 추가

```python
import argparse
from lerobot.utils.constants import HF_LEROBOT_HOME
from lerobot.utils.control_utils import sanity_check_dataset_robot_compatibility
```

### Step 2: main() 첫 줄에 argparse 추가

```python
def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--resume", action="store_true", help="Resume recording on an existing dataset")
    args = parser.parse_args()
    ...
```

`action="store_true"`: 플래그를 붙이면 `True`, 안 붙이면 `False`. 값을 따로 넘기지 않아도 된다.

### Step 3: dataset 생성 부분을 if/else로 분기

```python
root = HF_LEROBOT_HOME / HF_REPO_ID

if args.resume:
    dataset = LeRobotDataset.resume(
        repo_id=HF_REPO_ID,
        root=root,
        image_writer_threads=4,
    )
    sanity_check_dataset_robot_compatibility(dataset, robot, FPS, dataset_features)
else:
    dataset = LeRobotDataset.create(
        repo_id=HF_REPO_ID,
        fps=FPS,
        features=dataset_features,
        robot_type=robot.name,
        use_videos=True,
        image_writer_threads=4,
    )
```

### 사용법

```bash
# 새로 녹화 시작
python examples/lekiwi/record.py

# 이어서 녹화
python examples/lekiwi/record.py --resume
```

---

## resume()이 메타데이터를 읽는 방식

`LeRobotDataset.resume()` 내부에서는 이 한 줄이 핵심이다:

```python
obj.meta = LeRobotDatasetMetadata(obj.repo_id, obj._requested_root, ...)
```

이 안에서 로컬 파일 4개를 읽어온다:

| 파일 | 내용 |
|------|------|
| `meta/info.json` | fps, features, robot_type, total_episodes, total_frames |
| `meta/stats.json` | 각 feature의 통계값 (mean, std 등) |
| `meta/tasks.parquet` | task 목록 |
| `meta/episodes/*.parquet` | 에피소드별 메타데이터 |

`meta/info.json` 예시:

```json
{
  "fps": 30,
  "robot_type": "lekiwi_client",
  "total_episodes": 5,
  "total_frames": 4500,
  "features": {
    "arm_shoulder_pan.pos": {
      "dtype": "float32",
      "shape": [1]
    },
    "observation.front": {
      "dtype": "video",
      "shape": [480, 640, 3]
    }
  }
}
```

`resume()`이 `fps`나 `features`를 인자로 받지 않아도 되는 이유가 바로 이것이다. 이미 저장된 `info.json`에서 전부 꺼내온다.

### root가 필수인 이유

`create(root=None)`은 내부에서 `HF_LEROBOT_HOME / repo_id`를 자동으로 쓴다. 그런데 `resume(root=None)`은 `ValueError`를 던진다.

이유는 안전 때문이다. `resume()`은 `DatasetWriter`를 생성해서 파일을 **직접 쓴다**. Hub snapshot 캐시에 쓰면 공유 캐시가 오염되므로, 반드시 명시적인 로컬 경로를 요구한다.

그래서 우리는 `create(root=None)`의 기본값과 동일한 경로를 직접 계산해서 넘겨준다:

```python
root = HF_LEROBOT_HOME / HF_REPO_ID
```

### sanity_check_dataset_robot_compatibility

resume 직후 반드시 이걸 호출해야 한다. `info.json`에서 읽어온 값과 현재 코드 설정을 비교한다:

```
info.json의 fps       ↔  현재 FPS 상수
info.json의 robot_type ↔  robot.robot_type
info.json의 features   ↔  dataset_features 변수
```

하나라도 다르면 `ValueError`. 다른 로봇이나 변경된 피처로 이어쓰는 걸 막아준다.

---

## 정리

| 항목 | 내용 |
|------|------|
| 공식 CLI(`lerobot-record`) | Draccus 기반, lekiwi 미지원 (import 누락 + 구조적 한계) |
| 예제 파일 | 하드코딩 상수 + 별도 argparse로 `--resume` 추가 가능 |
| `resume()` 필수 조건 | `root` 명시 필요, `fps`/`features` 불필요 (info.json에서 로드) |
| 호환성 검증 | `sanity_check_dataset_robot_compatibility` 반드시 호출 |
