# AI Pilot Top Gun Challenge

F-16 1v1 공중전 AI 개발 프로젝트.  
BT(Behavior Tree), JSBSim(비행 물리), RL(강화학습) 세 파트로 구성됩니다.

---

## 프로젝트 구조

```
AIP_LIB/
├── AIP_DCS/                  # C++ DLL 소스 (BT + Controller)
│   ├── BehaviorTree/         # BT 노드, Rule 실행 엔진
│   └── Geometry/             # Controller_CY (VP → 조종 명령)
├── DogFightEnv/Release/      # Python 환경
│   ├── JSBSimWrapper.py      # JSBSim DLL ctypes 래퍼
│   ├── FighterSim.py         # 시뮬레이션 클래스
│   ├── src/dogfight/         # 환경, 보상, 관측, RL 인터페이스
│   ├── student/              # 팀원 수정 파일 (보상/관측/커리큘럼)
│   ├── experiments/          # 학습 설정 YAML
│   ├── train_rllib.py        # 단일 스테이지 학습
│   ├── train_curriculum.py   # 커리큘럼 학습
│   ├── run_local_dogfight.py # 로컬 시뮬 테스트
│   └── run_unreal_inference.py # 언리얼 서버 연결 (대회 제출)
├── Rule.xml                  # BT 기본 Rule
└── Windows/                  # 언리얼 뷰어 (별도 배포, git 미포함)
```

---

## 시뮬레이터 작동 구조

### 전체 흐름

```
run_local_dogfight.py
        │
        ▼
  DogFightEnv  (Gym 환경 / single_agent_env.py)
┌────────────────────────────────────────────────────────────┐
│ env.step() 루프                                             │
│                                                            │
│ for i in range(6):  ← 1 RL step = 6 physics steps          │
│                                                            │
│   ┌──────────────────────┐      ┌──────────────────────┐   │
│   │ ownship              │      │ target               │   │
│   │ ActionProvider       │      │ ActionProvider       │   │
│   │ (RL / BT / Hybrid)   │      │ (BT / fixed)         │   │
│   └──────────┬───────────┘      └──────────┬───────────┘   │
│              ▼                             ▼               │
│   FighterSim.JSBSim             FighterSim.JSBSim          │
│   51-state vector               51-state vector            │
│              └──────────────┬──────────────┘               │
│                             ▼                              │
│                    JSBSimAIPLib.dll                        │
│          C++ flight dynamics engine, 60 Hz                 │
│                             │                              │
│                    update_damage()                         │
│                    WEZ damage calculation                  │
│                                                            │
│ evaluate_termination()  → termination check                │
│ compute_reward()        → reward calculation               │
└────────────────────────────────────────────────────────────┘
```

### 1 스텝의 의미

| 단위 | 주파수 | 설명 |
|------|--------|------|
| JSBSim step | 60 Hz | 비행 역학 물리 연산 1회 |
| RL step | 10 Hz | 신경망 추론 1회 (6 JSBSim step 묶음) |
| WEZ 피해 계산 | JSBSim step마다 | 거리·각도 조건 충족 시 health 차감 |

### ActionProvider — RL/BT/Hybrid 교체 지점

세 백엔드는 동일한 `ActionProvider` 인터페이스를 구현하므로 `--ownship-backend`, `--target-backend` 인자 하나로 교체됩니다.

| 백엔드 | 동작 | DLL 필요 |
|--------|------|----------|
| `bt` | `AIP_BASE.dll` C++ BT 로직 실행 → 제어값 반환 | O |
| `rl` | PyTorch 신경망 `forward_inference()` → 제어값 반환 | X |
| `hybrid` | BT 기반값 + RL 잔차(residual) / 선형(blend) / 선택(switch) | O |

### 상태·관측·보상

