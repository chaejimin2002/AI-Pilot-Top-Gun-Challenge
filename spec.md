# 2026 AI Pilot Top Gun Challenge 프로젝트 수행 핵심 정리

> 목적: 대회 수행에 직접 필요한 **교전 규칙, 개발 환경, 학습/검증/제출 흐름, 실험 관리, 금지사항**만 압축 정리한 문서입니다.  
> 기준 자료: `2026 AI Pilot Top Gun Challenge`, `공식 규정집 v1.4`, `AIP 경진대회 매뉴얼 rev7`.

---

## 1. 프로젝트 목표

이 프로젝트의 핵심은 **공식 공중전 시뮬레이터에서 1:1 교전을 수행하는 AI 파일럿 에이전트**를 개발하는 것입니다.

참가팀은 제공된 개발 환경을 이용해 다음 중 하나 또는 복합 방식으로 AI를 구현합니다.

- **Behavior Tree 기반 규칙형 AI**
- **강화학습 기반 AI**
- **BT + RL Hybrid AI**

최종적으로 AI는 교전 서버가 주는 전장 정보를 바탕으로 매 프레임 조종 명령을 반환해야 합니다.

---

## 2. 반드시 기억해야 할 교전 구조

### 2.1 전장 정보

대회는 **Perfect State Information**을 사용합니다.

즉, 센서 오차, 탐지 지연, 노이즈, 가시성 제한 없이 시뮬레이터의 실제 상태값이 AI에게 제공됩니다.

AI가 기본적으로 활용할 수 있는 정보는 다음과 같습니다.

```text
Ownship state
Target state
Location: X, Y, Z
Rotation: Roll, Pitch, Yaw
Velocity: u, v, w
```

따라서 프로젝트의 핵심은 센서 추정이 아니라 **상황 판단 → 기동 결정 → 안정적 조종 명령 생성**입니다.

### 2.2 통신 주기

교전 서버는 **60 Hz**로 전장 정보를 전송합니다.

```text
DeltaTime = 0.016666 sec
1초당 60번 전장정보 수신 및 CMD 응답
```

중요한 점은 AI가 꼭 매번 새 판단을 1/60초마다 할 필요는 없지만, **전장정보에 대한 조종 명령 응답은 60 Hz로 반환해야 한다**는 것입니다.

AI 연산 응답시간이 지나치게 길면 패널티가 발생할 수 있으므로, 추론 코드는 반드시 가볍게 유지해야 합니다.

---

## 3. 승패와 교전 규칙

### 3.1 기본 승리 조건

라운드에서 다음 조건 중 하나를 먼저 만족하면 승리합니다.

1. 상대 기체 HP를 0으로 만들어 격추
2. 상대 기체 추락 유도
3. 상대 에이전트 응답 불능
4. 제한 시간 종료 시 잔여 HP 비율 우위

### 3.2 라운드 시간

규정집 기준 라운드 시간은 **1라운드 최대 2분**입니다.

교육자료에는 교전 시간 200초 기준의 설명도 포함되어 있으므로, 실제 평가 환경에서는 운영 공지를 우선 확인해야 합니다.

### 3.3 추락 조건

교육자료 기준으로는 **1000 ft 이하 고도 도달 시 추락 처리**가 언급됩니다.

규정집에서는 **고도 100 m 이하 30초 이상 지속 또는 지형 충돌**을 추락 조건으로 제시합니다.

따라서 학습 보상 설계에서는 보수적으로 다음 기준을 두는 것이 안전합니다.

```text
고도 1000 ft 근처 접근 시 강한 penalty
급격한 pitch/roll로 지면 접근하는 행동 억제
```

---

## 4. 대미지 / WEZ 규칙

### 4.1 기본 공격 개념

공격은 미사일이 아니라 **기총 기반 초근접 교전**입니다.

적기의 중심이 자신의 공격 Cone 안에 들어오면 대미지가 발생합니다.

기본 AlphaDogFight 방식은 다음 조건을 사용합니다.

```text
Distance: 500 ft ~ 3000 ft
LOS: 1 degree 이하
```

즉, AI의 핵심 목표는 단순히 가까이 가는 것이 아니라,

```text
적을 내 기수 방향의 좁은 Cone 안에 넣고
500~3000 ft의 유효 거리대를 유지하는 것
```

