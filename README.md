# 3D-RPG-Game
- Unity 기반의 3D 포트폴리오 프로젝트입니다.
- 플레이어 이동, 전투 시스템, 몬스터 AI, UI 등 기능을 구현하였습니다.
- [포트폴리오 다운로드 링크] https://drive.google.com/file/d/1Z_1VpIZxQoJhnxkAVnV7jrzf78L5xlYw/view?usp=drive_link
- [플레이 유튜브 링크] https://youtu.be/yGug4HMx4Ys
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
## ✅ GlobalValue 클래스 (주요 기능과 설명 포함)
- 로컬 저장 방식 사용
- 아이디,닉네임, 물약 수 등 로컬 저장
```
using System.Collections.Generic;
using UnityEngine;

public static class GlobalValue
{
    // ▶ 여러 계정을 저장하는 Dictionary
    // key: 계정 ID, value: (비밀번호, 닉네임, 골드, 다이아, 물약 수, 공격력, 최대체력, 스토리 본 여부)
    public static Dictionary<string, (string password, string nickname, int gold, int diaCount, int healPotionCount, float attackPower, float maxHp, bool storyShown)> userAccounts 
        = new Dictionary<string, (string, string, int, int, int, float, float, bool)>();

    // ▶ 현재 로그인된 사용자 ID 저장
    public static string currentUserId = "";

    // ================================
    // ▶ 게임 데이터 로드 (PlayerPrefs 기반)
    // 저장된 모든 계정 데이터를 불러와 userAccounts 딕셔너리에 세팅
    public static void LoadGameData()
    {
        int accountCount = PlayerPrefs.GetInt("AccountCount", 0);  // 저장된 계정 수 가져오기

        for (int i = 0; i < accountCount; i++)
        {
            // 각 계정의 저장된 데이터를 PlayerPrefs에서 읽어오기
            string id = PlayerPrefs.GetString("AccountId_" + i, "");
            string password = PlayerPrefs.GetString("AccountPassword_" + i, "");
            string nickname = PlayerPrefs.GetString("AccountNickName_" + i, "");
            int gold = PlayerPrefs.GetInt("AccountGold_" + i, 0);
            int diaCount = PlayerPrefs.GetInt("AccountDiaCount_" + i, 0);
            int healPotionCount = PlayerPrefs.GetInt("AccountHealPotionCount_" + i, 0);
            float attackPower = PlayerPrefs.GetFloat("AccountAttackPower_" + i, 20.0f);
            float maxHp = PlayerPrefs.GetFloat("AccountMaxHp_" + i, 200.0f);
            bool storyShown = PlayerPrefs.GetInt("AccountStoryShown_" + i, 0) == 1;

            // 유효한 ID일 때만 딕셔너리에 추가
            if (!string.IsNullOrEmpty(id))
            {
                userAccounts[id] = (password, nickname, gold, diaCount, healPotionCount, attackPower, maxHp, storyShown);
            }
        }
    }

    // ================================
    // ▶ 게임 데이터 저장 (PlayerPrefs 기반)
    // userAccounts 딕셔너리에 저장된 모든 계정 데이터를 PlayerPrefs에 저장
    public static void SaveGameData()
    {
        int accountIndex = 0;

        foreach (var account in userAccounts)
        {
            PlayerPrefs.SetString("AccountId_" + accountIndex, account.Key);
            PlayerPrefs.SetString("AccountPassword_" + accountIndex, account.Value.password);
            PlayerPrefs.SetString("AccountNickName_" + accountIndex, account.Value.nickname);
            PlayerPrefs.SetInt("AccountGold_" + accountIndex, account.Value.gold);
            PlayerPrefs.SetInt("AccountDiaCount_" + accountIndex, account.Value.diaCount);
            PlayerPrefs.SetInt("AccountHealPotionCount_" + accountIndex, account.Value.healPotionCount);
            PlayerPrefs.SetFloat("AccountAttackPower_" + accountIndex, account.Value.attackPower);
            PlayerPrefs.SetFloat("AccountMaxHp_" + accountIndex, account.Value.maxHp);
            PlayerPrefs.SetInt("AccountStoryShown_" + accountIndex, account.Value.storyShown ? 1 : 0);

            accountIndex++;
        }

        PlayerPrefs.SetInt("AccountCount", accountIndex); // 저장된 계정 수도 저장
        PlayerPrefs.Save(); // 저장 반영
    }

    // ================================
    // ▶ 로그인 처리
    // 입력한 id와 password가 일치하면 로그인 성공, currentUserId 세팅
    public static bool Login(string id, string password)
    {
        if (userAccounts.ContainsKey(id) && userAccounts[id].password == password)
        {
            currentUserId = id;
            return true;
        }
        return false;
    }

    // ================================
    // ▶ 계정 생성
    // 중복된 아이디가 없으면 새 계정을 만들고 초기값 세팅
    public static bool CreateAccount(string id, string password, string nickname)
    {
        if (userAccounts.ContainsKey(id))
            return false;  // 이미 존재하는 ID

        // 초기값: 골드, 다이아, 물약 수량 0, 공격력 10, 최대체력 200, 스토리 본 여부 false
        userAccounts[id] = (password, nickname, 0, 0, 0, 10.0f, 200.0f, false);

        SaveGameData();  // 저장
        return true;
    }

    // ================================
    // ▶ 골드 업데이트 (로그인한 계정 기준)
    public static void UpdateGold(int amount)
    {
        if (string.IsNullOrEmpty(currentUserId)) return;

        var account = userAccounts[currentUserId];
        account.gold += amount;
        if (account.gold < 0) account.gold = 0;

        userAccounts[currentUserId] = account;
        SaveGameData();
    }

    // ================================
    // ▶ 다이아 업데이트 (로그인한 계정 기준)
    public static void UpdateDiaCount(int amount)
    {
        if (string.IsNullOrEmpty(currentUserId)) return;

        var account = userAccounts[currentUserId];
        account.diaCount += amount;
        if (account.diaCount < 0) account.diaCount = 0;

        userAccounts[currentUserId] = account;
        SaveGameData();
    }

    // ================================
    // ▶ 물약 수량 업데이트 (로그인한 계정 기준)
    public static void UpdateHealPotionCount(int amount)
    {
        if (string.IsNullOrEmpty(currentUserId)) return;

        var account = userAccounts[currentUserId];
        account.healPotionCount += amount;
        if (account.healPotionCount < 0) account.healPotionCount = 0;

        userAccounts[currentUserId] = account;
        SaveGameData();
    }

    // ================================
    // ▶ 게임 데이터 초기화 (모든 저장 데이터 삭제)
    public static void ResetGameData()
    {
        PlayerPrefs.DeleteAll();
        currentUserId = "";
        userAccounts.Clear();
    }
}
```
# ✅캐릭터 데미지 처리 (TakeDamage 함수)
- 캐릭터가 데미지를 받을 때 호출되는 함수
- 체력이 0 이하일 경우 데미지를 무시함 (이미 죽은 상태)
- 공격자가 보스 몬스터면 데미지를 30으로 고정
- 체력을 차감하고 UI(체력바, 체력 텍스트)를 갱신
- 데미지 숫자 텍스트를 캐릭터 머리 위에 띄움
- 체력이 0 이하가 되면 사망 처리 실행

