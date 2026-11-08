---
title: "[팜가드] 공유 상태 클래스(GatewayState) 설계하고 직접 구현하기"
date: 2026-11-08
tags: [팜가드, 소프트웨어개발론, 멀티스레딩, Python, 클래스설계]
---

# [팜가드] 공유 상태 클래스(GatewayState) 설계하고 직접 구현하기

팀 프로젝트 **팜가드(FarmGuard)** — 스마트팜 게이트웨이의 UDP Flooding 공격 탐지·차단 보안 SW — 를 구현하면서, 처음으로 클래스 하나를 처음부터 끝까지 직접 설계하고 구현해본 기록이다.

## 1. 왜 다시 처음부터 짰나

기존에 돌아가던 프로토타입 코드가 있었지만, 팀원 아무도 그 코드가 왜 그렇게 짜여 있는지 이해하지 못하는 상태였다. 그래서 요구사항과 기능 범위는 이미 확정된 상태라고 가정하고, **설계(클래스 구조)를 먼저 정하고 → 그 설계를 보면서 직접 코드를 짜는** 순서로 처음부터 다시 진행했다.

## 2. 전체 구조: 역할을 6개로 쪼개기

팜가드가 해야 하는 일의 흐름을 그대로 책임으로 나누면 자연스럽게 6개 클래스가 나온다.

```
센서값 읽기 → 공격인지 판단 → (공격이면) 차단 → 저장 → 화면에 보여주기
```

| 역할 | 질문 | 담당 클래스 |
|---|---|---|
| 센서값을 누가 읽어오나? | "지금 온도/습도 얼마야?" | `SensorCollector` |
| 공격인지 누가 판단하나? | "이거 UDP flooding 맞아?" | `AttackDetector` |
| 공격이면 누가 막나? | "저 IP 차단해" | `Blocker` |
| 현재 상태를 누가 기억하나? | "지금 공격 중이야? 차단된 IP는?" | `GatewayState` |
| 기록을 누가 저장하나? | "나중에 조회할 수 있게 DB에 남겨" | `Storage` |
| 사람한테 누가 보여주나? | "지금 상태 웹으로 보고 싶어" | `WebServer` |

한 사람이 한 역할만 수정해도 다른 부분이 깨지지 않도록 하기 위한 책임 분리다. 오늘은 이 중 제일 기초가 되는 **`GatewayState`** 를 구현했다.

## 3. GatewayState란?

`GatewayState`는 시스템 전체가 "지금 무슨 상황인지"를 공유해서 보는 **공유 칠판** 같은 존재다. 직접 센서를 읽거나 공격을 탐지하거나 차단하지 않는다. 그냥 다른 클래스들이 알아낸 결과를 적어두고, 누가 물어보면 알려주는 저장소 역할만 한다.

- `SensorCollector`, `AttackDetector`, `Blocker` → 칠판에 **적기만** 함
- `WebServer` → 칠판을 **읽기만** 함 (대시보드 `/api/status` 응답용)

대시보드가 없더라도 이 클래스는 필요하다 — `AttackDetector`가 "지금 보호 모드 켜져 있나?"를 확인해서 탐지 로직을 돌릴지 말지 판단해야 하기 때문이다. 즉 본질은 **시스템 내부 로직끼리 정보를 주고받기 위한 공용 저장소**이고, 대시보드는 그중 "사람한테 보여주는" 창구 하나일 뿐이다.

## 4. 왜 Lock이 필요한가 — 경쟁 상태(Race Condition)

`GatewayState`는 여러 스레드(센서 수집, 공격 탐지, 웹 요청 처리)가 동시에 읽고 쓰는 공유 데이터다. 아래처럼 한 줄처럼 보이는 코드도 실제로는 "읽기 → 계산 → 쓰기" 세 단계로 쪼개져 실행된다.

```python
self._events = [new_event] + self._events  # ① 읽기 ② 계산 ③ 쓰기
```

스레드 A와 B가 동시에 ①을 수행하면, 둘 다 "옛날" 상태를 기준으로 계산해서 쓰기(③)를 하게 되고, **먼저 쓴 결과가 나중에 쓴 쪽에 덮어써져서 사라져버린다.** 느려지는 게 아니라 **데이터가 유실**되는 문제다.

해결책은 `threading.Lock()`:

```python
with self._lock:
    self._events = [new_event] + self._events
```

`with` 블록은 시작할 때 자동으로 `acquire()`, 끝나면(에러가 나도) 무조건 자동으로 `release()`를 해줘서, 그 구간 안에는 한 번에 하나의 스레드만 들어올 수 있게 보장한다.

또한 "보호 모드 켜기" 같은 동작은 변수 하나가 아니라 **여러 변수가 한 세트로 같이** 바뀌어야 하는데, 이걸 하나씩 따로 바꾸면 그 사이 "어중간한 상태"(예: `protected=True`인데 `mode`는 아직 이전 값)를 다른 스레드가 읽어가는 문제가 생긴다. 그래서 관련 변수들을 하나의 메소드로 묶어 Lock으로 통째로 감싼다.