```
JSBSim FDM 출력 (230 byte C 구조체)
    └─ _update_state() → 51개 상태 벡터
            │
            ▼
    build_observation() → 관측 벡터 (RL 입력)
      ├─ classic12  : 위치 6 + 자세 6
      ├─ relative14 : 상대 위치 + 각도 + 거리
      └─ tactical16 : 자세·속도·고도 + 상대 위치 + WEZ 정보 (권장)
            │
            ▼
    compute_reward()
      ├─ step_penalty    : 매 스텝 -0.01 (시간 효율)
      ├─ pursuit         : ATA × 거리 그래디언트 (추격 형성)
      ├─ damage          : target_damage - ownship_damage 미분
      ├─ safety          : 고도 300m 미만 페널티
      └─ terminal        : 승(+100) / 패(-100) / 무승부(-30)
```

### 종료 조건

| 조건 | 종류 |
|------|------|
| 고도 300m 미만 | terminated |
| health ≤ 0 (피해 누적) | terminated |
| 연료 소진 / FDM 오류 | terminated |
| 교전 시간 300초 초과 | truncated |
| 에피소드 스텝 18000 초과 | truncated |

### 각 파트가 건드리는 계층

| 파트 | 수정 위치 | 빌드 필요 |
|------|-----------|-----------|
| BT | `Rule.xml`, `AIP_DCS/BehaviorTree/` C++ 노드 | Visual Studio (DLL) |
| JSBSim | `AIP_DCS/Geometry/Controller_CY.cpp` | Visual Studio (DLL) |
| RL | `student/my_reward.py`, `my_observation.py`, `my_curriculum.py` | X (Python) |

---

## 공통 셋업 (전원 최초 1회)

```powershell
git clone https://github.com/chaejimin2002/AI-Pilot-Top-Gun-Challenge.git

# 가상환경 생성 (위치 무관, 최초 1회)
conda create -n aip python=3.11
conda activate aip

# 패키지 설치 및 실행은 Release/ 폴더에서
cd AI-Pilot-Top-Gun-Challenge/DogFightEnv/Release
pip install -r requirements.txt
```

**동작 확인:**
```powershell
python run_local_dogfight.py --ownship-backend bt --target-backend bt
```

---

## 파트별 작업 가이드

### BT 파트

> 필요 도구: Visual Studio (C++ 빌드)

**주요 파일:**
- `Rule.xml` — 트리 구조 (XML)
- `AIP_DCS/BehaviorTree/BT_Content/` — 노드 로직 (C++)
- `DogFightEnv/Release/AIP_BASE.dll` — 빌드 결과물

**작업 흐름:**
```
1. Rule.xml 또는 AIP_DCS/ C++ 노드 수정
2. Visual Studio → AIP_DCS 빌드  (아래 빌드 가이드 참고)
3. 빌드된 DLL → DogFightEnv/Release/AIP_BASE.dll 교체
4. python run_local_dogfight.py --ownship-backend bt --target-backend bt
5. git push (소스 + DLL 함께)
```

**AIP_BASE.dll 빌드 가이드 (Visual Studio 2022):**
```
1. AIP_DCS/AIP_DCS.sln 파일을 더블클릭하여 Visual Studio로 열기
2. 상단 툴바에서 구성을 Debug → Release, 플랫폼을 x64 로 변경
3. Ctrl+Shift+B  (또는 빌드 → 솔루션 빌드)
4. 빌드 성공 시 출력 경로: AIP_DCS/x64/Release/AIP_DCS.dll
5. 해당 파일을 DogFightEnv/Release/AIP_BASE.dll 로 복사
```

> Debug 빌드로 실수하면 시뮬 성능이 크게 느려집니다. 반드시 **Release | x64** 확인 후 빌드하세요.

---

### JSBSim / Controller 파트

> 필요 도구: Visual Studio (C++ 빌드)

**주요 파일:**
- `AIP_DCS/Geometry/Controller_CY.cpp` — VP → 조종 명령 변환
- `DogFightEnv/Release/aircraft/f16/f16_init.xml` — 항공기 초기 조건
- `DogFightEnv/Release/JSBSimAIPLib.dll` — JSBSim 빌드 결과물

**작업 흐름:**
```
1. Controller_CY.cpp 또는 aircraft xml 수정
2. Visual Studio → JSBSimAIPLib 빌드 후 DLL 교체
3. python run_local_dogfight.py --ownship-backend bt --target-backend bt
4. git push
```

---

### RL 파트

> Visual Studio 불필요, Python만으로 작업 가능

