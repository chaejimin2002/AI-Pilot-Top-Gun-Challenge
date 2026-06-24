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

## 공통 셋업 (전원 최초 1회)

```powershell
git clone https://github.com/chaejimin2002/AI-Pilot-Top-Gun-Challenge.git
cd AI-Pilot-Top-Gun-Challenge/DogFightEnv/Release

conda create -n aip python=3.11
conda activate aip
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
2. Visual Studio → AIP_DCS 빌드
3. 빌드된 DLL → DogFightEnv/Release/AIP_BASE.dll 교체
4. python run_local_dogfight.py --ownship-backend bt --target-backend bt
5. git push (소스 + DLL 함께)
```

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

## 파트 간 협업 규칙

| 상황 | 방법 |
|------|------|
| BT / JSBSim DLL 업데이트 | push 후 다른 파트 `git pull` → DLL 자동 반영 |
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