입니다.

### 4.2 경진대회 Phase별 공격 판정

교전 시간이 지날수록 공격 성공 판정 범위가 넓어집니다.

| Phase | 시간 | LOS 조건 | 거리 조건 | 대미지 계수 |
|---|---:|---:|---:|---:|
| Phase 1 | 0~100s | LOS < 1° | 500~3000 ft | 1.0 |
| Phase 2 | 100~150s | LOS < 2° | 500~3500 ft | 0.3 |
| Phase 3 | 150~200s | LOS < 3° | 500~4000 ft | 0.1 |

하위 Phase의 더 강한 조건을 만족하면 해당 Phase의 대미지가 적용됩니다.

예를 들어 Phase 3 시점이라도 적기가 Phase 1 조건 안에 있으면 Phase 1 대미지가 적용됩니다.

### 4.3 보상 설계에 반영할 핵심

좋은 보상은 다음 항목을 반영해야 합니다.

```text
LOS 감소
유효 거리 유지
WEZ 내부 체류 시간 증가
상대 후방/기수 정렬 우위
자기 고도 안정성
자기 속도/에너지 유지
급격한 조종으로 인한 crash 방지
```

단, 대회 매뉴얼은 “정답 보상”을 주는 것이 아니라 실험자가 보상 아이디어를 설계하는 구조입니다.

---

## 5. 대회 진행 구조

### 5.1 예선

예선은 온라인 대전이며, 스위스 시스템 방식으로 진행됩니다.

핵심은 다음과 같습니다.

```text
단판 경기
최소 3회 이상 교전
상위 8팀 + 와일드카드 4팀 본선 진출
```

와일드카드는 상위 8팀과 소속 대학이 중복되지 않는 팀을 대상으로 하며, 동일 대학에서는 최대 1팀만 선발될 수 있습니다.

### 5.2 컷오프

필요시 예선 스위스 시스템 이전에 컷오프가 운영될 수 있습니다.

컷오프 통과 조건은 운영위원회가 지정한 기체를 상대로 승리하는 것입니다.

따라서 초반 개발 목표는 우승 전략보다도 먼저,

```text
운영진 제공 baseline 또는 지정 기체를 안정적으로 이기는 정책
```

이어야 합니다.

### 5.3 본선

본선은 다음 구조입니다.

```text
3개 팀씩 4개 조
각 조 3판 2승 풀리그
조별 상위 2팀이 8강 토너먼트 진출
8강 토너먼트는 5판 3선승제
```

---

## 6. 개발 환경 구조

### 6.1 Release 폴더를 루트로 사용

작업은 반드시 `Release/`를 루트로 두고 수행합니다.

```text
Release/
├─ src/dogfight/              # 공통 플랫폼
├─ student/                   # 학생 작성 영역
├─ experiments/               # YAML 실험 템플릿
├─ train_rllib.py             # 단일 학습
├─ train_curriculum.py        # 커리큘럼 학습
├─ run_local_dogfight.py      # 로컬 교전 검증
├─ run_unreal_inference.py    # Unreal 서버 접속
└─ DLL / Rule XML / aircraft / engine / scripts
```

### 6.2 수정 중심

주로 수정해야 할 곳은 다음입니다.

```text
student/
experiments/*.yaml
```

반대로 다음은 기본적으로 직접 수정하지 않는 것이 좋습니다.

```text
src/dogfight/
DLL
XML 기본 자산
aircraft/
engine/
scripts/
```

---

## 7. Python 환경 세팅

권장 환경은 **Anaconda + Python 3.11.x**입니다.

기본 설치 흐름은 다음과 같습니다.

```bash
conda create -n aip python=3.11
conda activate aip

cd DogFightEnv\Release
python -m pip install -r requirements.txt
```

설치 직후에는 바로 학습하지 말고 다음 두 가지를 먼저 확인합니다.

```bash
python scripts\run_experiment.py experiments\student_sac_mlp.yaml --dry-run

python -c "import JSBSimWrapper; print('OK')"
```

SAC LSTM을 사용할 경우 Ray/RLlib 패치가 필요할 수 있습니다.

