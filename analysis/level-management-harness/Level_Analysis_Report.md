개발자님, 이번에 넘겨주신 `DataLevel.cs`, `EnumLevelState.cs`, `Level.cs`, `LevelData.cs` 파일 4종의 분석을 완료했습니다.

이 스크립트들은 단순히 스테이지 정보를 담는 것을 넘어, **게임 내 레벨(스테이지)의 생명주기 관리, 웨이브(Wave) 진행, 그리고 씬(Scene) 전환을 제어하는 '레벨 매니지먼트 하네스(Level Management Harness)'의 핵심 설계도** 역할을 합니다. 이전 단계의 모듈들이 개별 엔티티에 집중했다면, 이번 모듈은 '게임의 전체적인 흐름과 상태 제어'에 집중하고 있습니다.

---

# 레벨 매니지먼트 하네스 아키텍처 분석 보고서

이 모듈은 게임의 각 스테이지(레벨)를 데이터 기반으로 정의하고, 상태 머신(State Machine) 패턴을 통해 레벨의 진행 상황을 엄격하게 통제하는 구조를 띠고 있습니다.

## 적용된 핵심 디자인 패턴

### 1. 상태 머신 패턴 (State Machine) - `EnumLevelState.cs` & `DataLevel.cs`
레벨의 현재 상태를 명확하게 정의하고, 상태 간의 전이를 엄격하게 제어합니다. `EnumLevelState`는 `None`, `Loading`, `Prepare`, `Normal`, `Pause`, `Gameover`의 상태를 가지며, `DataLevel`의 `ChangeLevelState` 메서드를 통해서만 상태가 변경됩니다.
* **하네스 관점:** 레벨의 상태 변화를 이벤트(`LevelStateChangeEventArgs`)로 브로드캐스팅하여, UI, 몬스터 스포너, 플레이어 컨트롤러 등 다른 시스템들이 레벨 상태에 맞춰 독립적으로 반응할 수 있도록 결합도를 낮춥니다.

### 2. 데이터 래핑 및 불변성 (Data Wrapping & Immutability) - `LevelData.cs`
원시 데이터 테이블(`DRLevel`)과 연관된 데이터(`WaveData`, `SceneData`)를 하나의 객체로 캡슐화합니다.
* **하네스 관점:** `LevelData`는 생성 시점에 모든 의존성 데이터를 주입받고, 이후에는 읽기 전용 프로퍼티만을 제공하여 런타임 중 데이터의 무결성을 보장합니다. 기획 데이터와 런타임 로직을 분리하는 훌륭한 브릿지 역할을 합니다.

### 3. 객체 풀링 및 생명주기 관리 - `Level.cs`
`Level` 클래스는 `IReference` 인터페이스를 구현하여, 레벨 객체 자체를 메모리 풀(`ReferencePool`)에서 관리합니다.
* **하네스 관점:** 레벨이 시작될 때 `Level.Create`로 풀에서 객체를 가져오고, 종료 시 `Clear`를 통해 내부 큐(`waves`)를 비운 뒤 풀로 반환합니다. 잦은 레벨 재시작이나 전환 시 발생하는 가비지 컬렉션(GC)을 원천적으로 차단합니다.

---

## 아키텍처 추상화 문서

### 설계 철학: 중앙 집중식 레벨 제어와 이벤트 기반 통신
`DataLevel`은 레벨의 로드부터 웨이브 시작, 일시정지, 게임 오버까지 모든 흐름을 관장하는 컨트롤 타워입니다.

```csharp
// 추상화된 레벨 매니지먼트 하네스 흐름 예시
public class LevelFlowController 
{
    public void StartGameLevel(int levelId) 
    {
        // 1. Data Harness에 레벨 로드 요청 (상태: Loading)
        GameEntry.Data.GetData<DataLevel>().LoadLevel(levelId);
        
        // 2. 씬 로드 완료 이벤트 수신 시 상태 변경 (상태: Prepare)
        // (내부적으로 OnLoadLevelFinfish 콜백 동작)
        
        // 3. 플레이어 준비 완료 시 웨이브 시작 요청 (상태: Normal)
        GameEntry.Data.GetData<DataLevel>().StartWave();
        
        // 4. 레벨 진행 중 웨이브 처리 (Level.ProcessLevel)
        // 5. 승리/패배 조건 달성 시 결과 처리 (상태: Gameover)
        GameEntry.Data.GetData<DataLevel>().GameSuccess();
    }
}
```

