# UniTask — 사용법 가이드

> Unity 전용 고성능 async/await 라이브러리 | `com.cysharp.unitask` | Unity 2018.4+

---

## 목차

1. [개요](#1-개요)
2. [설치](#2-설치)
3. [기본 사용법](#3-기본-사용법)
4. [UniTask 생성 패턴](#4-unitask-생성-패턴)
5. [취소 (CancellationToken)](#5-취소-cancellationtoken)
6. [병렬 / 순차 실행](#6-병렬--순차-실행)
7. [Unity 이벤트 대기](#7-unity-이벤트-대기)
8. [PlayerLoop 타이밍](#8-playerloop-타이밍)
9. [예외 처리](#9-예외-처리)
10. [주의사항 및 트러블슈팅](#10-주의사항-및-트러블슈팅)

---

## 1. 개요

`UniTask`는 Unity에서 C# `async/await`를 **GC 없이** 사용할 수 있도록 설계된 라이브러리다.  
기존 `Task`/`ValueTask` 대비 Unity PlayerLoop와 긴밀하게 통합되어, 프레임 단위 대기·코루틴 대체·취소 처리가 자연스럽다.

| 기존 방식 | UniTask |
|---|---|
| `IEnumerator` 코루틴 | `async UniTask` |
| `Task.Delay(1000)` (Thread 기반) | `UniTask.Delay(1000)` (PlayerLoop 기반) |
| `yield return null` | `await UniTask.Yield()` |
| `yield return new WaitForSeconds(t)` | `await UniTask.WaitForSeconds(t)` |

---

## 2. 설치

### UPM (권장)

`Packages/manifest.json`에 추가:

```json
{
  "dependencies": {
    "com.cysharp.unitask": "https://github.com/Cysharp/UniTask.git?path=src/UniTask/Assets/Plugins/UniTask"
  }
}
```

### OpenUPM

```bash
openupm add com.cysharp.unitask
```

---

## 3. 기본 사용법

```csharp
using Cysharp.Threading.Tasks;
using UnityEngine;

public class BasicExample : MonoBehaviour
{
    private void Start()
    {
        // Fire-and-forget: 예외 처리를 위해 Forget() 호출
        LoadDataAsync().Forget();
    }

    private async UniTaskVoid LoadDataAsync()
    {
        Debug.Log("시작");
        await UniTask.Delay(2000); // 2초 대기 (GC 없음)
        Debug.Log("2초 후");
    }

    // 반환값이 있는 경우
    private async UniTask<int> FetchScoreAsync()
    {
        await UniTask.Delay(1000);
        return 100;
    }

    // 호출부
    private async UniTaskVoid ShowScore()
    {
        int score = await FetchScoreAsync();
        Debug.Log($"점수: {score}");
    }
}
```

### 반환 타입 선택 기준

| 타입 | 설명 |
|---|---|
| `UniTask` | 반환값 없는 비동기 메서드 |
| `UniTask<T>` | 반환값 있는 비동기 메서드 |
| `UniTaskVoid` | Fire-and-forget (`.Forget()` 없이 사용 가능) |

---

## 4. UniTask 생성 패턴

```csharp
// 한 프레임 대기
await UniTask.Yield();
await UniTask.NextFrame();

// 시간 대기
await UniTask.Delay(1000);                            // 밀리초
await UniTask.Delay(TimeSpan.FromSeconds(1.5f));      // TimeSpan
await UniTask.WaitForSeconds(1.5f);                   // float 초

// 조건 대기
await UniTask.WaitUntil(() => isReady);
await UniTask.WaitWhile(() => isLoading);

// 특정 프레임 수 대기
await UniTask.DelayFrame(5);

// 즉시 완료
await UniTask.CompletedTask;

// 값 반환
UniTask<int> task = UniTask.FromResult(42);
```

---

## 5. 취소 (CancellationToken)

### MonoBehaviour 자동 취소

```csharp
public class CancelExample : MonoBehaviour
{
    private async UniTaskVoid Start()
    {
        // this.GetCancellationTokenOnDestroy() : 오브젝트 파괴 시 자동 취소
        await UniTask.Delay(5000, cancellationToken: this.GetCancellationTokenOnDestroy());
        Debug.Log("5초 완료 (파괴되지 않은 경우에만 출력)");
    }
}
```

### 수동 CancellationTokenSource

```csharp
public class ManualCancelExample : MonoBehaviour
{
    private CancellationTokenSource _cts;

    private void Start()
    {
        _cts = new CancellationTokenSource();
        RunAsync(_cts.Token).Forget();
    }

    private async UniTask RunAsync(CancellationToken ct)
    {
        try
        {
            await UniTask.Delay(10000, cancellationToken: ct);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("취소됨");
        }
    }

    public void Cancel() => _cts?.Cancel();

    private void OnDestroy()
    {
        _cts?.Cancel();
        _cts?.Dispose();
    }
}
```

### CancellationToken 연결 (Linked)

```csharp
using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
    _cts.Token,
    this.GetCancellationTokenOnDestroy()
);
await SomeAsync(linkedCts.Token);
```

---

## 6. 병렬 / 순차 실행

```csharp
// ── 병렬 실행 (모두 완료 대기) ──────────────────────────
var (a, b) = await UniTask.WhenAll(
    FetchDataAsync(),
    FetchImageAsync()
);

// 반환 타입이 다른 경우
await UniTask.WhenAll(
    UniTask.Delay(1000),
    UniTask.Delay(2000)
);

// ── 가장 빠른 것 하나만 대기 ────────────────────────────
int winner = await UniTask.WhenAny(
    TaskA(),   // 0 반환 시 이 태스크가 먼저 완료
    TaskB()    // 1 반환 시 이 태스크가 먼저 완료
);

// ── 순차 실행 ────────────────────────────────────────────
await StepOneAsync();
await StepTwoAsync();
await StepThreeAsync();
```

---

## 7. Unity 이벤트 대기

```csharp
using Cysharp.Threading.Tasks;
using UnityEngine;
using UnityEngine.UI;

public class EventAwaitExample : MonoBehaviour
{
    [SerializeField] private Button _confirmButton;
    [SerializeField] private Animator _animator;

    private async UniTaskVoid ShowDialogAndWait()
    {
        // 버튼 클릭 대기
        await _confirmButton.OnClickAsync(this.GetCancellationTokenOnDestroy());
        Debug.Log("확인 버튼 클릭됨");

        // 애니메이션 완료 대기
        await _animator.GetCurrentAnimatorStateInfo(0).normalizedTime < 1f
            ? UniTask.WaitUntil(() =>
                _animator.GetCurrentAnimatorStateInfo(0).normalizedTime >= 1f)
            : UniTask.CompletedTask;
    }

    // AsyncTrigger 활용 — 물리 이벤트 대기
    private async UniTaskVoid WaitForCollision()
    {
        var trigger = this.GetAsyncTrigger<AsyncCollisionTrigger>();
        var collision = await trigger.OnCollisionEnterAsync();
        Debug.Log($"충돌: {collision.gameObject.name}");
    }
}
```

---

## 8. PlayerLoop 타이밍

`UniTask.Yield(PlayerLoopTiming)`으로 실행 타이밍을 지정한다.

```csharp
await UniTask.Yield(PlayerLoopTiming.Update);           // Update 직후
await UniTask.Yield(PlayerLoopTiming.FixedUpdate);      // FixedUpdate
await UniTask.Yield(PlayerLoopTiming.LastUpdate);       // LateUpdate
await UniTask.Yield(PlayerLoopTiming.PreLateUpdate);    // LateUpdate 이전
await UniTask.Yield(PlayerLoopTiming.PostLateUpdate);   // LateUpdate 이후
```

---

## 9. 예외 처리

```csharp
// ── try/catch (권장) ─────────────────────────────────────
private async UniTask SafeLoadAsync()
{
    try
    {
        await SomeRiskyOperationAsync();
    }
    catch (OperationCanceledException)
    {
        Debug.Log("취소됨 — 정상 흐름");
    }
    catch (Exception e)
    {
        Debug.LogError($"오류: {e.Message}");
    }
}

// ── Forget()에서 예외 처리 ───────────────────────────────
SomeAsync()
    .Forget(e => Debug.LogException(e));

// ── 전역 예외 핸들러 등록 ────────────────────────────────
UniTaskScheduler.UnobservedTaskException += e =>
{
    Debug.LogException(e);
};
```

---

## 10. 주의사항 및 트러블슈팅

| 문제 | 원인 | 해결 |
|---|---|---|
| `UniTask` 반환 메서드에 `async void` 사용 | 예외가 삼켜짐 | `async UniTaskVoid` + `.Forget()` 사용 |
| GC 스파이크 발생 | `Task` 혼용 | 전부 `UniTask`로 교체 |
| 오브젝트 파괴 후 계속 실행 | 취소 토큰 미사용 | `GetCancellationTokenOnDestroy()` 연결 |
| `WhenAll` 중 하나 실패 시 나머지 미취소 | 기본 동작 | `LinkedTokenSource`로 수동 취소 |
| 에디터에서만 느림 | 디버그 모드 오버헤드 | 프로파일러로 `PlayerLoop` 타이밍 확인 |

---

> **참고 문서**  
> - [UniTask GitHub](https://github.com/Cysharp/UniTask)