```bash
python RLLibLstm\tools\apply_rllib_sac_lstm_patch.py C:\Users\USER\anaconda3\envs\aip --dry-run
```

---

## 8. 환경 계약: Observation / Action / Reward / Termination

대회 환경은 크게 네 가지 계약으로 움직입니다.

```text
Observation: 정책 입력 벡터
Action: 4차원 연속 제어
Reward: float reward + components
Termination / Info: 종료 사유와 로그 해석 근거
```

이 계약이 깨지면 학습, 로컬 검증, 제출 실행이 모두 불안정해집니다.

---

## 9. Observation 설계

### 9.1 기본 관측 모드

제공되는 기본 관측 모드는 다음과 같습니다.

| 모드 | 차원 | 의미 |
|---|---:|---|
| classic12 | 12 | 기본 비교용 |
| relative14 | 14 | 상대 위치 중심 |
| tactical16 | 16 | 기본 제공 예시 |
| legacy37 | 37 | 기존 확장 관측 |
| custom | 직접 선언 | 학생 커스텀 관측 |

`tactical16`은 기본 예시일 뿐, 반드시 최적 관측이라는 뜻은 아닙니다.

### 9.2 Custom observation 계약

`student/my_observation.py`에 다음 형태로 작성합니다.

```python
OBSERVATION_MODE = "student8"
OBSERVATION_SIZE = 8

def build_observation(ownship_state, target_state, geo_info, wez_config=None):
    obs = [...]
    return np.asarray(obs, dtype=np.float32)
```

필수 조건은 다음입니다.

```text
반환 shape == OBSERVATION_SIZE
dtype == np.float32 권장
1-D vector 권장
학습 / 로컬 / Unreal 모두 같은 module path 사용
```

### 9.3 관측 변경 시 주의

관측 차원 또는 module이 바뀌면 기존 checkpoint/bundle과 호환되지 않습니다.

따라서 observation을 바꾼 실험은 반드시 별도 `output_name`, `output_tag`를 사용해야 합니다.

---

## 10. Action 설계

정책이 반환하는 Action은 4차원 연속 제어입니다.

| Index | 의미 | 범위 |
|---:|---|---|
| 0 | roll | [-1, 1] |
| 1 | pitch | [-1, 1] |
| 2 | rudder / yaw | [-1, 1] |
| 3 | throttle | [-1, 1] → 내부에서 [0, 1] 변환 |

초기 정책은 action이 과격해 crash가 자주 발생할 수 있으므로, 보상과 action smoothing 또는 penalty 설계를 함께 고려해야 합니다.

---

## 11. Reward 설계

### 11.1 작성 위치

보상은 보통 다음 파일에 작성합니다.

```text
student/my_reward.py
```

기본 반환 계약은 다음입니다.

```python
MY_REWARD_CONFIG = { ... }

def compute_reward(...):
    components = {}
    total = ...
    return float(total), components
```

### 11.2 components의 의미

`components`에 넣은 값은 `ep_reward_<name>` 형태로 기록되어 metric 분석에 사용됩니다.

예시:

```python
components = {
    "los": los_reward,
    "distance": distance_reward,
    "altitude": altitude_penalty,
    "wez": wez_reward,
}
```

### 11.3 추천 reward 구성

초기 프로젝트에서는 reward를 너무 복잡하게 만들기보다 다음 정도로 나누는 것이 좋습니다.

```text
survival_reward
altitude_penalty
distance_reward
los_reward
wez_reward
damage_reward
crash_penalty
energy_penalty
```

실험할 때는 한 번에 전부 바꾸지 말고 **가설 1개, 조건 1개만 변경**하는 방식이 좋습니다.

---

## 12. Curriculum 설계

커리큘럼은 다음 파일에서 관리합니다.

```text
student/my_curriculum.py
```

작성 계약은 다음과 같습니다.

```python
from dogfight.ai.curriculum import CurriculumStage

def get_stages() -> list[CurriculumStage]:
    return [
        CurriculumStage(
            name=...,
            initial_scenario=...,
            ...
        ),
        ...
    ]
```

기본 curriculum은 대략 다음 흐름을 참고합니다.

```text
Stage 0~3: survival / pursuit / WEZ / autopilot
Stage 4~13: two_circle_headon a000~a180
Stage 14: full_dogfight
```

