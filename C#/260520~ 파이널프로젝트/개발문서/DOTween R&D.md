# 🎮 DOTween 사용 가이드

## 목차
📚 목차
		0. DOTween이란?
		1. DOTween 초기화
		2. DOTween 기본 구조
		3. 자주 사용하는 DO API
		4. 콜백 함수 ON
		5. 설정 함수 SET	
		6. Sequence 사용법
		7. 루프 및 재사용
		8. 실무 팁
### 0. DOTween이란?
- Unity에서 가장 많이 사용하는 트윈(Tween) 애니메이션 라이브러리
- 코드를 통해 부드러운 애니메이션 제작 가능
- 성능이 매우 뛰어나며 사용법이 직관적
- UI / 캐릭터 / 카메라 / 이펙트 등 거의 모든 애니메이션에 사용 가능

-----
### 1. DOTween 초기화
```c++
using UnityEngine;
using System.Collections;
using DG.Tweening;

public class DotweenTest : MonoBehaviour
{
    DOTween.Init(
            recycleAllByDefault: true,  // 트윈 재사용 (GC 절감)
            useSafeMode: true,          // 에러 방지 모드
            logBehaviour: LogBehaviour.Default
        ).SetCapacity(200, 10);        // Tweener 200, Sequence 10 사전 할당
}    
```

💡 SetCapacity(intA, intB) 
- intA -> tweenerCapacity(트위너 용량) 동시에 재생될 수 있는 트위너의 최대 개수
- intB -> sequenceCapacity(시퀀스 용량) 동시에 재생될 수 있는 시퀀스의 최대 개수

✅ 실무 권장 설정
```c#
DOTween.Init(false, true, LogBehaviour.Verbose)
       .SetCapacity(200, 50);
```

-----

### 2. DOTween의 기본형

- DOTween은 기본적으로 Getter / Setter 람다 함수로 동작합니다.
- 람다함수로 원하는 동작을 수행시킬 수 있지만 DOTween 내장 함수로 해결할 수 있는 경우가 많다.
```c++
float myValue = 0f;
DOTween.To(
    () => myValue,          // getter: 시작할 때의 값
    v => myValue = v,       // setter: 변화되는 값을 실시간으로 대입
    endValue: 10f,          // 목표값
    duration: 2f            // 소요 시간
);
```
-----


### 3. DOTween API 활용하기

3.1   **DO**
```csharp
// ── 이동 ─────────────────────────────────────────────────
transform.DOMove(target, duration);
transform.DOLocalMoveY(2f, 0.5f);
transform.DOJump(targetPos, jumpPower: 3f, numJumps: 1, duration: 1f);
transform.DOPath(waypoints, duration, PathType.CatmullRom);  // 경로 이동

// ── 회전 ─────────────────────────────────────────────────
transform.DORotate(endValue, duration);
transform.DORotateQuaternion(targetRotation, duration);
transform.DOLookAt(targetPosition, duration);

// ── 흔들기 ───────────────────────────────────────────────
transform.DOShakePosition(duration: 0.5f, strength: 0.5f, vibrato: 10, randomness: 90);
transform.DOShakeRotation(0.5f, 30f);
transform.DOShakeScale(0.5f, 0.3f);

// ── 색상 ─────────────────────────────────────────────────
GetComponent<Renderer>().material.DOColor(Color.red, 1f);
GetComponent<Light>().DOColor(Color.yellow, 1f);
GetComponent<Light>().DOIntensity(2f, 0.5f);

// ── Rigidbody ────────────────────────────────────────────
GetComponent<Rigidbody>().DOMove(target, duration);
```
    
3.2   **ON**
- On 키워드는 콜백함수로써 각 함수가 원하는 시간대에 람다식이나 함수를 실행 할 수 있도록 한다.
```c++
// 1. 끝났을 때 코루틴이나 함수를 호출하는 경우
transform.DOScale(1.0f, 1.0f)
        .OnComplete(() => StartCoroutine(WaitAndMove()));

```
```c++
// 람다식을 호출한 경우
// 2. 끝났을 때 연속적인 다른 트윈을 실행하는 경우
transform.DOScale(1.0f, 1.0f)
        .OnComplete(() => transform.DOMove(new Vector3(0, 5f, 0), 2.0f));

```
#### 📌 주요 콜백 정리
| 함수         | 설명         |
| ---------- | ---------- |
| OnStart    | 최초 시작 시 1회 |
| OnUpdate   | 매 프레임      |
| OnComplete | 종료 시       |
| OnKill     | 트윈 제거 시    |





