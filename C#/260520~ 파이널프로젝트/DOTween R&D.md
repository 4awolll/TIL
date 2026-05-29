# DOTween 사용 가이드

## 목차
0. DOTween이란?
    - DOTween은 유니티 게임엔진에서 가장 널리 쓰이는 코드 기반의 애니메이션 제작 라이브러리이다. 

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

    // 디폴트로 이걸로 쓰면 됩니다.
    DOTween.Init(false, true, LogBehaviour.Verbose).SetCapacity(200, 50);

}    
```
### 2. DOTween의 기본형

- DOTween은 기본적으로 람다함수로 동작한다.
- 람다함수로 원하는 동작을 수행시킬 수 있지만 DOTween 내장 함수로 해결할 수 있는 경우가 많다.
```c++
float myValue = 0f;
        DOTween.To(
            () => myValue,
            v => myValue = v,
            endValue: 10f,
            duration: 2f
        );
```


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
- On 키워드는 콜백함수로써 각 함수에 따라 원하는 시간대에 람다식이나 함수를 실핼할 수 있다.
```c++
// 함수를 호출하는 경우
transform.DOScale(1.0f, 1.0f)
        .SetDelay(1.0f)
        .OnComplete(() => StartCoroutine(WaitAndMove()));

```
```c++
// 람다식을 호출한 경우
transform.DOScale(1.0f, 1.0f)
    .SetDelay(1.0f)
    .SetDelay(2.0f)
    .OnComplete(() => transform.DOMove(new Vector3(0, 5f, 0), 2.0f));

```

3.3   **SET**
- 트윈의 성격을 결정하는 설정 함수들입니다. SetEase, SetLoops 등이 대표적입니다.
```c++
transform.DOScale(1.5f, 1.0f).SetEase(Ease.InBounce); // 통통 튀는 효과 적용
```

### 4. DOTween - sequence

.Prepend : 맨 처음에 추가

.Append : 트윈 마지막에 추가, 앞의 Append가 끝나고 나서 실행됨

.Insert : 일정 시간에 시작, 오직 자기가 지정받은 그 시간이 되면 칼같이 실행

.Join : 앞에 추가된 트윈과 동시 시작, Append, Insert에 붙을수있음

```c++

public class DotweenTest : MonoBehaviour
{
    Sequence sequence;

    private void Start()
    {
        sequence = DOTween.Sequence();
 
        // 방식 1
        sequence.Append(transform.DOMove(new Vector3(0f, 5f, 0f), 2.0f));
        sequence.Join(transform.DORotate(new Vector3(0f, -180f, 0f), 2.0f));
        sequence.Append(transform.DORotate(new Vector3(0f, 360f, 0f), 2.0f));
        sequence.Insert(4.0f, transform.DOScale(new Vector3(1.5f, 1.5f, 1.5f), 1.0f));
        sequence.Prepend(transform.DOScale(new Vector3(0.5f, 0.5f, 0.5f), 2.0f));
        
        // 방식 2
        sequence.Prepend(transform.DOScale(new Vector3(0.5f, 0.5f, 0.5f), 2.0f))
                .Append(transform.DOMove(new Vector3(0f, 5f, 0f), 2.0f))
                .Join(transform.DORotate(new Vector3(0f, -180f, 0f), 2.0f))
                .Append(transform.DORotate(new Vector3(0f, 360f, 0f), 2.0f))
                .Insert(4.0f, transform.DOScale(new Vector3(1.5f, 1.5f, 1.5f), 1.0f));
    }
}       

```

1. Append에 Join이 붙는 경우 (동시 연출)

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

2. Insert에 Join이 붙는 경우(특정 타이밍의 세트 연출)

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
### 5. 참고하면 좋을 기능

5.1 **무한 루프 및 루프 설정 - SetLoops**


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

5.1 **시퀀스 재사용하기**

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
     uiSequence.PlayBackwards(); // 역방향 재생 (닫기 - 갓기능!)
 }
```