실행 시에는 다음 인자를 사용합니다.

```bash
--stages-module student.my_curriculum
```

---

## 13. YAML 실험 관리

### 13.1 YAML의 역할

YAML은 실험 조건을 보존하는 파일입니다.

보상, 관측, 알고리즘, runtime 조건을 YAML에 남겨야 나중에 어떤 실험이 왜 좋아졌는지 추적할 수 있습니다.

대표 템플릿은 다음입니다.

```text
student_sac_mlp.yaml
student_ppo_mlp.yaml
student_sac_lstm.yaml
student_ppo_lstm.yaml
student_mixed_initial_sac_mlp.yaml
student_mixed_initial_sac_lstm.yaml
```

### 13.2 dry-run

새 실험 전에 항상 dry-run을 먼저 수행합니다.

```bash
python scripts\run_experiment.py experiments\student_sac_mlp.yaml --dry-run
```

dry-run에서 확인할 것:

```text
YAML → CLI 변환 결과
output.name / tag
선택된 reward module
선택된 observation module
env_config nested 설정
```

---

## 14. 학습 실행

### 14.1 짧은 smoke test

처음에는 좋은 reward를 찾는 것이 아니라 실행 가능성을 확인합니다.

```bash
python train_rllib.py --algorithm sac --iterations 1 --output-name team01 --output-tag smoke
```

또는 wrapper를 사용할 수 있습니다.

```bash
python student\my_train.py --iterations 1
```

확인할 것:

```text
Ray/RLlib 의존성 문제 없음
관측 shape 오류 없음
reward 반환 오류 없음
bundle 저장 성공
```

### 14.2 알고리즘 선택

| 알고리즘 | 특징 | 추천 용도 |
|---|---|---|
| PPO | 업데이트 폭 제한, 비교적 안정적 | 안정 baseline |
| SAC | entropy 기반 탐색, 연속 제어에 적합 | 기동 탐색, continuous control |
| LSTM/RNNSAC | 시계열 기억 사용 가능 | 관측 history가 필요할 때 |

SAC LSTM은 RLlib 패치가 필요할 수 있습니다.

---

## 15. Checkpoint / Bundle 차이

### 15.1 Native checkpoint

```bash
--restore-checkpoint
```

보존되는 것:

```text
policy
optimizer
replay buffer
```

중단된 학습을 그대로 이어갈 때 사용합니다.

### 15.2 Lightweight bundle

```bash
--init-bundle
```

보존되는 것:

```text
policy weight only
```

좋은 정책을 seed로 사용해 새 실험을 시작할 때 적합합니다.

### 15.3 주의

```text
--restore-checkpoint와 --init-bundle은 동시에 사용하지 않기
OBSERVATION_SIZE 또는 observation_module 변경 시 기존 bundle 재사용 금지
```

---

## 16. 모델 Bundle 구조

학습 모델의 제출 단위는 bundle입니다.

```text
artifacts/models/<team>/<tag>/
├─ metadata.json
└─ policy_weights.pkl.gz
```

`metadata.json`에서 반드시 확인할 값:

```text
algorithm
obs_mode
observation_module
reward_module
use_lstm_sac
lstm_scope
max_seq_len
```

제출/검증에서 `metadata.json`과 실제 설정이 불일치하면 observation mismatch가 발생합니다.

---

## 17. 로컬 검증

### 17.1 기본 검증 명령

```bash
python run_local_dogfight.py ^
  --ownship-backend rl ^
  --ownship-bundle-dir artifacts\models\team01\v1 ^
  --target-backend bt ^
  --save-log
```

### 17.2 Custom observation 검증

```bash
python run_local_dogfight.py ^
  --ownship-backend rl ^
  --ownship-bundle-dir artifacts\models\team01\observation_v1 ^
  --target-backend bt ^
  --observation-mode custom ^
  --observation-module student.my_observation
```

### 17.3 먼저 볼 지표

승률보다 먼저 봐야 할 것은 실패 원인입니다.

```text
Crash / Altitude: 비행 안정성 실패
FDM / NaN: 시뮬레이터 상태 오류
Observation mismatch: mode/module/shape 불일치
Distance / WEZ: 접근과 공격 기회 부족
```

---

## 18. Unreal / 대회 서버 접속