3.3   **SET**
- 트윈의 성격을 결정하는 설정 함수들입니다. 다음은 대표적인 함수들입니다.
```c++
// 1. SetEase
// 트윈의 완급조절(가속도 곡선)을 정합니다. (Linear: 등속, OutQuad: 서서히 감속 등)
transform.DOScale(1.5f, 1.0f).SetEase(Ease.InBounce); // 통통 튀는 효과 적용

──────────────────────────────────

// 2. SetLink : 메모리 및 버그 방지
// 트윈이 실행 중일 때 해당 GameObject가 Destroy되면 MissingReferenceException(Null 에러)이 터집니다.
// SetLink를 붙여두면 지정된 오브젝트가 파괴될 때 트윈도 알아서 안전하게 파괴(Kill)됩니다.
transform.DOMoveX(5f, 2.0f).SetLink(gameObject);

──────────────────────────────────

// 3. SetUpdate
// Time.timeScale = 0f으로 게임을 일시정지(Pause)시켜도, 포즈 창 UI 등은 정상 작동해야 합니다.
// true를 넣어주면 유니티의 타임스케일을 무시하고 실제 현실 시간(RealTime) 기준으로 작동합니다.
transform.DOScale(1.0f, 0.5f).SetUpdate(true);

──────────────────────────────────

// 4. SetRelative : 기준점 상대값 변환
// 기본적으로 DOTween은 '절대 좌표'로 이동합니다. 
// 하지만 이 옵션을 켜면 "현재 오브젝트가 있는 위치를 기준(0,0,0)으로 +5만큼 더 이동"하는 '상대 좌표'로 작동합니다.
transform.DOMoveX(5f, 2.0f).SetRelative();

──────────────────────────────────

// 5. SetId : ID 부여를 통한 일괄 제어
// 생성되는 트윈에 고유한 이름(문자열, int, 오브젝트 등)의 ID를 부여합니다.
// 나중에 특정 그룹의 트윈들만 골라서 일시정지하거나 삭제할 때(`DOTween.Kill("MyUI");`) 매우 유용합니다.
transform.DOFade(0f, 1.0f).SetId("MyUI");

──────────────────────────────

// 6. SetAutoKill :  재생 후 자동 파괴 방지
// DOTween은 기본적으로 재생이 끝나면 메모리 관리를 위해 트윈을 자동 파괴합니다.
// UI 창처럼 켰다 껐다 하며 트윈을 '재사용'하고 싶다면 이 값을 false로 꺼두어야 합니다.
mySequence.SetAutoKill(false);
```
-----

### 4. DOTween - 🎬**'sequence'**

📌 핵심 함수

| 함수      | 설명               |
| ------- | ---------------- |
| Append  | 마지막 뒤에 추가        |
| Join    | 이전 Tween과 동시에 실행 |
| Insert  | 특정 시간에 실행        |
| Prepend | 맨 앞에 추가          |


```c++

public class DotweenTest : MonoBehaviour
{
    Sequence sequence;

    private void Start()
    {
        sequence = DOTween.Sequence();
 
        // 방식 1: 줄바꿈 독립 실행형 (런타임 동적 조립이나 조건문 분기에 유리)
        sequence.Append(transform.DOMove(new Vector3(0f, 5f, 0f), 2.0f));
        sequence.Join(transform.DORotate(new Vector3(0f, -180f, 0f), 2.0f));
        sequence.Append(transform.DORotate(new Vector3(0f, 360f, 0f), 2.0f));
        sequence.Insert(4.0f, transform.DOScale(new Vector3(1.5f, 1.5f, 1.5f), 1.0f));
        sequence.Prepend(transform.DOScale(new Vector3(0.5f, 0.5f, 0.5f), 2.0f));
        
        // 방식 2: 메서드 체이닝 구조 (가독성이 좋고 한눈에 흐름을 파악하기 유리)
        sequence.Prepend(transform.DOScale(new Vector3(0.5f, 0.5f, 0.5f), 2.0f))
                .Append(transform.DOMove(new Vector3(0f, 5f, 0f), 2.0f))
                .Join(transform.DORotate(new Vector3(0f, -180f, 0f), 2.0f))
                .Append(transform.DORotate(new Vector3(0f, 360f, 0f), 2.0f))
                .Insert(4.0f, transform.DOScale(new Vector3(1.5f, 1.5f, 1.5f), 1.0f));
    }
}       

```

