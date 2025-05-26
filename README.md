# 3D-RPG-Game
- Unity 기반의 3D 포트폴리오 프로젝트입니다.
- 플레이어 이동, 전투 시스템, 몬스터 AI, UI 등 기능을 구현하였습니다.
- [포트폴리오 다운로드 링크] https://drive.google.com/file/d/1J7B_zu6HMrxjvlOZYeY1fT6kVRzrr28Q/view?usp=drive_link
- 플레이 유튜브 업로드 예정
# 프로젝트 개요
| 항목 | 내용 |
|------|------|
| 프로젝트명 | Valkyrie |
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

## ✅ 캐릭터 컨트롤
- 마우스 클릭 기반 이동 구현  
- **우클릭**: 바닥 클릭 시 이동 / 몬스터 클릭 시 추적  
- **좌클릭**: 적 타겟 지정  
- 스킬 사용 중 이동 불가  
- UI 위 클릭 시 이동 방지 처리  

### 📌 핵심 코드 (MousePickCheck)
```
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
```
## ✅ 몬스터 AI
- 플레이어 감지 → 추적 → 공격
- 공격/추적 거리 계산은 벡터 연산으로 수행
- 상태별 애니메이션 및 이동 적용

### 📌 상태 업데이트 (MonStateUpdate)
```
void MonStateUpdate() {
    if (m_AggroTarget == null) { // 타겟이 없으면 주변 플레이어 탐색
        SearchPlayer();
    } else { // 타겟이 있으면 상태 갱신
        UpdateAggroTargetState();
    }
}
void SearchPlayer() {
    GameObject[] a_Players = GameObject.FindGameObjectsWithTag("Player"); // 모든 플레이어 탐색
    foreach (GameObject player in a_Players) {
        Vector3 direction = player.transform.position - transform.position; // 플레이어까지의 방향 벡터
        direction.y = 0.0f; // 수직 거리 무시 (XZ 평면 기준)

        float distance = direction.magnitude; // 거리 계산

        if (distance <= m_AttackDist) { // 공격 거리 안이면
            SetAggro(player, AnimState.attack); // 공격 상태로 전이
            break;
        } else if (distance <= m_TraceDist) { // 추적 거리 안이면
            SetAggro(player, AnimState.trace); // 추적 상태로 전이
            break;
        }
    }

    if (m_AggroTarget == null) { // 탐색 실패 시
        AI_State = AnimState.idle; // 대기 상태로 전환
        m_AggroTgId = -1; // 타겟 ID 초기화
    }
}
void UpdateAggroTargetState() {
    Vector3 direction = m_AggroTarget.transform.position - transform.position; // 현재 타겟 방향 계산
    direction.y = 0.0f;

    float distance = direction.magnitude; // 타겟까지 거리 계산

    m_MoveDir = direction.normalized; // 정규화 방향 저장
    m_CacDist = distance; // 거리 저장

    if (distance <= m_AttackDist) {
        AI_State = AnimState.attack; // 공격 상태 전환
    } else if (distance <= m_TraceDist) {
        AI_State = AnimState.trace; // 추적 상태 유지
    } else {
        m_AggroTarget = null; // 범위 밖이면 타겟 해제
        m_AggroTgId = -1;
        AI_State = AnimState.idle; // 대기 상태로 복귀
    }
}
void SetAggro(GameObject target, AnimState state) {
    m_AggroTarget = target; // 타겟 지정
    m_CacVLen = target.transform.position - transform.position;
    m_CacVLen.y = 0.0f;
    m_MoveDir = m_CacVLen.normalized; // 이동 방향 계산
    m_CacDist = m_CacVLen.magnitude; // 거리 계산
    AI_State = state; // 상태 설정
}
```
### 📌 행동 처리 (MonActionUpdate)
```
void MonActionUpdate() {
    if (m_AggroTarget == null) {
        ChangeAnimState(AnimState.idle, 0.12f); // 타겟이 없으면 대기 애니메이션 전환
        return;
    }
    switch (AI_State) {
        case AnimState.attack:
            DoAttack(); // 공격 상태
            break;
        case AnimState.trace:
            DoTrace(); // 추적 상태
            break;
        case AnimState.idle:
            ChangeAnimState(AnimState.idle, 0.12f); // 대기 상태
            break;
    }
}
void DoAttack() {
    RotateToTarget(); // 타겟을 향해 회전
    ChangeAnimState(AnimState.attack, 0.12f); // 공격 애니메이션 실행
}
void DoTrace() {
    RotateToTarget(); // 타겟을 향해 회전

    if (IsAttackAnim()) return; // 공격 애니메이션 중엔 이동 금지

    m_MoveNextStep = m_MoveDir * (m_MoveVelocity * Time.deltaTime); // 다음 이동 거리 계산
    m_MoveNextStep.y = 0.0f;
    transform.position += m_MoveNextStep; // 몬스터 이동

    ChangeAnimState(AnimState.trace, 0.12f); // 추적 애니메이션 실행
}
```