### 18.1 RL 제출 실행

```bash
python run_unreal_inference.py --mode rl ^
  --bundle-dir artifacts\models\team01\sac_mlp_v1 ^
  --team-name team01 ^
  --server-ip <IP> ^
  --server-port 9999 ^
  --action-repeat 6
```

### 18.2 Custom observation 제출 실행

```bash
python run_unreal_inference.py --mode rl ^
  --bundle-dir artifacts\models\team01\observation_v1 ^
  --observation-mode custom ^
  --observation-module student.my_observation ^
  --team-name team01 ^
  --server-ip <IP> ^
  --server-port 9999 ^
  --action-repeat 6
```

### 18.3 Hybrid 제출 실행

```bash
python run_unreal_inference.py --mode hybrid ^
  --bundle-dir artifacts\models\team01\sac_mlp_v1 ^
  --bt-dll AIP_BASE.dll ^
  --bt-rule-xml Rule_팀이름.xml ^
  --team-name team01 ^
  --server-ip <IP> ^
  --server-port 9999
```

또는 다음 파일을 수정해 실행할 수 있습니다.

```bash
python student\my_submission.py
```

---

## 19. `my_submission.py` 최종 수정 항목

제출 전 다음 항목을 확인합니다.

| 항목 | 의미 | 체크 |
|---|---|---|
| TEAM_NAME | 팀 표시 이름 | 등록명과 일치 |
| SERVER_IP / PORT | Unreal 서버 주소 | 운영 공지 기준 |
| MODE | rl / bt / hybrid | 실험 의도와 일치 |
| BUNDLE_DIR | 제출 policy 경로 | metadata + weights 존재 |
| OBSERVATION_MODE | policy 입력 계약 | 학습 설정과 일치 |
| OBSERVATION_MODULE | custom module path | 학습/로컬/제출 동일 |
| BT_RULE_XML | BT rule 파일 | Hybrid/BT 사용 시 확인 |
| ACTION_REPEAT | action 반복 간격 | 학습/검증 조건과 비교 |

---

## 20. 로그 분석

실험 결과는 승률 하나로 판단하면 안 됩니다.

다음 로그를 같이 봐야 합니다.

| 로그 | 볼 것 | 용도 |
|---|---|---|
| training_log.csv | reward, crash, win, WEZ, distance | 학습 추세 |
| metrics.jsonl | dashboard scalar | 시각 비교 |
| config.json | YAML / CLI / module 설정 | 조건 비교 |
| Tacview CSV | 위치, 자세, health | 로컬 교전 복기 |
| summary.json | end_condition, outcome | 종료 사유 분석 |

핵심 지표:

```text
reward_mean
crash_rate
win_rate
ep_min_distance
ep_wez_steps
end_condition
```

---

## 21. 금지사항 / 실격 위험

다음은 프로젝트 수행 중 반드시 피해야 합니다.

```text
다른 팀과 코드, 모델, 학습 결과, 핵심 아이디어 공유
타인이 개발한 코드/모델을 자신이 개발한 것처럼 제출
경기 중 인간 개입
경기 중 인터넷, 외부 서버, 클라우드 API 호출
공식 시뮬레이터/운영 시스템 취약점 악용
다른 팀 시스템 무단 접근 또는 방해
승부조작, 담합
합리적 학습 과정 검증 요청에 대한 소명 거부
```

특히 **외부 통신, 코드 도용, 팀 간 자료 공유, 타 팀 시스템 침입**은 실격 및 상금 환수까지 이어질 수 있습니다.

대회 PC에는 운영사무국이 사전 허가한 패키지/라이브러리 외 프로그램을 설치하거나 실행할 수 없습니다.

---

## 22. 프로젝트 수행 우선순위

### 1단계: 실행 환경 안정화

```text
conda 환경 생성
requirements 설치
JSBSimWrapper import 확인
YAML dry-run 확인
1 iteration smoke 학습 확인
```

### 2단계: 기본 baseline 확보

```text
student_sac_mlp.yaml 또는 student_ppo_mlp.yaml 실행
기본 observation으로 학습
로컬 BT 상대 교전
crash_rate / end_condition 확인
```

### 3단계: 보상 개선

