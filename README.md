# The Seventh Bullet

> Unreal Engine 5.5 기반 3인칭 슈터 게임 · 적 AI 시스템 설계 및 데이터 드리븐 확장 구조 구현

**[▶ 시연 영상]([The Seventhbullet Demo Video - YouTube](https://www.youtube.com/watch?v=4-i-LEJV7RU))** 

---

## 프로젝트 개요

|             |                                                                                    |
| ----------- | ---------------------------------------------------------------------------------- |
| **기간**      | 2026.02 ~ 2026.03 (2개월)                                                            |
| **팀 구성**    | 5명 (클라이언트 5)                                                                       |
| **본인 역할**   | 클라이언트 · 적 AI 시스템 설계 및 구현 · 기획 일부                                                   |
| **엔진 / 언어** | Unreal Engine 5.5 · C++                                                            |
| **주요 기술**   | Behavior Tree, Blackboard, AI Perception, EQS, Primary Data Asset, Unreal Insights |

> 팀 프로젝트입니다. 본 레포는 원본 저장소의 포크본이며, 아래 "본인 구현 범위"에 명시한 부분이 직접 작성한 코드입니다.

---

## 본인 구현 범위

**[전체 커밋 보기](https://github.com/khs081215/7th-Team12-CH3-Project/commits?author=khs081215)**

| 영역                         | 주요 파일                                                        |
| -------------------------- | ------------------------------------------------------------ |
| 적 베이스 클래스 / EnemyBase 통합   | `Source/TheSeventhbullet/Enemy/EnemyBase.h`, `EnemyBase.cpp` |
| AI 컨트롤러                    | `Source/TheSeventhbullet/Enemy/EnemyAIControllerBase.h`      |
| BT Service · Task 노드 (C++) | `Source/TheSeventhbullet/Enemy/BehaviorTree/`                |
| 발사체 · 오브젝트 풀               | `Source/TheSeventhbullet/Enemy/Projectile/ProjectileActor.h` |
| EnemyDataAsset (PDA)       | `Source/TheSeventhbullet/DataAsset/EnemyDataAsset.h`         |
| 보스 패턴 · LevelSequence 컷씬   | `Source/TheSeventhbullet/Enemy/Boss/`                        |

---

## 핵심 구현

### 1. 다종 Enemy 데이터 드리븐 확장 구조 리팩토링

**문제**  
처음엔 WereWolf 1종만 두고 BT·Blackboard·AnimBP가 짜여 있어, 적 5종(WereWolf·Kwang·Rampage·Sparrow·Boss)을 추가하려면 자산을 복제하고 공통 행동(Detect/Attack/Hit/Dead)을 매번 다시 작성해야 했음.

**해결**  
WereWolf 자산을 `EnemyBase`로 승격하고 BT·Blackboard를 단일 공유 트리로 통합. 적마다 달라지는 값(스탯·공격 패턴·감지 특성)만 `EnemyDataAsset`(Primary Data Asset)으로 분리. 신규 적은 DataAsset 하나만 작성하면 공통 트리를 그대로 사용하도록 함. 보스는 공격 패턴 구조가 달라 별도 BT를 유지.

**설계 판단**  
공통 행동 플로우(순찰→추격→공격→피격→사망)가 모든 적에 동일하므로 BT/BB는 통합하고 값만 PDA로 분리하는 방식이 유지보수·확장 양쪽에 유리하다고 판단. 적마다 BT를 별도로 유지하는 방식은 행동 수정 시 모든 BT를 동기화해야 하는 문제가 있어 제외. Skeletal Mesh·Anim Montage 등 무거운 에셋은 `TSoftObjectPtr` + Asset Bundle로 비동기 로딩해 웨이브 시작 전 선행 로드.

**결과**  
신규 적 추가 시 `EnemyDataAsset` 하나만 작성하면 되어, WereWolf → Kwang → Rampage → Sparrow 순서로 적 추가 공수가 크게 낮아짐. (Boss도 PDA에서 BT만 변경하면 동일한 구조를 따른다.)

---

### 2. AI Perception 기반 감지 시스템 도입

**문제**  
거리 기반 감지는 시야각·장애물을 무시하고 반경 내 플레이어를 즉시 감지해 게임플레이상 부자연스러웠음.

**해결**  
AI Perception으로 전환해 Sight·Hearing·Prediction 3종 센스를 조합. 시야각·감지 거리 등 SenseConfig 파라미터는 AIController 블루프린트의 디폴트 값으로 설정. 단, 공격 사거리가 시야 반경보다 길면 런타임에서 SightRadius를 공격 사거리에 맞게 보정(`SightRadius = AttackRadius + 50`)해 감지 누락을 방지. 감지 이벤트가 Blackboard에 대상을 기록하면 BT가 타겟 유무로 추격/순찰을 자동 분기.

**설계 판단**  
SenseConfig 값은 블루프린트 디폴트로 관리해 에디터에서 직접 조정 가능하도록 함. 공격 사거리 보정 로직은 코드로 처리해 SightRadius 설정을 잊었을 때 발생하는 "공격 범위 내인데 감지 못 함" 버그를 방지.

**결과**  
Sight로 플레이어 포착 시 추격, 시야 이탈 시 Prediction으로 예상 위치를 0.25초 뒤에 마킹해 자연스러운 수색 행동 구현. Hearing으로 총성 위치도 Blackboard에 기록해 청각 반응도 동작.

---

### 3. 오브젝트 풀링 성능 검증 (Unreal Insights + CVar)

**문제**  
발사체 풀링을 도입했으나 실제 개선 효과가 수치로 검증되지 않은 상태. 성능 향상을 주장하려면 근거가 필요.

**해결**  
풀 경로와 `SpawnActor` 경로를 `CVar(ECVF_Cheat)` 런타임 토글로 분리해 ON/OFF 각 5회 측정. GPU 트랙을 대조군으로 두어 측정 간 머신 상태 변동을 배제. 스폰 집계에 발사체 외 액터가 섞인 것을 발견해 `TRACE_CPUPROFILER_EVENT_SCOPE`로 생성·소멸 구간만 격리해 재측정.

**설계 판단**  
단순 FPS 비교 대신 CPU/GPU를 분리해 병목 위치를 먼저 파악. 프레임 병목이 GPU에 있음을 확인했고, 발사체 풀링의 절감분이 프레임타임의 **0.01% 미만**으로 측정 편차 범위 내에 있어 FPS 개선을 수치로 주장하지 않기로 결정. 스폰 빈도가 더 높은 시나리오에서는 효과가 있을 수 있으므로 풀 구조 자체는 유지.

**결과**  
이 프로젝트 조건에서는 풀링의 FPS 개선 효과가 측정 편차를 넘지 못함을 확인. 검증 방법론(CVar 토글 + CPU 프로파일러 격리)은 이후 다른 최적화 항목에도 재사용 가능.

---

## 빌드 및 실행

```
엔진 버전 : Unreal Engine 5.5
필요 항목 : Visual Studio 2022 (MSVC 14.38, Windows 11 SDK 22621)
```

1. 레포를 클론합니다.
2. `TheSeventhbullet.uproject` 우클릭 → Generate Visual Studio project files
3. `TheSeventhbullet.sln`을 열고 빌드 타겟을 `Development Editor / Win64`로 설정 후 빌드

---

## 참고

- [원본 팀 저장소](https://github.com/NBcampUnrealTrack/7th-Team12-CH3-Project)