**주요 파일:**
- `student/my_reward.py` — 보상 함수
- `student/my_observation.py` — 관측 벡터 (선택)
- `student/my_curriculum.py` — 커리큘럼 (선택)
- `experiments/*.yaml` — 학습 설정

**작업 흐름:**
```powershell
# 1. student/ 파일 수정 후 학습
python scripts/run_experiment.py experiments/student_sac_mlp.yaml

# 2. 학습 결과 확인
python tools/dashboard.py --port 7860
# 브라우저 → http://127.0.0.1:7860

# 3. RL vs BT 테스트
python run_local_dogfight.py `
  --ownship-backend rl `
  --ownship-bundle-dir artifacts/models/<팀이름>/<버전> `
  --target-backend bt

# 4. git push (student/ 파일만)
```

---

## Push 전 필수 확인 (로컬 테스트)

수정 내용에 따라 아래 명령을 실행하고 에러 없이 종료되면 push하세요.

**BT 또는 JSBSim/Controller 수정 시:**
```powershell
cd DogFightEnv/Release
python run_local_dogfight.py --ownship-backend bt --target-backend bt
```

**RL 수정 시 (학습된 모델 필요):**
```powershell
cd DogFightEnv/Release
python run_local_dogfight.py `
  --ownship-backend rl `
  --ownship-bundle-dir artifacts/models/<팀이름>/<버전> `
  --target-backend bt
```

> RL 모델이 아직 없으면 `--dry-run`으로 문법 오류만 먼저 확인:
> ```powershell
> python scripts/run_experiment.py experiments/student_sac_mlp.yaml --dry-run
> ```

---

## 파트 간 협업 규칙

### Git 브랜치 전략

**브랜치 명명 규칙:**
```
<파트>/<작업내용>
예) bt/missile-evade-node
    rl/reward-reshape
    jsbsim/controller-gain-tune
```

**작업 흐름 (필수):**
```powershell
# 1. main 최신화
git checkout main
git pull origin main

# 2. 작업 브랜치 생성
git checkout -b bt/my-feature

# 3. 작업 및 로컬 테스트 (위 파트별 가이드 참고)

# 4. 커밋
git add <수정파일>
git commit -m "bt: 미사일 회피 노드 추가"

# 5. push 후 PR 생성 → main 병합
git push origin bt/my-feature
```

> main 브랜치에 직접 커밋하지 마세요. 반드시 브랜치에서 작업 후 PR로 병합합니다.

### 파트 간 DLL/파일 의존성

| 상황 | 방법 |
|------|------|
| BT / JSBSim DLL 업데이트 | main 병합 후 다른 파트 `git pull` → DLL 자동 반영 |
| RL 학습 기준 BT 필요 | `--target-backend bt` 로 현재 BT DLL 상대로 학습 |
| 전체 통합 테스트 | `run_local_dogfight.py --ownship-backend rl --target-backend bt` |

---

## 언리얼 뷰어 (대회 제출용)

`Windows/` 폴더는 용량 문제로 git에 포함되지 않습니다. 별도로 다운로드 후 프로젝트 루트에 배치하세요.

서버 연결:
```powershell
python run_unreal_inference.py `
  --mode rl `
  --bundle-dir artifacts/models/<팀이름>/<버전> `
  --team-name <팀이름> `
  --server-ip <IP> --server-port 9999
```

---

## 참고

- 상세 학습 옵션: `DogFightEnv/Release/README.md`
- BT 실행 구조: `Rule.xml`, `AIP_DCS/BehaviorTree/`
- 환경 인터페이스: `src/dogfight/envs/single_agent_env.py`

## 파트별 진행 상황

각 파트의 현재 상태, 완료된 작업, AI 컨텍스트는 아래 파일에서 관리합니다.

| 파트 | 파일 |
|------|------|
| BT | `AIP_DCS/BehaviorTree/progress.md` |
| JSBSim / Controller | `AIP_DCS/Geometry/progress.md` |
| RL | `DogFightEnv/Release/student/progress.md` |

AI 툴을 이용할 때 해당 파일을 컨텍스트로 제공하고, 작업 후 내용을 업데이트하세요.