```text
고도 안정성 penalty
거리 shaping
LOS shaping
WEZ 체류 reward
damage reward
```

한 번에 하나씩 변경합니다.

### 4단계: observation 개선

```text
상대 위치 벡터
거리
LOS angle
closing rate
상대 방위각
고도 차
속도/에너지
ownship attitude
```

관측 변경 시 기존 bundle과 호환되지 않으므로 새 실험으로 관리합니다.

### 5단계: curriculum 도입

```text
survival
pursuit
WEZ 유지
head-on
full dogfight
```

처음부터 full dogfight만 학습시키면 reward가 sparse해질 수 있습니다.

### 6단계: 로컬 검증 루프

```text
학습
bundle 저장
로컬 교전
로그 분석
실패 원인 분류
다음 가설 설정
```

### 7단계: 제출 환경 일치

```text
YAML
train 설정
metadata.json
run_local_dogfight 설정
my_submission.py / CLI 설정
```

위 항목의 observation mode/module, bundle path, action_repeat이 모두 일치해야 합니다.

---

## 23. 실험 이름 관리 예시

```text
team01/
├─ smoke_sac_mlp
├─ sac_mlp_reward_altitude_v1
├─ sac_mlp_reward_los_v1
├─ sac_mlp_obs_relative_v1
├─ sac_mlp_curriculum_wez_v1
└─ sac_lstm_curriculum_full_v1
```

추천 규칙:

```text
알고리즘_네트워크_변경내용_버전
```

예:

```text
sac_mlp_losreward_v1
ppo_mlp_safealt_v2
sac_lstm_curriculum_v1
```

---

## 24. 자주 나는 오류와 해결

| 증상 | 먼저 확인할 것 | 조치 |
|---|---|---|
| DLL 오류 | Release 루트, 파일명 | 위치/이름 확인 |
| import 실패 | Python/venv/requirements | 가상환경 재확인 |
| 모델 로드 실패 | BUNDLE_DIR, metadata/weights | 경로와 파일 2개 확인 |
| 관측 오류 | mode/module/shape | 학습 설정과 제출 설정 비교 |
| 서버 무응답 | IP/port/방화벽 | 운영 공지와 네트워크 확인 |

---

## 25. 최종 제출 전 체크리스트

```text
[ ] Release 루트에서 실행하는가?
[ ] student/ 수정 파일이 정리되어 있는가?
[ ] experiments/*.yaml에 실험 조건이 남아 있는가?
[ ] 최종 bundle에 metadata.json이 있는가?
[ ] 최종 bundle에 policy_weights.pkl.gz가 있는가?
[ ] metadata의 obs_mode와 제출 OBSERVATION_MODE가 같은가?
[ ] metadata의 observation_module과 제출 OBSERVATION_MODULE이 같은가?
[ ] custom observation의 OBSERVATION_SIZE가 실제 반환 shape와 같은가?
[ ] run_local_dogfight로 제출 bundle 검증을 했는가?
[ ] summary.json에서 종료 사유가 정상적인가?
[ ] crash_rate가 과도하지 않은가?
[ ] Unreal 접속 IP/PORT가 운영 공지와 같은가?
[ ] TEAM_NAME이 등록명과 같은가?
[ ] 외부 통신/API 호출이 없는가?
[ ] 허가되지 않은 패키지/프로그램을 사용하지 않는가?
```

---

## 26. 개발 전략 요약

가장 현실적인 개발 전략은 다음입니다.

```text
1. 기본 SAC/PPO MLP baseline을 빠르게 성공시킨다.
2. crash를 줄이는 안전 보상을 먼저 넣는다.
3. Distance + LOS + WEZ 중심으로 공격 기회를 늘린다.
4. 관측은 작고 해석 가능한 feature부터 시작한다.
5. curriculum으로 survival → pursuit → WEZ → full dogfight 순서로 확장한다.
6. 실험은 YAML과 output tag로 관리한다.
7. 최종 제출에서는 observation/module/bundle 일관성을 최우선으로 확인한다.
```

이 대회에서 중요한 것은 한 번에 완벽한 AI를 만드는 것이 아니라,

```text
작은 가설
짧은 학습
로그 확인
실패 원인 분류
다음 실험
```

의 루프를 빠르게 반복하는 것입니다.
