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
void MousePickCheck()  // 마우스 클릭 감지를 위한 함수
{
    if (Input.GetMouseButtonDown(1) == true)  // 오른쪽 마우스 버튼 클릭시 (이동)
    {
        if (GameMgr.IsPointerOverUIObject() == false) // UI가 아닌 곳을 클릭했을 때만 피킹 이동 허용
        {
            if (IsSkill() == true)  // 스킬 발동 중일 때는 마우스 클릭 이동 무시
                return;

            m_MousePos = Camera.main.ScreenPointToRay(Input.mousePosition);

            if (Physics.Raycast(m_MousePos, out hitInfo, Mathf.Infinity, LayerMask.value))
            {
                if (hitInfo.collider.gameObject.layer == LayerMask.NameToLayer("Enemy"))
                {  // 마우스로 몬스터를 피킹 했다면 (왼쪽 클릭은 몬스터 선택, 오른쪽 클릭은 이동)
                   // 몬스터 클릭은 이동을 무시하도록 따로 처리할 필요가 없음
                    MousePicking(hitInfo.point, hitInfo.collider.gameObject);

                    if (GameMgr.Inst.m_MsClickMark != null)
                        GameMgr.Inst.m_MsClickMark.SetActive(false);
                }
                else // 지형 바닥 피킹일 때 (이동)
                {
                    MousePicking(hitInfo.point);
                    GameMgr.Inst.MsClickMarkOn(hitInfo.point);
                }
            }
        }
    }
    else if (Input.GetMouseButtonDown(0) == true)  // 왼쪽 마우스 버튼 클릭시 (몬스터 선택)
    {
        if (GameMgr.IsPointerOverUIObject() == false) // UI가 아닌 곳을 클릭했을 때만 피킹 허용
        {
            m_MousePos = Camera.main.ScreenPointToRay(Input.mousePosition);

            if (Physics.Raycast(m_MousePos, out hitInfo, Mathf.Infinity, LayerMask.value))
            {
                if (hitInfo.collider.gameObject.layer == LayerMask.NameToLayer("Enemy"))
                {  // 몬스터 클릭
                    MousePicking(hitInfo.point, hitInfo.collider.gameObject);

                    if (GameMgr.Inst.m_MsClickMark != null)
                        GameMgr.Inst.m_MsClickMark.SetActive(false);
                }
            }
        }
    }
}  // void MousePickCheck()