## 5. 변수(속성) 설계 — 화면에 뭘 보여줘야 하는지부터 거꾸로 생각

```python
def __init__(self):
    self._lock = threading.Lock()
    self._protected = False
    self._mode = "정상"
    self._attacker = None
    self._msg = "정상 동작 중입니다."
    self._latest = None
    self._events = []
    self._overloaded = False
```

| 변수 | 의미 |
|---|---|
| `_protected` | 팜가드 보호 기능 켜짐/꺼짐 |
| `_mode` | "정상" / "공격 감지" — 신호등 색깔 결정 |
| `_attacker` | 공격자 IP (없으면 None) |
| `_msg` | 화면에 보여줄 한 줄 설명 |
| `_latest` | 방금 들어온 센서값 (딕셔너리, 매번 덮어씀) |
| `_events` | 최근 이벤트 로그 (리스트, 계속 쌓임) |
| `_overloaded` | 무방비 상태에서 실제로 센서 수집이 끊겼는지 |

`_overloaded`가 따로 필요한 이유: `_mode`는 `AttackDetector`가 "탐지해서 아는" 상태인데, 보호 모드가 꺼져 있으면 애초에 탐지 로직 자체가 안 돈다. 그래서 무방비 상태에서 공격이 들어와도 `_mode`는 영원히 "정상"으로 거짓말을 하게 된다. `_overloaded`는 진단(탐지) 없이도 "센서 수집이 실제로 멈췄다"는 증상만으로 위험을 알리는 변수다 — "팜가드가 없으면 공격당하는 줄도 모른 채 그냥 멈춰버린다"는 데모의 핵심 메시지를 표현하기 위한 설계.

## 6. 메소드 7개 — 최종 코드

```python
import threading
from datetime import datetime


class GatewayState:
    def __init__(self):
        self._lock = threading.Lock()
        self._protected = False
        self._mode = "정상"
        self._attacker = None
        self._msg = "정상 동작 중입니다."
        self._latest = None
        self._events = []
        self._overloaded = False

    def toggle_protection(self):
        with self._lock:
            self._protected = not self._protected
            self._attacker = None
            self._overloaded = False
            self._mode = "정상"
            if self._protected:
                self._msg = "정상 동작 중입니다."
            else:
                self._msg = "팜가드가 꺼져있습니다."

    def markAttack(self, ip):
        with self._lock:
            self._attacker = ip
            self._mode = "공격탐지"
            self._msg = f"{ip}에서 공격이 탐지되었습니다."
            now = datetime.now().strftime("%H:%M:%S")
            self._events = [{"ts": now, "kind": "DETECT", "detail": self._msg}] + self._events

    def markRecovery(self, ip):
        with self._lock:
            self._attacker = None
            self._mode = "정상"
            self._msg = f"{ip}로부터 받은 공격을 복구했습니다."
            now = datetime.now().strftime("%H:%M:%S")
            self._events = [{"ts": now, "kind": "RECOVER", "detail": self._msg}] + self._events

    def setLatest(self, sample):
        with self._lock:
            self._latest = sample

    def setOverloaded(self, value):
        with self._lock:
            self._overloaded = value

    def is_protected(self):
        with self._lock:
            return self._protected

    def snapshot(self):
        with self._lock:
            return {
                "protected": self._protected,
                "mode": self._mode,
                "attacker": self._attacker,
                "msg": self._msg,
                "latest": self._latest,
                "events": self._events,
                "overloaded": self._overloaded,
            }
```

쓰기 메소드(`toggle_protection`, `markAttack`, `markRecovery`, `setLatest`, `setOverloaded`)는 상태를 바꾸고, 읽기 메소드(`is_protected`, `snapshot`)는 상태를 바꾸지 않고 꺼내서 보여주기만 한다는 점이 구분 포인트였다.

## 7. 짜면서 실수했던 것들

- `with self._lock:` 자체를 빼먹음 — "Lock으로 감싼다"는 게 추상적인 개념이 아니라, 실제로 코드 블록을 들여쓰기해서 그 안에 넣는 행위라는 걸 체감
- `self._events = {...}` 처럼 **리스트를 통째로 딕셔너리로 덮어씀** — 리스트에 "추가"하려면 `[새항목] + 기존리스트` 형태로 기존 것을 이어 붙여야 한다는 걸 두 번 반복해서 놓침
- `self.msg` vs `self._msg` 오타, `strtime` vs `strftime` 오타, f-string 접두사(`f`)를 따옴표 안쪽에 넣는 실수 — 작은 문법 실수가 실제로 제일 자주 나옴
- "이미 초기값인데 또 리셋해야 하나?"라는 질문 — `__init__`의 초기값은 프로그램이 시작된 "그 순간"에만 유효하고, 프로그램이 돌아가면서 값은 계속 바뀌기 때문에 상태 전환(토글, 복구) 시점마다 명시적으로 리셋해줘야 한다는 걸 깨달음

## 8. 다음 할 일

같은 방식으로 `Storage`(DB 저장) → `SensorCollector` → `AttackDetector`(연구 주제와도 연결되는 핵심 탐지 로직) → `Blocker` → `WebServer` 순으로 이어서 구현 예정.
