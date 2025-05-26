# 3D-RPG-Game
Unity 기반의 3D 포트폴리오 프로젝트입니다.
플레이어 이동, 전투 시스템, 몬스터 AI, UI 등 기능을 구현하였습니다.
[포트폴리오 다운로드 링크]
https://drive.google.com/file/d/1J7B_zu6HMrxjvlOZYeY1fT6kVRzrr28Q/view?usp=drive_link
# 프로젝트 개요
| 항목 | 내용 |
|------|------|
| 프로젝트명 | 3D-RPG-valkyrie |
| 장르 | 3D 액션 RPG |
| 개발 기간 | 2025.03 ~ 2025.04 |
| 사용 엔진 | Unity 2022.3 LTS |
| 개발 인원 | 1인 개발 |
| 주요 역할 | 전체 시스템 기획 및 개발 , UI(Asset Store 제공된 UI 활용) 등 |
## 🛠 사용 기술
- Unity (3D 환경)
- C# 스크립팅
- Animator, NavMesh, Rigidbody, Raycast 등 활용
- UI 시스템 (Canvas, Image, Text, Button)
## 🔧 주요 구현 기능

### ✅ 캐릭터 컨트롤
- 마우스 클릭 기반 이동 구현
- **우클릭**: 바닥 클릭 시 이동 / 몬스터 클릭 시 추적
- **좌클릭**: 적 타겟 지정
- 스킬 사용 중 이동 불가
- UI 위 클릭 시 이동 방지 처리

```csharp
void MousePickCheck() {
    if (Input.GetMouseButtonDown(1)) { // 우클릭 - 이동 또는 추적
        if (!GameMgr.IsPointerOverUIObject() && !IsSkill()) {
            Ray ray = Camera.main.ScreenPointToRay(Input.mousePosition);
            if (Physics.Raycast(ray, out hitInfo, Mathf.Infinity, LayerMask.value)) {
                if (hitInfo.collider.gameObject.layer == LayerMask.NameToLayer("Enemy")) {
                    MousePicking(hitInfo.point, hitInfo.collider.gameObject); // 몬스터 추적
                    if (GameMgr.Inst.m_MsClickMark != null)
                        GameMgr.Inst.m_MsClickMark.SetActive(false);
                } else {
                    MousePicking(hitInfo.point); // 이동
                    GameMgr.Inst.MsClickMarkOn(hitInfo.point);
                }
            }
        }
    } else if (Input.GetMouseButtonDown(0)) { // 좌클릭 - 타겟 지정
        if (!GameMgr.IsPointerOverUIObject()) {
            Ray ray = Camera.main.ScreenPointToRay(Input.mousePosition);
            if (Physics.Raycast(ray, out hitInfo, Mathf.Infinity, LayerMask.value)) {
                if (hitInfo.collider.gameObject.layer == LayerMask.NameToLayer("Enemy")) {
                    MousePicking(hitInfo.point, hitInfo.collider.gameObject);
                    if (GameMgr.Inst.m_MsClickMark != null)
                        GameMgr.Inst.m_MsClickMark.SetActive(false);
                }
            }
        }
    }
}

### ✅ 몬스터 AI

- 플레이어 감지, 추적, 공격 상태를 전환하며 동작하는 **FSM 기반 AI**
- `idle → trace → attack` 상태 흐름에 따라 적절한 애니메이션과 이동 적용
- 공격 거리, 추적 거리 계산은 `Vector3.Distance` 대신 벡터 연산으로 직접 처리

#### ✅ 상태 업데이트 (`MonStateUpdate`)

```csharp
void MonStateUpdate()
{
    if (m_AggroTarget == null) {
        GameObject[] a_Players = GameObject.FindGameObjectsWithTag("Player");
        foreach (GameObject player in a_Players) {
            m_CacVLen = player.transform.position - transform.position;
            m_CacVLen.y = 0.0f;
            m_MoveDir = m_CacVLen.normalized;
            m_CacDist = m_CacVLen.magnitude;

            if (m_CacDist <= m_AttackDist) {
                AI_State = AnimState.attack;
                m_AggroTarget = player;
                break;
            } else if (m_CacDist <= m_TraceDist) {
                AI_State = AnimState.trace;
                m_AggroTarget = player;
                break;
            }
        }

        if (m_AggroTarget == null) {
            AI_State = AnimState.idle;
            m_AggroTgId = -1;
        }
    }
    else {
        m_CacVLen = m_AggroTarget.transform.position - transform.position;
        m_CacVLen.y = 0.0f;
        m_MoveDir = m_CacVLen.normalized;
        m_CacDist = m_CacVLen.magnitude;

        if (m_CacDist <= m_AttackDist)
            AI_State = AnimState.attack;
        else if (m_CacDist <= m_TraceDist)
            AI_State = AnimState.trace;
        else {
            AI_State = AnimState.idle;
            m_AggroTarget = null;
            m_AggroTgId = -1;
        }
    }
}

### ✅ 행동 처리 (MonActionUpdate)
```csharp
void MonActionUpdate()
{
    if (m_AggroTarget == null) {
        ChangeAnimState(AnimState.idle, 0.12f);
        return;
    }

    if (AI_State == AnimState.attack) {
        RotateToTarget();
        ChangeAnimState(AnimState.attack, 0.12f);
    }
    else if (AI_State == AnimState.trace) {
        RotateToTarget();

        if (IsAttackAnim()) return;

        m_MoveNextStep = m_MoveDir * (m_MoveVelocity * Time.deltaTime);
        m_MoveNextStep.y = 0.0f;
        transform.position += m_MoveNextStep;

        ChangeAnimState(AnimState.trace, 0.12f);
    }
    else if (AI_State == AnimState.idle) {
        ChangeAnimState(AnimState.idle, 0.12f);
    }
}

void RotateToTarget() {
    if (m_MoveDir.magnitude > 0.0001f) {
        m_TargetRot = Quaternion.LookRotation(m_MoveDir);
        transform.rotation = Quaternion.Slerp(transform.rotation, m_TargetRot, Time.deltaTime * m_RotSpeed);
    }
}
