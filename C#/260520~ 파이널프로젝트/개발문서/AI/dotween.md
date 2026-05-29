# DOTween — 사용법 가이드

> Unity 고성능 트윈 엔진 | `com.demigiant.dotween` | Unity 5+

---

## 목차

1. [개요](#1-개요)
2. [설치 및 초기화](#2-설치-및-초기화)
3. [기본 트윈](#3-기본-트윈)
4. [Shortcut (단축 API)](#4-shortcut-단축-api)
5. [Ease 종류](#5-ease-종류)
6. [Sequence (시퀀스)](#6-sequence-시퀀스)
7. [콜백](#7-콜백)
8. [트윈 제어](#8-트윈-제어)
9. [UniTask 연동 (await)](#9-unitask-연동-await)
10. [주요 옵션 레퍼런스](#10-주요-옵션-레퍼런스)
11. [주의사항 및 트러블슈팅](#11-주의사항-및-트러블슈팅)

---

## 1. 개요

`DOTween`은 Unity에서 가장 널리 쓰이는 **트윈(Tween) 애니메이션 엔진**이다.  
코드 한 줄로 이동, 회전, 페이드, 크기 변환 등을 구현하며, Sequence로 복잡한 연출도 선언적으로 작성 가능하다.

---

## 2. 설치 및 초기화

### Asset Store / UPM

- [Asset Store — DOTween (HOTween v2)](https://assetstore.unity.com/packages/tools/animation/dotween-hotween-v2-27676)
- 설치 후 **Tools → Demigiant → DOTween Utility Panel → Setup DOTween** 실행

### 초기화 (선택)

DOTween은 자동 초기화되지만, 명시적으로 설정하면 더 많은 제어가 가능하다.

```csharp
using DG.Tweening;

public class AppInitializer : MonoBehaviour
{
    private void Awake()
    {
        DOTween.Init(
            recycleAllByDefault: true,  // 트윈 재사용 (GC 절감)
            useSafeMode: true,          // 에러 방지 모드
            logBehaviour: LogBehaviour.Default
        ).SetCapacity(200, 10);        // Tweener 200, Sequence 10 사전 할당
    }
}
```

---

## 3. 기본 트윈

```csharp
using DG.Tweening;
using UnityEngine;

public class BasicTweenExample : MonoBehaviour
{
    private void Start()
    {
        // ── DOTween.To : 값 직접 트윈 ────────────────────
        float myValue = 0f;
        DOTween.To(
            () => myValue,
            v => myValue = v,
            endValue: 10f,
            duration: 2f
        );

        // ── Transform 트윈 ───────────────────────────────
        transform.DOMove(new Vector3(5, 0, 0), 1f);        // 월드 이동
        transform.DOLocalMove(new Vector3(0, 2, 0), 0.5f); // 로컬 이동
        transform.DOMoveX(10f, 1f);                        // X축만 이동
        transform.DORotate(new Vector3(0, 180, 0), 1f);    // 회전
        transform.DOScale(Vector3.one * 2f, 0.3f);         // 스케일

        // ── RectTransform ────────────────────────────────
        GetComponent<RectTransform>().DOAnchorPos(new Vector2(100, 0), 0.5f);
        GetComponent<RectTransform>().DOSizeDelta(new Vector2(300, 100), 0.3f);

        // ── CanvasGroup / UI ─────────────────────────────
        GetComponent<CanvasGroup>().DOFade(0f, 1f);         // 페이드 아웃
        GetComponent<UnityEngine.UI.Image>().DOFade(1f, 0.5f);
        GetComponent<UnityEngine.UI.Text>().DOText("Hello!", 1f); // 타이핑 효과

        // ── 카메라 ────────────────────────────────────────
        Camera.main.DOFieldOfView(60f, 1f);
        Camera.main.DOOrthoSize(5f, 1f);

        // ── 오디오 ────────────────────────────────────────
        GetComponent<AudioSource>().DOFade(0f, 2f);
    }
}
```

---

## 4. Shortcut (단축 API)

DOTween의 확장 메서드로 컴포넌트에서 직접 호출한다.

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

---

## 5. Ease 종류

```csharp
// SetEase()로 곡선 지정
transform.DOMove(target, 1f).SetEase(Ease.OutBounce);
transform.DOScale(Vector3.one * 1.2f, 0.3f).SetEase(Ease.OutBack);

// ── 주요 Ease 값 ─────────────────────────────────────────
// Linear       : 선형
// InQuad       : 가속 (2차)        OutQuad  : 감속 (2차)
// InOutQuad    : 가속 후 감속       InOutCubic : 부드러운 S곡선
// InBack       : 약간 당겼다가 출발 OutBack  : 지나쳤다가 돌아옴
// OutBounce    : 바운스 감속         InElastic : 탄성 가속
// OutElastic   : 탄성 감속           Flash    : 점멸
// Custom AnimationCurve 사용
AnimationCurve myCurve = AnimationCurve.EaseInOut(0, 0, 1, 1);
transform.DOMove(target, 1f).SetEase(myCurve);
```

---

## 6. Sequence (시퀀스)

여러 트윈을 순서대로 또는 동시에 실행한다.

```csharp
public class SequenceExample : MonoBehaviour
{
    private void Start()
    {
        Sequence seq = DOTween.Sequence();

        // ── Append : 순차 실행 ────────────────────────────
        seq.Append(transform.DOMove(new Vector3(5, 0, 0), 1f));
        seq.Append(transform.DOScale(Vector3.one * 2f, 0.5f));
        seq.Append(transform.DORotate(new Vector3(0, 360, 0), 1f, RotateMode.FastBeyond360));

        // ── Join : 이전 Append와 동시 실행 ───────────────
        seq.Append(transform.DOMoveY(3f, 1f));
        seq.Join(GetComponent<SpriteRenderer>().DOFade(0f, 1f)); // 이동과 동시에 페이드

        // ── Insert : 지정 시간에 삽입 ─────────────────────
        seq.Insert(0.5f, transform.DOScale(Vector3.one * 1.5f, 0.3f)); // 0.5초 시점에 실행

        // ── AppendInterval : 대기 ────────────────────────
        seq.AppendInterval(1f); // 1초 대기

        // ── AppendCallback : 콜백 ─────────────────────────
        seq.AppendCallback(() => Debug.Log("중간 콜백"));

        // ── 옵션 ─────────────────────────────────────────
        seq.SetLoops(3, LoopType.Yoyo) // 3회 반복, 왕복
           .SetEase(Ease.InOutSine)
           .OnComplete(() => Debug.Log("시퀀스 완료"));
    }
}
```

---

## 7. 콜백

```csharp
transform.DOMove(target, 1f)
    .OnStart(() => Debug.Log("시작"))
    .OnUpdate(() => Debug.Log($"진행: {transform.position}"))
    .OnComplete(() => Debug.Log("완료"))
    .OnKill(() => Debug.Log("강제 종료"))
    .OnPlay(() => Debug.Log("재생"))
    .OnPause(() => Debug.Log("일시정지"))
    .OnStepComplete(() => Debug.Log("루프 1회 완료")); // SetLoops와 함께
```

---

## 8. 트윈 제어

```csharp
// ── 개별 트윈 제어 ────────────────────────────────────────
Tween tween = transform.DOMove(target, 2f);

tween.Play();
tween.Pause();
tween.Restart();
tween.Kill();                // 트윈 즉시 종료
tween.Kill(complete: true);  // 완료 상태로 종료 (endValue로 이동)
tween.Complete();            // 즉시 endValue로 이동
tween.Flip();                // 방향 반전
tween.Rewind();              // 처음으로 되감기
tween.TogglePause();

// ── 옵션 체이닝 ──────────────────────────────────────────
transform.DOMove(target, 1f)
    .SetDelay(0.5f)              // 0.5초 후 시작
    .SetLoops(-1, LoopType.Yoyo) // 무한 반복 왕복
    .SetSpeedBased()             // duration을 속도로 해석
    .SetRelative()               // endValue를 상대값으로
    .SetUpdate(true)             // TimeScale 무시 (UI 등에 유용)
    .SetId("moveAnim")           // ID 부여
    .SetAutoKill(false)          // 완료 후 자동 제거 안 함
    .SetRecyclable(true);        // 재사용 허용

// ── ID로 일괄 제어 ────────────────────────────────────────
DOTween.Pause("moveAnim");
DOTween.Play("moveAnim");
DOTween.Kill("moveAnim");

// ── 타겟으로 일괄 제어 ────────────────────────────────────
DOTween.Kill(transform);        // transform에 바인딩된 모든 트윈 종료
DOTween.PauseAll();
DOTween.KillAll();
```

---

## 9. UniTask 연동 (await)

```csharp
using DG.Tweening;
using Cysharp.Threading.Tasks;

public class TweenAwaitExample : MonoBehaviour
{
    private async UniTaskVoid Start()
    {
        // ── 트윈 완료 await ──────────────────────────────
        await transform.DOMove(new Vector3(5, 0, 0), 1f).ToUniTask();

        Debug.Log("이동 완료");

        // ── Sequence await ────────────────────────────────
        var seq = DOTween.Sequence()
            .Append(transform.DOMoveY(3f, 0.5f))
            .Append(transform.DOScale(Vector3.zero, 0.3f));

        await seq.ToUniTask();

        Debug.Log("시퀀스 완료");

        // ── 취소 토큰 연동 ────────────────────────────────
        var ct = this.GetCancellationTokenOnDestroy();
        await transform.DOMove(Vector3.zero, 2f)
            .ToUniTask(cancellationToken: ct);
    }
}
```

---

## 10. 주요 옵션 레퍼런스

| 메서드 | 설명 |
|---|---|
| `.SetDelay(float)` | 시작 지연 시간 |
| `.SetLoops(int, LoopType)` | 반복 횟수 (-1: 무한) |
| `.SetEase(Ease)` | 이징 곡선 |
| `.SetRelative()` | 상대값으로 계산 |
| `.SetSpeedBased()` | duration을 속도로 해석 |
| `.SetUpdate(bool)` | TimeScale 무시 여부 |
| `.SetId(object)` | 트윈 ID 부여 |
| `.SetAutoKill(bool)` | 완료 후 자동 파괴 (기본 true) |
| `.SetRecyclable(bool)` | 재사용 풀링 |
| `.From()` | 현재값 → startValue 방향 반전 |
| `.From(startValue)` | 지정 startValue에서 시작 |

| LoopType | 설명 |
|---|---|
| `Restart` | 처음부터 반복 |
| `Yoyo` | 왕복 반복 |
| `Incremental` | 누적 반복 |

---

## 11. 주의사항 및 트러블슈팅

| 문제 | 원인 | 해결 |
|---|---|---|
| 오브젝트 파괴 후 트윈 에러 | `SetAutoKill(false)` + 수동 Kill 누락 | `OnDestroy`에서 `DOTween.Kill(this)` 호출 |
| 트윈이 즉시 종료됨 | `SetAutoKill(true)` + `Pause` 후 미재생 | `.Play()` 명시 호출 |
| 스케일 0 이후 트윈 무반응 | Scale 0이면 일부 트윈 무시됨 | 최솟값 0.001 설정 |
| `SetUpdate(true)` 미적용 | `DOTween.Init` 이후 변경 | `Init` 전에 설정하거나 트윈 생성 시 지정 |
| GC 스파이크 | 트윈 반복 생성 | `SetAutoKill(false)` + `Restart()` 재사용 |
| TimeScale 0에서 멈춤 | 기본값은 TimeScale 영향받음 | `.SetUpdate(true)` 로 UnscaledTime 사용 |

---

> **참고 문서**  
> - [DOTween 공식 문서](http://dotween.demigiant.com/documentation.php)  
> - [DOTween Ease 시각화](https://easings.net/)