#### 4.1 Append + Join (동시 연출)

    기본적으로 자주 쓰는 방식, 연속 연출을 만들 때 유용하다.

```c++
Sequence appendSeq = DOTween.Sequence();

1. [0초~1초] 앞으로 슥 돌진합니다.
appendSeq.Append(transform.DOMoveZ(5f, 1.0f));
 
2. 직전의 'DOMoveZ'와 동시에 시작해서 360도 회전합니다. (0초에 같이 시작)
appendSeq.Join(transform.DORotate(new Vector3(0, 360, 0), 1.0f));
 
3. 앞의 이동+회전 무리가 '1초'에 완전히 끝나면, 이어서 위로 튀어 오릅니다.
appendSeq.Append(transform.DOMoveY(3f, 1.0f));
```

#### 4.2 Insert + Join (특정 타이밍의 세트 연출)

    전체의 흐름과 상관없이 정확히 몇초 뒤에 효과를 나타나게 할 때

```c++
Sequence insertSeq = DOTween.Sequence();

1. [0초~5초] 배경이나 발판이 아주 느리게 오른쪽으로 이동합니다.
insertSeq.Append(transform.DOMoveX(10f, 5.0f));

2. 배경 이동 도중, 정확히 '3.0초'가 되는 순간에 함정 발동 오브젝트의 색상을 빨간색으로 바꿉니다. (3초~4초)
insertSeq.Insert(3.0f, myRenderer.material.DOColor(Color.red, 1.0f));

3. 직전의 'DOColor'와 동시에 시작해서 오브젝트를 덜덜 떨게 만듭니다. (3초에 같이 시작)
insertSeq.Join(transform.DOShakePosition(1.0f, 0.5f));
```

-----
### 5. 참고하면 좋을 기능

5.1 **무한 루프 및 루프 설정 - SetLoops**
- -1을 넣으면 무한 루프로 돌릴 수 있다. 다른 자연수를 넣으면 그 숫자만큼 반복한다.

| 타입          | 설명       |
| ----------- | -------- |
| Restart     | 처음부터 반복  |
| Yoyo        | 왕복 반복    |
| Incremental | 누적 증가 반복 |

```c++
transform.DORotate(new Vector3(0, 180, 0), 2.0f)
    .SetLoops(-1, LoopType.Yoyo);
```

5.2 **게임 정지 또는 일시정지 무시하기 - SetUpdate**

- Time.timeScale이 0이 되어 게임이 멈춰도, 이 UI 애니메이션은 정상 작동합니다.
```c++
transform.DOScale(1.0f, 0.5f)
           .SetUpdate(true); // true = 유니티 타임스케일 무시
```

5.3 **시퀀스 재사용하기**

- DOTween의 모든 트윈과 시퀀스는 한번 재생이 끝나면 자동 파괴된다. 창을 열고 닫을 때처럼 하나의 시퀀스를 켜고 끄며 재사용하고 싶다면 옵션을 꺼야된다.
```c++
     private void Awake()
 {
     uiSequence = DOTween.Sequence();
     uiSequence.Append(transform.DOScale(1.0f, 0.5f));
 
     // ⭐ 재생이 끝나도 시퀀스가 자동으로 파괴되지 않게 설정!
     uiSequence.SetAutoKill(false).Pause();
 }
 
 public void OpenUI()
 {
     uiSequence.PlayForward(); // 정방향 재생 (열기)
 }
 
 public void CloseUI()
 {
     uiSequence.PlayBackwards(); // 역방향 재생 (닫기)
 }
```