### 📌 핵심 코드

```csharp
public void TakeDamage(GameObject a_Attacker, float a_Damage = 10.0f)
{
    if (CurHp <= 0.0f)
        return;

    if (a_Attacker != null && a_Attacker.GetComponent<Monster_Ctrl>() != null)
    {
        Monster_Ctrl monsterCtrl = a_Attacker.GetComponent<Monster_Ctrl>();
        if (monsterCtrl.monType == MonType.Boss)
        {
            a_Damage = 30.0f;
        }
    }

    CurHp -= a_Damage;
    if (CurHp < 0.0f)
        CurHp = 0.0f;

    ImgHpbar.fillAmount = CurHp / MaxHp;
    Hpbar2.fillAmount = CurHp / MaxHp;

    UpdateHpText();

    SetAttackColor();

    Vector3 a_CacPos = this.transform.position;
    a_CacPos.y += 2.65f;
    GameMgr.Inst.SpawnDamageText((int)a_Damage, a_CacPos, 1);

    if (CurHp <= 0.0f)
    {
        Die();
    }
}
```
## 프로젝트 경험으로 느낀 점
- 프로젝트를 통해 게임 캐릭터 및 몬스터 AI 구현에 대해 경험
- 스킬 마다 쿨타임 적용 등 UI와 상호 작용 처리 등 완성도를 높이려 노력
## 아쉬운 점
- 스킬 이펙트가 단조롭고 스킬과 일반공격이 자연스럽지 못함
- 퀘스트 등 게임을 지속 플레이 하는 것이 부족
- 코드 중복 및 기능 분리 스크립팅 부족
