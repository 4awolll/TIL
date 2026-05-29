# VContainer — 사용법 가이드

> Unity 전용 고성능 DI 컨테이너 | `com.hadashikick.vcontainer` | Unity 2019.3+

---

## 목차

1. [개요](#1-개요)
2. [설치](#2-설치)
3. [LifetimeScope 기본 구조](#3-lifetimescope-기본-구조)
4. [등록 방법 (Register)](#4-등록-방법-register)
5. [주입 방법 (Inject)](#5-주입-방법-inject)
6. [스코프 계층 구조](#6-스코프-계층-구조)
7. [MessagePipe 연동](#7-messagepipe-연동)
8. [EntryPoint (수명주기 훅)](#8-entrypoint-수명주기-훅)
9. [UniTask 연동](#9-unitask-연동)
10. [주의사항 및 트러블슈팅](#10-주의사항-및-트러블슈팅)

---

## 1. 개요

`VContainer`는 Unity를 위한 **경량 DI(Dependency Injection) 컨테이너**다.  
`Zenject`보다 빠르고 GC 부담이 적으며, Source Generator 기반 코드 생성으로 리플렉션을 최소화한다.

| Zenject | VContainer |
|---|---|
| `MonoInstaller` | `LifetimeScope` |
| `Container.Bind<T>()` | `builder.Register<T>()` |
| `[Inject]` | `[Inject]` (동일) |
| `IInitializable` | `IStartable` |
| `ITickable` | `ITickable` (동일) |

---

## 2. 설치

```json
{
  "dependencies": {
    "com.hadashikick.vcontainer": "https://github.com/hadashiA/VContainer.git?path=VContainer/Assets/VContainer"
  }
}
```

또는 OpenUPM:

```bash
openupm add com.hadashikick.vcontainer
```

---

## 3. LifetimeScope 기본 구조

`LifetimeScope`는 DI 컨테이너의 루트다. 씬에 하나의 `LifetimeScope` 파생 클래스를 컴포넌트로 추가한다.

```csharp
using VContainer;
using VContainer.Unity;

// 1. LifetimeScope 파생 클래스 생성
public class GameLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        // 여기에 모든 등록(Register) 작성
        builder.Register<PlayerService>(Lifetime.Singleton);
        builder.Register<EnemyService>(Lifetime.Transient);
        builder.RegisterEntryPoint<GameController>();
    }
}
```

> 씬 오브젝트에 `GameLifetimeScope` 컴포넌트를 추가하면 씬 로드 시 자동으로 컨테이너가 빌드된다.

---

## 4. 등록 방법 (Register)

```csharp
protected override void Configure(IContainerBuilder builder)
{
    // ── 순수 C# 클래스 ───────────────────────────────────
    builder.Register<MyService>(Lifetime.Singleton);        // 싱글톤
    builder.Register<MyService>(Lifetime.Transient);        // 매 주입마다 새 인스턴스
    builder.Register<MyService>(Lifetime.Scoped);           // 스코프 내 공유

    // ── 인터페이스 바인딩 ─────────────────────────────────
    builder.Register<MyService>(Lifetime.Singleton).As<IMyService>();
    builder.Register<MyService>(Lifetime.Singleton).AsImplementedInterfaces();
    builder.Register<MyService>(Lifetime.Singleton).AsSelf().As<IMyService>();

    // ── 인스턴스 직접 등록 ───────────────────────────────
    var config = new GameConfig { MaxEnemies = 10 };
    builder.RegisterInstance(config);
    builder.RegisterInstance(config).As<IGameConfig>();

    // ── MonoBehaviour 등록 ───────────────────────────────
    builder.RegisterComponentInHierarchy<PlayerController>();  // 씬에 있는 컴포넌트
    builder.RegisterComponent(_playerController);               // 직접 참조 (SerializeField)

    // ── ScriptableObject 등록 ────────────────────────────
    builder.RegisterInstance(_configSO).As<IConfig>();

    // ── 팩토리 등록 ─────────────────────────────────────
    builder.Register<EnemyFactory>(Lifetime.Singleton);
    builder.RegisterFactory<Enemy>(container =>
        () => container.Resolve<Enemy>(),
        Lifetime.Singleton
    );
}
```

---

## 5. 주입 방법 (Inject)

### 생성자 주입 (권장)

```csharp
public class PlayerService
{
    private readonly IEnemyRepository _enemyRepo;
    private readonly GameConfig _config;

    // 생성자 주입 — [Inject] 생략 가능 (생성자가 하나인 경우)
    public PlayerService(IEnemyRepository enemyRepo, GameConfig config)
    {
        _enemyRepo = enemyRepo;
        _config = config;
    }
}
```

### MonoBehaviour 메서드 주입

```csharp
using VContainer;
using UnityEngine;

public class PlayerController : MonoBehaviour
{
    private IPlayerService _playerService;
    private GameConfig _config;

    // MonoBehaviour는 생성자 사용 불가 → [Inject] 메서드 주입
    [Inject]
    public void Construct(IPlayerService playerService, GameConfig config)
    {
        _playerService = playerService;
        _config = config;
    }
}
```

### 프로퍼티 / 필드 주입

```csharp
public class UiPresenter : MonoBehaviour
{
    [Inject] private IPlayerModel _playerModel;       // 필드 주입
    [Inject] public IScoreService ScoreService { get; private set; }  // 프로퍼티 주입
}
```

### Resolve (수동)

```csharp
// LifetimeScope가 있는 씬에서
var scope = LifetimeScope.Find<GameLifetimeScope>();
var service = scope.Container.Resolve<IPlayerService>();
```

---

## 6. 스코프 계층 구조

씬 전환이나 게임 모드 변경 시 **부모-자식 스코프**를 활용한다.

```csharp
// 부모 스코프 (앱 전체 싱글톤)
public class AppLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.Register<UserDataService>(Lifetime.Singleton);
        builder.Register<NetworkService>(Lifetime.Singleton);
    }
}

// 자식 스코프 (게임 씬 전용)
public class GameSceneLifetimeScope : LifetimeScope
{
    // Inspector에서 Parent에 AppLifetimeScope 연결
    protected override void Configure(IContainerBuilder builder)
    {
        // 부모 스코프의 UserDataService, NetworkService 자동 접근 가능
        builder.Register<PlayerService>(Lifetime.Singleton);
        builder.RegisterEntryPoint<GameController>();
    }
}
```

### 동적 자식 스코프 생성

```csharp
// 런타임에 스코프 생성 (예: 던전 입장 시)
using (LifetimeScope.EnqueueParent(parentScope))
{
    var childScope = Instantiate(_dungeonScopePrefab);
}
```

---

## 7. MessagePipe 연동

`VContainer`와 `MessagePipe`를 함께 사용하면 이벤트 버스를 DI로 관리한다.

```csharp
// 등록
builder.AddMessagePipe();
builder.AddMessageBroker<DamageEvent>();
builder.AddMessageBroker<GameOverEvent>();

// 발행자
public class EnemyService
{
    private readonly IPublisher<DamageEvent> _publisher;

    public EnemyService(IPublisher<DamageEvent> publisher)
        => _publisher = publisher;

    public void Attack(int damage)
        => _publisher.Publish(new DamageEvent { Amount = damage });
}

// 구독자
public class HpPresenter : IStartable, IDisposable
{
    private readonly ISubscriber<DamageEvent> _subscriber;
    private IDisposable _subscription;

    public HpPresenter(ISubscriber<DamageEvent> subscriber)
        => _subscriber = subscriber;

    public void Start()
        => _subscription = _subscriber.Subscribe(e => Debug.Log($"데미지: {e.Amount}"));

    public void Dispose() => _subscription?.Dispose();
}
```

---

## 8. EntryPoint (수명주기 훅)

`RegisterEntryPoint`로 등록하면 Unity 수명주기에 연결된다.

```csharp
public class GameController : IStartable, ITickable, IFixedTickable, IDisposable
{
    private readonly IPlayerService _player;

    public GameController(IPlayerService player) => _player = player;

    public void Start()
    {
        Debug.Log("씬 시작");
    }

    public void Tick()
    {
        // Update에 해당
    }

    public void FixedTick()
    {
        // FixedUpdate에 해당
    }

    public void Dispose()
    {
        Debug.Log("스코프 파괴");
    }
}

// 등록
builder.RegisterEntryPoint<GameController>();
builder.RegisterEntryPoint<GameController>().AsSelf(); // Resolve 가능하게
```

| 인터페이스 | 대응 이벤트 |
|---|---|
| `IStartable` | Start |
| `IPostStartable` | Start 이후 |
| `ITickable` | Update |
| `IPostTickable` | Update 이후 |
| `IFixedTickable` | FixedUpdate |
| `ILateTickable` | LateUpdate |
| `IDisposable` | 스코프 파괴 시 |
| `IAsyncStartable` | Start (UniTask 비동기) |

---

## 9. UniTask 연동

```csharp
using VContainer.Unity;
using Cysharp.Threading.Tasks;

public class AsyncLoader : IAsyncStartable
{
    private readonly IDataRepository _repo;

    public AsyncLoader(IDataRepository repo) => _repo = repo;

    // 씬 Start 타이밍에 비동기 초기화
    public async UniTask StartAsync(CancellationToken ct)
    {
        await _repo.LoadAsync(ct);
        Debug.Log("데이터 로드 완료");
    }
}

// 등록
builder.RegisterEntryPoint<AsyncLoader>();
```

---

## 10. 주의사항 및 트러블슈팅

| 문제 | 원인 | 해결 |
|---|---|---|
| `VContainerException: Type not registered` | Register 누락 | `Configure`에서 해당 타입 등록 확인 |
| MonoBehaviour 주입 안 됨 | 생성자 주입 시도 | `[Inject]` 메서드 주입으로 변경 |
| 자식 스코프에서 부모 타입 미해결 | Parent 연결 누락 | Inspector에서 Parent 필드 연결 |
| `Lifetime.Singleton`인데 여러 인스턴스 생성 | 스코프가 다름 | 스코프 계층 재확인 |
| Source Generator 비활성화 시 느림 | 리플렉션 사용 | `VContainer.EnableCodeGeneration` 활성화 |

---

> **참고 문서**  
> - [VContainer 공식 문서](https://vcontainer.hadashikick.jp/)  
> - [VContainer GitHub](https://github.com/hadashiA/VContainer)
