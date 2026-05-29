# Unity Addressable Asset System — 리소스 로드 및 관리

> Unity 2019.3+ 권장 | `com.unity.addressables` 패키지 기준

---

## 목차

1. [개요](#1-개요)
2. [설치 및 초기 설정](#2-설치-및-초기-설정)
3. [Addressable 등록](#3-addressable-등록)
4. [런타임 로드 API](#4-런타임-로드-api)
5. [씬 로드](#5-씬-로드)
6. [배치 로드 (LoadAssetsAsync)](#6-배치-로드-loadassetsasync)
7. [메모리 관리 및 해제](#7-메모리-관리-및-해제)
8. [원격 번들 (Remote Hosting)](#8-원격-번들-remote-hosting)
9. [그룹 전략 및 빌드 설정](#9-그룹-전략-및-빌드-설정)
10. [자주 쓰는 패턴](#10-자주-쓰는-패턴)
11. [주의사항 및 트러블슈팅](#11-주의사항-및-트러블슈팅)

---

## 1. 개요

Addressable Asset System은 Unity의 **리소스 참조 방식을 주소(string/label) 기반으로 추상화**하는 패키지다.  
`Resources.Load`, `AssetBundle` 직접 관리의 단점을 해소하고, 로컬/원격 번들을 동일한 API로 다룬다.

| 기존 방식 | Addressables |
|---|---|
| `Resources.Load<T>("path")` | `Addressables.LoadAssetAsync<T>("key")` |
| AssetBundle 직접 로드/언로드 | 자동 참조 카운팅 |
| 경로 하드코딩 | 주소(Address) or 레이블(Label) |
| 빌드 포함 고정 | 로컬 + 원격 번들 혼용 가능 |

---

## 2. 설치 및 초기 설정

### 패키지 설치

**Package Manager → Add by name**
```
com.unity.addressables
```

### Groups 창 열기

```
Window → Asset Management → Addressables → Groups
```

초기 실행 시 `AddressableAssetSettings` 에셋이 자동 생성된다 (`Assets/AddressableAssetsData/`).

### 필수 설정 확인

| 항목 | 위치 | 권장값 |
|---|---|---|
| Build Path | Profile → LocalBuildPath | `[UnityEngine.AddressableAssets.Addressables.BuildPath]` |
| Load Path | Profile → LocalLoadPath | `{UnityEngine.AddressableAssets.Addressables.RuntimePath}` |
| Catalog Update | RemoteCatalogBuildPath | 원격 배포 시 설정 |

---

## 3. Addressable 등록

### Inspector에서 등록

에셋 선택 → Inspector 하단 **"Addressable"** 체크박스 활성화.  
기본 주소는 파일 경로가 자동 입력되며 자유롭게 수정 가능하다.

### 레이블(Label) 활용

```
Groups 창 → 에셋 선택 → Labels 열 클릭 → 레이블 추가
```

레이블을 사용하면 복수 에셋을 하나의 키로 일괄 로드할 수 있다.

### 스크립트로 등록 (에디터 전용)

```csharp
#if UNITY_EDITOR
using UnityEditor.AddressableAssets;
using UnityEditor.AddressableAssets.Settings;

public static void RegisterAsset(string assetPath, string address, string groupName = "Default Local Group")
{
    var settings = AddressableAssetSettingsDefaultObject.Settings;
    var group = settings.FindGroup(groupName) ?? settings.DefaultGroup;

    var guid = AssetDatabase.AssetPathToGUID(assetPath);
    var entry = settings.CreateOrMoveEntry(guid, group);
    entry.address = address;

    settings.SetDirty(AddressableAssetSettings.ModificationEvent.EntryMoved, entry, true);
}
#endif
```

---

## 4. 런타임 로드 API

### 기본 로드 패턴 (async/await)

```csharp
using UnityEngine;
using UnityEngine.AddressableAssets;
using UnityEngine.ResourceManagement.AsyncOperations;

public class AddressableLoader : MonoBehaviour
{
    [SerializeField] private string _address = "Characters/Player";
    [SerializeField] private AssetReference _assetRef; // Inspector 직접 연결

    private AsyncOperationHandle<GameObject> _handle;

    // ── 주소(string) 로드 ──────────────────────────────────
    public async void LoadByAddress()
    {
        _handle = Addressables.LoadAssetAsync<GameObject>(_address);
        await _handle.Task;

        if (_handle.Status == AsyncOperationStatus.Succeeded)
        {
            Instantiate(_handle.Result, transform.position, Quaternion.identity);
        }
        else
        {
            Debug.LogError($"로드 실패: {_handle.OperationException}");
        }
    }

    // ── AssetReference 로드 (Inspector 직접 연결 권장) ──────
    public async void LoadByReference()
    {
        var handle = _assetRef.LoadAssetAsync<GameObject>();
        await handle.Task;

        if (handle.Status == AsyncOperationStatus.Succeeded)
        {
            Instantiate(handle.Result);
        }
    }

    // ── Instantiate 단축 API ────────────────────────────────
    public async void InstantiateDirectly()
    {
        var handle = Addressables.InstantiateAsync(_address, transform.position, Quaternion.identity);
        await handle.Task;

        // 해제 시 Addressables.ReleaseInstance(handle.Result) 사용
    }

    private void OnDestroy()
    {
        // 직접 로드한 경우 반드시 수동 해제
        if (_handle.IsValid())
            Addressables.Release(_handle);
    }
}
```

### 로드 방식 비교

| API | 반환값 | 해제 방법 |
|---|---|---|
| `LoadAssetAsync<T>` | `AsyncOperationHandle<T>` | `Addressables.Release(handle)` |
| `InstantiateAsync` | `AsyncOperationHandle<GameObject>` | `Addressables.ReleaseInstance(go)` |
| `AssetReference.LoadAssetAsync<T>` | `AsyncOperationHandle<T>` | `assetRef.ReleaseAsset()` |
| `AssetReference.InstantiateAsync` | `AsyncOperationHandle<GameObject>` | `assetRef.ReleaseInstance(go)` |

---

## 5. 씬 로드

```csharp
using UnityEngine.AddressableAssets;
using UnityEngine.ResourceManagement.AsyncOperations;
using UnityEngine.ResourceManagement.ResourceProviders;
using UnityEngine.SceneManagement;

public class SceneLoader : MonoBehaviour
{
    private AsyncOperationHandle<SceneInstance> _sceneHandle;

    public async void LoadScene(string sceneAddress, LoadSceneMode mode = LoadSceneMode.Single)
    {
        _sceneHandle = Addressables.LoadSceneAsync(sceneAddress, mode);
        await _sceneHandle.Task;

        if (_sceneHandle.Status != AsyncOperationStatus.Succeeded)
            Debug.LogError("씬 로드 실패");
    }

    public async void UnloadScene()
    {
        if (_sceneHandle.IsValid())
            await Addressables.UnloadSceneAsync(_sceneHandle).Task;
    }
}
```

---

## 6. 배치 로드 (LoadAssetsAsync)

레이블 또는 복수 키로 여러 에셋을 한 번에 로드한다.

```csharp
using System.Collections.Generic;
using UnityEngine.AddressableAssets;
using UnityEngine.AddressableAssets.ResourceLocators;
using UnityEngine.ResourceManagement.AsyncOperations;

public class BatchLoader : MonoBehaviour
{
    private AsyncOperationHandle<IList<Sprite>> _batchHandle;

    // ── 레이블로 일괄 로드 ──────────────────────────────────
    public async void LoadByLabel(string label)
    {
        _batchHandle = Addressables.LoadAssetsAsync<Sprite>(
            label,
            sprite => Debug.Log($"개별 콜백: {sprite.name}")  // 각 에셋 로드 완료 시 호출
        );

        await _batchHandle.Task;

        if (_batchHandle.Status == AsyncOperationStatus.Succeeded)
        {
            foreach (var sprite in _batchHandle.Result)
                Debug.Log(sprite.name);
        }
    }

    // ── 복수 키 + AND/OR 조건 ─────────────────────────────
    public async void LoadByMultipleKeys()
    {
        var keys = new List<string> { "UI", "HUD" };

        var handle = Addressables.LoadAssetsAsync<Sprite>(
            keys,
            null,
            Addressables.MergeMode.Union   // OR: Union / AND: Intersection
        );

        await handle.Task;
        Addressables.Release(handle);
    }

    private void OnDestroy()
    {
        if (_batchHandle.IsValid())
            Addressables.Release(_batchHandle);
    }
}
```

---

## 7. 메모리 관리 및 해제

Addressables는 **참조 카운팅** 방식으로 동작한다.  
`LoadAssetAsync` 횟수만큼 `Release`를 호출해야 번들이 언로드된다.

```csharp
// ✅ 올바른 해제
Addressables.Release(handle);           // LoadAssetAsync 대응
Addressables.ReleaseInstance(instance); // InstantiateAsync 대응

// ❌ 흔한 실수
// handle.Release() — 존재하지 않음
// Destroy(instance) — 번들이 언로드되지 않음 (메모리 누수)
```

### 에셋별 수명 주기 권장 패턴

```csharp
public class ManagedAsset<T> : System.IDisposable where T : Object
{
    public T Asset { get; private set; }
    private AsyncOperationHandle<T> _handle;

    public async System.Threading.Tasks.Task LoadAsync(string address)
    {
        _handle = Addressables.LoadAssetAsync<T>(address);
        await _handle.Task;
        Asset = _handle.Status == AsyncOperationStatus.Succeeded ? _handle.Result : null;
    }

    public void Dispose()
    {
        if (_handle.IsValid())
            Addressables.Release(_handle);
    }
}

// 사용 예시
var asset = new ManagedAsset<Texture2D>();
await asset.LoadAsync("Textures/Background");
// ... 사용 ...
asset.Dispose(); // using 블록으로도 사용 가능
```

---

## 8. 원격 번들 (Remote Hosting)

### 프로파일 설정

```
Addressables → Profiles → 새 프로파일 생성 (예: Remote)

RemoteBuildPath  : ServerData/[BuildTarget]
RemoteLoadPath   : https://your-cdn.com/[BuildTarget]
```

### 카탈로그 업데이트 확인

```csharp
public class CatalogUpdater : MonoBehaviour
{
    public async void CheckAndUpdate()
    {
        // 1. 업데이트 가능한 카탈로그 확인
        var checkHandle = Addressables.CheckForCatalogUpdates(false);
        await checkHandle.Task;

        var catalogs = checkHandle.Result;
        Addressables.Release(checkHandle);

        if (catalogs == null || catalogs.Count == 0)
        {
            Debug.Log("업데이트 없음");
            return;
        }

        // 2. 카탈로그 업데이트
        var updateHandle = Addressables.UpdateCatalogs(catalogs, false);
        await updateHandle.Task;
        Addressables.Release(updateHandle);

        // 3. 번들 다운로드 사이즈 확인 후 다운로드
        var sizeHandle = Addressables.GetDownloadSizeAsync("RemoteLabel");
        await sizeHandle.Task;

        if (sizeHandle.Result > 0)
        {
            var dlHandle = Addressables.DownloadDependenciesAsync("RemoteLabel", false);
            await dlHandle.Task;
            Addressables.Release(dlHandle);
        }

        Addressables.Release(sizeHandle);
    }
}
```

---

## 9. 그룹 전략 및 빌드 설정

### 그룹 분리 기준

| 그룹 | 포함 에셋 | 번들 모드 |
|---|---|---|
| Default Local Group | 필수 초기 에셋 (UI, 폰트 등) | Pack Together |
| Characters | 캐릭터 프리팹, 애니메이션 | Pack Separately |
| Levels | 씬, 레벨 별 에셋 | Pack Together By Label |
| Remote Content | DLC, 패치 콘텐츠 | Pack Together |

### 빌드 명령어

```csharp
// 에디터 스크립트로 자동화
#if UNITY_EDITOR
using UnityEditor.AddressableAssets.Build;
using UnityEditor.AddressableAssets.Settings;

[UnityEditor.MenuItem("Build/Addressables")]
public static void BuildAddressables()
{
    AddressableAssetSettings.BuildPlayerContent(out var result);

    if (!string.IsNullOrEmpty(result.Error))
        UnityEngine.Debug.LogError($"빌드 실패: {result.Error}");
    else
        UnityEngine.Debug.Log($"빌드 완료: {result.Duration}초");
}
#endif
```

### CI 커맨드라인 빌드

```bash
Unity -batchmode -quit \
  -projectPath /path/to/project \
  -executeMethod AddressableBuildScript.BuildAddressables \
  -logFile build.log
```

---

## 10. 자주 쓰는 패턴

### 풀링(Pooling)과 Addressables 결합

```csharp
public class AddressablePool : MonoBehaviour
{
    [SerializeField] private string _address;
    [SerializeField] private int _initialSize = 10;

    private readonly Queue<GameObject> _pool = new();
    private AsyncOperationHandle<GameObject> _assetHandle;

    private async void Start()
    {
        _assetHandle = Addressables.LoadAssetAsync<GameObject>(_address);
        await _assetHandle.Task;

        for (int i = 0; i < _initialSize; i++)
            _pool.Enqueue(CreateInstance());
    }

    private GameObject CreateInstance()
    {
        var go = Instantiate(_assetHandle.Result);
        go.SetActive(false);
        return go;
    }

    public GameObject Get(Vector3 position)
    {
        var go = _pool.Count > 0 ? _pool.Dequeue() : CreateInstance();
        go.transform.position = position;
        go.SetActive(true);
        return go;
    }

    public void Return(GameObject go)
    {
        go.SetActive(false);
        _pool.Enqueue(go);
    }

    private void OnDestroy()
    {
        if (_assetHandle.IsValid())
            Addressables.Release(_assetHandle);
    }
}
```

### 진행률 표시

```csharp
public async void LoadWithProgress(string address, System.Action<float> onProgress)
{
    var handle = Addressables.LoadAssetAsync<GameObject>(address);

    while (!handle.IsDone)
    {
        onProgress?.Invoke(handle.PercentComplete);
        await System.Threading.Tasks.Task.Yield();
    }

    onProgress?.Invoke(1f);
}
```

---

## 11. 주의사항 및 트러블슈팅

| 문제 | 원인 | 해결 |
|---|---|---|
| `InvalidKeyException` | 주소가 Groups에 없음 | Groups 창에서 주소 확인, 빌드 재실행 |
| 메모리 누수 | `Release` 미호출 | 모든 `Load` 에 대응하는 `Release` 필수 |
| 에디터와 빌드 동작 차이 | 빌드 없이 Play Mode | `Use Asset Database (fastest)` 모드로 테스트 후 실제 빌드 검증 |
| 원격 번들 로드 실패 | CDN URL 불일치 | `RemoteLoadPath` 프로파일 값 확인 |
| 중복 번들 증가 | 공유 에셋 미분리 | 공용 에셋을 별도 그룹으로 분리 |
| `handle.IsValid()` = false | 이미 Release 됨 | 이중 해제 방지 로직 추가 |

### Play Mode Script 선택 기준

| 모드 | 설명 | 용도 |
|---|---|---|
| Use Asset Database (fastest) | 번들 없이 DB 직접 참조 | 빠른 반복 개발 |
| Simulate Groups (advanced) | 번들 구조 시뮬레이션 | 그룹 구성 검증 |
| Use Existing Build | 실제 번들 사용 | 최종 통합 테스트 |

---

> **참고 문서**  
> - [Unity Addressables 공식 문서](https://docs.unity3d.com/Packages/com.unity.addressables@latest)  
> - [Addressables Best Practices](https://unity.com/how-to/best-practices-assets-unity-addressables)