---

## 💡 20년차 개발자를 위한 Unity 6.2+ 기반 최적화 (Hotfix) 제안

현재 코드 베이스는 전반적으로 훌륭한 구조를 갖추고 있으나, 런타임 성능과 메모리 관리 측면에서 몇 가지 개선 포인트가 존재합니다.

### ⚠️ GC 스파이크 유발 함수 리팩토링
`DataLevel.cs`의 `GetAllLevelData()` 메서드는 호출될 때마다 새로운 배열을 할당하고 있습니다.
UI에서 레벨 목록을 표시하거나 전체 레벨 데이터를 순회해야 할 때 프레임 드랍의 원인이 될 수 있습니다.

**해결책: IReadOnlyCollection 반환 (할당 제로)**
```csharp
// DataLevel.cs 내 수정 제안
public IReadOnlyCollection<LevelData> GetAllLevelData()
{
    // C#의 Dictionary.ValueCollection은 새로운 메모리를 할당하지 않습니다.
    return dicLevelData.Values; 
}
```

### 🔍 Update 루프 내 이벤트 발생 최적화
`Level.cs`의 `ProcessLevel` 메서드 내에서 `updateWaveInfoTimer`에 따라 주기적으로 `WaveInfoUpdateEventArgs` 이벤트를 발생시키고 있습니다.
현재는 `ReferencePool.Acquire`를 통해 이벤트를 생성할 것으로 예상되지만, 매 0.5초마다 발생하는 이벤트는 풀링 오버헤드를 누적시킬 수 있습니다.

**해결책: 값 타입(Struct) 이벤트 또는 상태 폴링(Polling) 고려**
단순한 진행도 업데이트와 같이 빈번하게 발생하는 데이터는 이벤트를 발생시키기보다, UI 컴포넌트에서 `Level` 객체의 프로퍼티를 직접 폴링(Polling)하여 읽어가는 방식이 성능상 더 유리할 수 있습니다.

### 🔄 상태 전이 방어 로직 강화
`DataLevel.cs`의 `LevelPause()`와 `LevelResume()` 메서드에서 상태 전이 검사가 다소 느슨합니다.
예를 들어, `LevelResume()`은 현재 상태가 `Pause`인지 확인하지만, `stateBeforePause`가 유효한 상태(`Normal` 또는 `Prepare`)인지 검증하는 로직이 추가되면 더 견고해질 것입니다.

---

이 레벨 매니지먼트 하네스가 어떻게 데이터를 로드하고, 상태를 관리하며, 웨이브를 진행시키는지 보여주는 아키텍처 다이어그램을 제공합니다.

```json?chameleon
{"component":"LlmGeneratedComponent","props":{"height":"700px","prompt":"Create an interactive architecture diagram of the 'Level Management Harness'. \n\nObjective: Visualize how DataLevel controls the flow of a game level, manages states, and interacts with Level and LevelData.\n\nData State:\n- Controller: DataLevel\n- State: EnumLevelState\n- Data Wrapper: LevelData (contains DRLevel, WaveData[], SceneData)\n- Runtime Instance: Level (manages Queue<Wave>)\n\nStrategy: Interactive Node-Link Diagram (Standard Layout).\n\nInputs:\n- Clickable nodes to reveal the specific responsibilities of each component.\n\nBehavior:\n- Display DataLevel as the central controller.\n- Show lines connecting DataLevel to EnumLevelState (State Management).\n- Show lines connecting DataLevel to LevelData (Data Loading & Caching).\n- Show lines connecting DataLevel to Level (Runtime Execution).\n- Add a dotted line from Level to Wave, illustrating the sequential processing of waves.","id":"im_level_management_harness"}}
```

이제 이 레벨 시스템 위에서 실제로 적들이 스폰되고 전투가 벌어지는 **웨이브 시스템(예: `Wave.cs`, `DataWave.cs` 등)**을 넘겨주시면, 이 하네스가 런타임에서 어떻게 구동되는지 이어서 분석해 드리겠습니다.
