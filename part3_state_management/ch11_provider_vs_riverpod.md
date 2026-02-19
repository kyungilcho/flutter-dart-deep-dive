# Chapter 11: Provider → Riverpod — 무엇이 달라졌는가

## 11.1 왜 Riverpod인가?

Provider가 `InheritedWidget` 위의 얇은 래퍼라면, Riverpod은 **위젯 트리에서 완전히 독립된 상태 그래프**다.
Provider에서 Riverpod으로 진화한 이유를 이해하려면, Provider의 구조적 한계를 먼저 정확히 파악해야 한다.

### Provider의 구조적 한계

**1. 위젯 트리 종속성**

Provider는 `InheritedWidget`에 기반하므로, 프로바이더의 위치가 위젯 트리에 의해 결정된다:

```dart
// Provider: 위젯 트리 안에 존재해야 한다
MaterialApp(
  home: ChangeNotifierProvider(  // ← 여기에 배치해야
    create: (_) => CounterModel(),
    child: CounterPage(),
  ),
);
```

이로 인해:
- 같은 타입의 프로바이더를 **트리 내 여러 위치에 배치**하면, `Provider.of<T>(context)`는 가장 가까운 것만 찾는다
- 프로바이더 접근 시 반드시 `BuildContext`가 필요하다
- **컴파일 타임 안전성**이 없다 — 프로바이더가 트리에 없으면 **런타임** `ProviderNotFoundException`

**2. 런타임 타입 조회의 한계**

```dart
// Provider 내부: 타입으로 조회
static _InheritedProviderScopeElement<T?>? _inheritedElementOf<T>(
  BuildContext context,
) {
  final inheritedElement = context.getElementForInheritedWidgetOfExactType<
      _InheritedProviderScope<T?>>() as _InheritedProviderScopeElement<T?>?;
  // ...
}
```

`getElementForInheritedWidgetOfExactType<T>()`는 **제네릭 타입 `T`**로만 조회하므로:
- 같은 타입의 프로바이더 **2개**를 구분할 수 없다
- `Provider<String>`이 2개 있으면, 구분 방법이 없다

**3. 프로바이더 간 의존성 관리**

Provider에서 프로바이더 A가 프로바이더 B를 읽으려면, 반드시 위젯 트리에서 B가 A의 상위에 있어야 한다.
이 의존 관계가 복잡해지면 `MultiProvider`의 순서가 **암묵적 의존성 계약**이 된다.

### 커뮤니티의 컴파일 타임 안전성 노력과 그 한계

위의 한계, 특히 **컴파일 타임 안전성 부재**를 해결하려는 커뮤니티 차원의 시도가 있었다.

**1. custom_lint를 통한 정적 분석**

Provider 작성자인 Remi Rousselet 본인이 `custom_lint` 패키지를 만들었다.
이를 통해 Provider 사용 시 흔한 안티패턴을 정적으로 감지할 수 있다:

- `context.read`를 `build()` 안에서 호출하면 경고
- `Provider.of(context, listen: false)`를 build 메서드 안에서 쓰면 경고

하지만 이것은 **"잘못된 사용법을 경고"**하는 수준이지, **프로바이더가 위젯 트리에 존재하는지** 컴파일 시점에 보장하는 것은 원리적으로 불가능하다.

**2. 왜 구조적으로 불가능한가?**

Provider의 상태 접근은 `getElementForInheritedWidgetOfExactType`에 의존한다:

```dart
final inheritedElement = context.getElementForInheritedWidgetOfExactType<
    _InheritedProviderScope<T?>>() as _InheritedProviderScopeElement<T?>?;
// 없으면? → 런타임 ProviderNotFoundException
```

이 API는 **Flutter 프레임워크의 런타임 트리 탐색**이다.
Dart 컴파일러가 "이 위젯이 런타임에 트리의 상위에 존재할 것인가?"를 
미리 판단할 방법은 없다. `InheritedWidget` 기반인 한 **원리적으로 해결 불가능**하다.

**3. 대안적 접근들과 그 한계**

| 접근 | 패키지 | 전략 | 한계 |
|------|--------|------|------|
| 서비스 로케이터 | `get_it` | 전역 Map에 타입 등록 | 런타임 조회, 위젯 리빌드 미지원 |
| 코드 생성 DI | `injectable` (get_it 기반) | `@Injectable` 어노테이션 → DI 그래프 생성 | 빌드 시 검증 가능하나 위젯 바인딩 없음 |
| 커스텀 린트 | `riverpod_lint` | Riverpod 전용 린트 규칙 | Riverpod 도입이 전제 |

`get_it` + `injectable` 조합이 DI 레벨의 안전성을 잡으려 했지만,
이것은 **의존성 주입(DI)**의 문제를 해결한 것이지, **상태 관리와 위젯 리빌드**를 해결하지 못했다.

**4. Riverpod의 탄생 — Remi 본인의 결론**

Remi Rousselet은 Provider를 만들고 유지하면서 이 한계를 가장 잘 인지하고 있었다:

> "Provider는 InheritedWidget의 래퍼이므로, InheritedWidget의 한계를 그대로 상속한다.
> 타입 기반 조회, BuildContext 의존, 같은 타입 다수 불가 — 
> 이것들은 래퍼 수준에서 해결할 수 없는 구조적 문제다."

그의 해결 방식은 InheritedWidget을 더 잘 감싸는 것이 아니라,
**InheritedWidget은 container 참조 전달에만 쓰고, 상태 그래프를 위젯 트리 밖에 새로 만드는 것**이었다.

```dart
// Provider: 런타임 타입 조회 → 존재 여부는 실행해봐야 안다
final user = Provider.of<UserModel>(context); // ← ProviderNotFoundException 가능

// Riverpod: 전역 변수 참조 → 컴파일러가 존재를 보장
final userProvider = Provider((_) => UserModel());
final user = ref.watch(userProvider); // ← userProvider 미정의 시 컴파일 에러
```

Riverpod이라는 이름 자체가 **Provider의 아나그램**(Provider → Riverpod, 글자 재배열)인 것도 
"같은 문제를 근본부터 다시 풀었다"는 의미를 담고 있다.

---

## 11.2 아키텍처 비교: 위젯 트리 vs 프로바이더 그래프

### Provider 아키텍처

```
Widget Tree
    │
    ├─ ProviderScope (MultiProvider)
    │   ├─ ChangeNotifierProvider<AuthModel>       ← InheritedWidget
    │   │   └─ ChangeNotifierProvider<UserModel>   ← InheritedWidget (AuthModel에 의존)
    │   │       └─ Consumer Widget
    │   │            └─ Provider.of<UserModel>(context)  ← 트리 탐색
    │
    └─ 상태 위치 = 위젯 트리 위치
```

Provider는 **`InheritedWidget`의 트리 탐색 메커니즘**을 그대로 사용한다.
Ch10에서 분석했듯이, `_InheritedProviderScopeElement`가 `notifyClients`를 호출하면 
등록된 모든 의존자(dependent)의 `didChangeDependency()`가 실행된다.

### Riverpod 아키텍처

```
ProviderContainer (독립 객체)
    │
    ├─ ProviderElement<AuthState>      ← 자체 관리
    │   ├─ ref.watch(configProvider)   ← 그래프 에지
    │   └─ listeners: [Consumer1, Provider2]
    │
    ├─ ProviderElement<UserState>
    │   ├─ ref.watch(authProvider)     ← 그래프 에지
    │   └─ listeners: [Consumer2]
    │
    └─ 상태 위치 = ProviderContainer (위젯 트리와 무관)

Widget Tree
    │
    └─ ProviderScope ← ProviderContainer를 InheritedWidget으로 전달할 뿐
        └─ ConsumerWidget
             └─ ref.watch(userProvider) ← container에서 직접 조회
```

핵심 차이: **Riverpod에서 `InheritedWidget`은 `ProviderContainer`를 전달하는 데만 사용**된다.
상태 관리 자체는 위젯 트리 밖의 `ProviderContainer` → `ProviderElement` 그래프에서 이루어진다.

---

## 11.3 Ref란 무엇인가 — BuildContext에서 Ref로

Riverpod 코드를 처음 접하면 `ref.watch(provider)`, `ref.read(provider)` 같은 표현이 낯설 수 있다.
`ref`가 정확히 무엇이고, 왜 필요한지를 먼저 이해해야 이후 소스코드 분석이 수월하다.

### Provider에서의 상태 접근: BuildContext

Provider에서는 상태에 접근하려면 **항상 `BuildContext`가 필요**했다:

```dart
// Provider 방식
@override
Widget build(BuildContext context) {
  final counter = context.watch<CounterModel>();  // BuildContext 기반
  final auth = context.read<AuthModel>();           // BuildContext 기반
  return Text('${counter.count}');
}
```

`BuildContext`는 **위젯이 트리에서 자기 위치를 아는 신분증**이다.
`context.watch<T>()`는 "내 위치에서 위로 올라가며 `T`를 제공하는 InheritedWidget을 찾아라"라는 명령이다.

이 모델에서는 `BuildContext`가 없으면(예: 서비스 클래스, 유틸 함수) 프로바이더에 접근할 수 없다.

### Riverpod에서의 상태 접근: Ref

Riverpod은 **상태가 위젯 트리 밖**(`ProviderContainer`)에 존재한다.
따라서 상태에 접근하는 데 "위젯 트리에서의 위치(BuildContext)"가 불필요하다.
대신, `ProviderContainer`와 통신할 수 있는 **별도의 접근 통로**가 필요한데, 그것이 **`Ref`**다.

```dart
// Riverpod 방식
final counterProvider = NotifierProvider<Counter, int>(Counter.new);

class Counter extends Notifier<int> {
  @override
  int build() {
    // ref: 다른 프로바이더에 접근하는 통로
    final config = ref.watch(configProvider);  // ← BuildContext 없이!
    return config.initialCount;
  }
}
```

### Ref의 본질: 소스코드로 증명

`Ref`의 정체를 소스코드에서 확인하자:

```dart
// riverpod/lib/src/core/ref.dart
sealed class Ref implements MutationTarget {
  // ※ MutationTarget: Riverpod의 Mutation API를 위한 인터페이스.
  //    ProviderContainer에 접근 가능한 객체(Ref, WidgetRef, Container)의 공통 계약.
  ProviderElement<Object?, Object?> get _element;
  ProviderContainer get container => _element.container;
  // ...
}
```

`Ref`는 결국 **`ProviderElement`에 대한 래퍼**다.
`ProviderElement`는 `ProviderContainer` 안에서 하나의 프로바이더 상태를 관리하는 단위이므로,
`Ref`는 **"현재 프로바이더가 소속된 container를 통해 다른 프로바이더에 접근하는 인터페이스"**다.

### 비유로 정리

```
Provider 세계:
  BuildContext = "위젯 트리에서의 내 위치" (신분증)
  context.watch<T>() = "내 위에 있는 T를 찾아서 의존" (위로 탐색)

Riverpod 세계:
  Ref = "ProviderContainer로의 접근 통로" (리모컨)
  ref.watch(provider) = "이 리모컨으로 저 프로바이더를 구독" (그래프 조회)
```

핵심 차이:
- `BuildContext`는 **위젯 트리의 위치에 종속** → 위젯 밖에서 사용 불가
- `Ref`는 **ProviderContainer에 연결** → 위치 무관, 프로바이더 내부 어디서든 사용 가능

### Ref vs WidgetRef — 두 개의 인터페이스

Riverpod에는 `Ref`가 **두 곳**에서 등장한다:

| | `Ref` | `WidgetRef` |
|---|---|---|
| **사용 위치** | 프로바이더 내부 | 위젯 내부 |
| **소유자** | `ProviderElement` | `ConsumerStatefulElement` |
| **`watch()`의 효과** | 현재 프로바이더 **invalidate** | 위젯 **markNeedsBuild** |
| **생명주기** | 프로바이더의 생명주기 | 위젯의 생명주기 |

```dart
// ① Ref — 프로바이더 안에서
final userProvider = Provider((ref) {         // ← ref: Ref
  final token = ref.watch(authProvider);       // 의존 등록
  return UserRepository(token);
});

// ② WidgetRef — 위젯 안에서
class UserPage extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {  // ← ref: WidgetRef
    final user = ref.watch(userProvider);               // 구독
    return Text(user.name);
  }
}
```

이름은 같은 `ref`이지만, **`Ref`의 `watch()`는 프로바이더를 무효화**하고 
**`WidgetRef`의 `watch()`는 위젯을 리빌드**한다.
이 차이의 내부 메커니즘은 다음 섹션에서 소스코드로 분석한다.

---

## 11.4 ProviderContainer — 상태 그래프의 허브

### 소스코드 분석

`ProviderScope`가 생성하는 `ProviderContainer`의 핵심을 살펴보자:

```dart
// flutter_riverpod/lib/src/core/provider_scope.dart
final class ProviderScopeState extends State<ProviderScope> {
  late final ProviderContainer container;

  @override
  void initState() {
    super.initState();
    final parent = _getParent();
    container = ProviderContainer(
      parent: parent,           // ← 부모 ProviderScope의 container
      overrides: widget.overrides,
      observers: widget.observers,
      retry: widget.retry,
      // ...
    );
  }

  ProviderContainer? _getParent() {
    final scope = context
        .getElementForInheritedWidgetOfExactType<_UncontrolledProviderScope>()
        ?.widget as _UncontrolledProviderScope?;
    return scope?.container;
  }

  @override
  Widget build(BuildContext context) {
    // _UncontrolledProviderScope: 단순히 container를 InheritedWidget으로 전달
    return UncontrolledProviderScope(
      container: container,
      child: widget.child,
    );
  }
}
```

#### 핵심 포인트

1. **`_getParent()`**: `getElementForInheritedWidgetOfExactType`로 부모 `ProviderContainer`를 찾는다. 
   **`dependOnInheritedWidgetOfExactType`가 아니다** — 즉 부모가 변해도 자식 `ProviderScope`는 리빌드되지 않는다.

2. **`_UncontrolledProviderScope`**: 가장 안쪽의 `InheritedWidget`

```dart
// 내부 InheritedWidget — container만 전달
final class _UncontrolledProviderScope extends InheritedWidget {
  final ProviderContainer container;

  @override
  bool updateShouldNotify(_UncontrolledProviderScope oldWidget) {
    return container != oldWidget.container;
    // ← Provider와 달리, container 참조가 바뀔 때만 true
  }
}
```

Provider의 `_InheritedProviderScope.updateShouldNotify`가 **항상 `false`**를 반환했던 것과 비교하라.
Riverpod에서는 `InheritedWidget`이 상태 전파가 아닌 **container 참조 전달**에만 사용되므로, 
일반적인 `!=` 비교면 충분하다.

### ProviderPointerManager — O(1) 상태 접근

`ProviderContainer` 내부에서 각 Provider의 상태(`ProviderElement`)를 관리하는 것이 `ProviderPointerManager`다:

```dart
// riverpod/lib/src/core/provider_container.dart

/// An object responsible for storing the a O(1) access to providers,
/// while also enabling the "scoping" of providers and ensuring all
/// ProviderContainers are in sync.
class ProviderPointerManager {
  // Family별 디렉토리
  final familyPointers = <Family, ProviderDirectory>{};
  // 독립 Provider별 디렉토리  
  final orphanPointers = ProviderDirectory._();
}
```

Provider는 `getElementForInheritedWidgetOfExactType`로 **O(1)** 조회를 달성했지만 (HashMap 기반),
Riverpod은 `ProviderPointerManager`가 **자체 HashMap**으로 O(1) 접근을 보장하면서도 
**scoping과 override**를 체계적으로 관리한다.

---

## 11.5 Ref vs WidgetRef — 소스코드 분석

Riverpod의 핵심 설계 혁신 중 하나는 **`Ref`와 `WidgetRef`의 분리**다.

### Ref: 프로바이더-to-프로바이더 상호작용

```dart
// riverpod/lib/src/core/ref.dart
sealed class Ref implements MutationTarget {
  // ※ MutationTarget: Riverpod 자체 인터페이스 (Flutter가 아님).
  //    Mutation.run()에 Ref/WidgetRef/Container 중 아무거나 전달할 수 있게 하는 공통 계약.
  ProviderElement<Object?, Object?> get _element;
  
  // 프로바이더 간 의존 — 값이 변하면 현재 프로바이더 전체를 invalidate
  StateT watch<StateT>(ProviderListenable<StateT> listenable) {
    _throwIfInvalidUsage();
    late ProviderSubscription<StateT> sub;
    sub = _element.listen<StateT>(
      listenable,
      (prev, value) => invalidateSelf(asReload: true),  // ← 핵심!
      onError: (err, stack) => invalidateSelf(asReload: true),
      onDependencyMayHaveChanged: _element._markDependencyMayHaveChanged,
    );
    return sub.readSafe().valueOrProviderException;
  }

  // 현재 값만 읽기 (의존성 등록 없음)
  StateT read<StateT>(ProviderListenable<StateT> listenable) {
    _throwIfInvalidUsage();
    return container.read(listenable);  // ← container에서 직접 읽기
  }
}
```

**`ref.watch()`의 핵심 메커니즘:**

1. `_element.listen(listenable, callback)` — 대상 프로바이더를 구독
2. 콜백이 `invalidateSelf(asReload: true)` — 대상이 변하면 **현재 프로바이더 자체를 무효화**
3. 무효화된 프로바이더는 다음 프레임에서 **build 함수가 처음부터 다시 실행**

이것은 Provider의 `context.watch`와 근본적으로 다르다:
- **Provider `context.watch`**: 위젯의 `build()` 메서드를 다시 호출 (위젯 리빌드)
- **Riverpod `ref.watch`**: 프로바이더의 build 함수를 **처음부터 다시 실행** (상태 재계산)

### WidgetRef: 위젯-to-프로바이더 상호작용

```dart
// flutter_riverpod/lib/src/core/consumer.dart
base class ConsumerStatefulElement extends StatefulElement 
    implements WidgetRef {
  
  late ProviderContainer container = ProviderScope.containerOf(this);
  var _dependencies = <ProviderListenable, ProviderSubscription>{};
  Map<ProviderListenable, ProviderSubscription>? _oldDependencies;

  @override
  StateT watch<StateT>(ProviderListenable<StateT> target) {
    _assertNotDisposed();
    return _dependencies
        .putIfAbsent(target, () {
          final oldDependency = _oldDependencies?.remove(target);
          if (oldDependency != null) return oldDependency;

          final sub = container.listen<StateT>(
            target,
            (_, _) => markNeedsBuild(),  // ← 위젯 리빌드
          );
          _applyTickerMode(sub);
          return sub;
        })
        .readSafe()
        .valueOrProviderException as StateT;
  }
}
```

**`WidgetRef.watch()`의 핵심 메커니즘:**

1. `container.listen(target, markNeedsBuild)` — 프로바이더 값 변경 시 위젯 리빌드
2. `_dependencies` Map으로 **구독 캐싱** — 같은 프로바이더를 매 빌드마다 재구독하지 않음
3. `_oldDependencies` 패턴으로 **이전 빌드의 구독 재활용**

#### 빌드 시 구독 관리 흐름

```dart
@override
Widget build() {
  try {
    _oldDependencies = _dependencies;  // ① 이전 구독 보존
    for (var i = 0; i < _listeners.length; i++) {
      _listeners[i].close();            // ② listen() 구독은 매번 재생성
    }
    _listeners.clear();
    _dependencies = {};                 // ③ 새 구독 Map 시작
    return super.build();               // ④ build() 실행 → watch() 호출들
  } finally {
    for (final dep in _oldDependencies!.values) {
      dep.close();                      // ⑤ 더 이상 watch하지 않는 구독 정리
    }
    _oldDependencies = null;
  }
}
```

이 패턴의 의미:
- **매 빌드마다** `watch()`로 구독하는 프로바이더 목록이 **동적으로 변할 수 있다**
- 이전 빌드에서 `watch`했지만 이번 빌드에서 `watch`하지 않은 프로바이더는 **자동 구독 해제**
- Provider에서는 `dependOnInheritedWidgetOfExactType`로 등록한 의존성이 **위젯이 unmount될 때까지 유지**되지만,
  Riverpod에서는 **매 빌드마다 재평가**된다

### 실행 흐름 추적 — 추상화를 한 꺼풀씩 벗기기

위 소스코드를 읽어도 "결국 무슨 일이 일어나는데?"가 불명확할 수 있다.
Riverpod은 추상화가 깊어서(평균 3~4단계) 이름만으로는 실제 동작을 추론하기 어렵다.
그래서 **사용자가 `ref.watch(otherProvider)`를 호출한 순간부터 화면이 업데이트되기까지**의 
전체 경로를 한 줄씩 추적해본다.

#### 시나리오: 프로바이더 A가 `ref.watch(providerB)`를 호출

```
사용자 코드:
  final a = Provider((ref) => ref.watch(providerB));
```

**▶ 1단계: Ref.watch() (ref.dart)**
```dart
StateT watch<StateT>(ProviderListenable<StateT> listenable) {
  sub = _element.listen<StateT>(
    listenable,
    (prev, value) => invalidateSelf(asReload: true),  // 콜백 등록
  );
  return sub.readSafe().valueOrProviderException;  // 현재 값 반환
}
```
→ `_element`는 현재 프로바이더 A의 `ProviderElement`
→ `listen()`으로 B에 대한 **구독(Subscription)**을 만듦
→ 콜백 내용: "B가 변하면, **나(A)를 무효화**해라"

**▶ 2단계: ProviderElement.listen() (element.dart)**
```dart
ProviderSubscription<ListenedStateT> listen<ListenedStateT>(
  ProviderListenable<ListenedStateT> listenable,
  void Function(ListenedStateT? previous, ListenedStateT value) listener, {
  ...
}) {
  final sub = listenable._addListener(this, listener, ...);  // 구독 객체 생성
  sub.impl._listenedElement.addDependentSubscription(sub.impl);  // B의 의존자 목록에 A 등록
  return sub;
}
```
→ `_addListener`: 내부적으로 `ProviderSubscriptionImpl`을 만들어 **B의 `dependents` 리스트에 추가**
→ 이제 B의 값이 변하면 B의 `_notifyListeners()`가 호출됨

**▶ 3단계: B의 값이 변했을 때 → _notifyListeners() (element.dart)**
```dart
void _notifyListeners(AsyncValue<ValueT> newState, AsyncValue<ValueT>? previousState, ...) {
  // ... updateShouldNotify 체크 후 ...
  final listeners = [...weakDependents, ...?dependents];
  for (var i = 0; i < listeners.length; i++) {
    container.runBinaryGuarded(
      listeners[i].providerSub._notifyData,  // ← 각 구독자의 콜백 실행
      previousState, newState,
    );
  }
}
```
→ `dependents`에는 1단계에서 등록한 A의 **구독**이 있음
→ A의 구독 콜백 = `invalidateSelf(asReload: true)`가 실행됨

**▶ 4단계: invalidateSelf() (element.dart)**
```dart
void invalidateSelf({required bool asReload}) {
  _mustRecomputeState = true;              // "다시 계산해야 함" 플래그
  runOnDispose();                           // 이전 상태 정리
  container.scheduler.scheduleProviderRefresh(this);  // 스케줄러에 재빌드 예약

  // 나에게 의존하는 프로바이더들(C, D, ...)에게도 전파
  visitChildren((element) {
    element._markDependencyMayHaveChanged();
  });
}
```
→ **핵심**: `scheduleProviderRefresh(this)` — A를 **스케줄러의 리프레시 큐**에 넣음
→ 스케줄러가 다음 마이크로태스크에서 A의 `flush()` → `_performRebuild()` → `create(ref)` 실행
→ 즉, **A의 build 함수가 처음부터 다시 실행**됨

**전체 흐름 요약:**
```
ref.watch(providerB)
  ↓
_element.listen(B, invalidateSelf)    ← B에 구독 등록
  ↓
[B의 값이 변함]
  ↓
B._notifyListeners()                  ← B가 모든 구독자에게 알림
  ↓
A의 콜백: invalidateSelf()            ← A 자신을 무효화
  ↓
scheduler.scheduleProviderRefresh(A)  ← 다음 마이크로태스크에 A 재빌드 예약
  ↓
A.flush() → _performRebuild() → create(ref)  ← A의 build 함수 재실행
  ↓
[A에 의존하는 C, D, ... 에게도 연쇄 전파]
```

---

#### 시나리오: 위젯이 `ref.watch(providerB)`를 호출

```
사용자 코드:
  class MyWidget extends ConsumerWidget {
    Widget build(BuildContext context, WidgetRef ref) {
      final value = ref.watch(providerB);
      return Text('$value');
    }
  }
```

**▶ 1단계: WidgetRef.watch() (consumer.dart)**
```dart
StateT watch<StateT>(ProviderListenable<StateT> target) {
  return _dependencies
      .putIfAbsent(target, () {
        // 이전 빌드에서 같은 프로바이더를 watch했다면 구독 재활용
        final oldDependency = _oldDependencies?.remove(target);
        if (oldDependency != null) return oldDependency;

        // 새 구독 생성
        final sub = container.listen<StateT>(
          target,
          (_, _) => markNeedsBuild(),  // 콜백: 위젯 리빌드
        );
        _applyTickerMode(sub);
        return sub;
      })
      .readSafe()
      .valueOrProviderException as StateT;
}
```
→ `container.listen()`으로 B에 대한 구독을 만듦
→ 콜백 내용: "B가 변하면, **이 위젯을 리빌드**해라" (`markNeedsBuild`)
→ `_dependencies` Map에 구독을 **캐싱** (같은 build에서 두 번 watch해도 구독 1개)

**▶ 2단계: B의 값이 변했을 때 → 위젯만 리빌드**
```
B._notifyListeners()
  ↓
위젯 구독의 콜백: markNeedsBuild()   ← Flutter Element API
  ↓
다음 프레임에서 위젯의 build() 재실행
  ↓
build() 안에서 다시 ref.watch(providerB)
  ↓
_dependencies에서 기존 구독 재활용 (새 구독 생성 ✘)
  ↓
readSafe()로 최신 값 반환
```

**핵심 차이점:**
- `Ref.watch`의 콜백 = **`invalidateSelf`** → 프로바이더의 build 함수 **전체를 다시 실행** (상태 재계산)
- `WidgetRef.watch`의 콜백 = **`markNeedsBuild`** → 위젯의 build 메서드만 다시 실행 (UI 갱신)

이 차이가 왜 중요한가:
```dart
// Ref.watch — 프로바이더의 build()가 재실행되면
// 기존 상태가 사라지고 완전히 새로 계산된다
final expensiveProvider = Provider((ref) {
  final data = ref.watch(configProvider);    // config 변경 시
  return heavyComputation(data);              // ← 이 전체가 다시 실행됨
});

// WidgetRef.watch — 위젯의 build()만 재실행되므로
// 프로바이더의 상태 자체는 그대로, 최신 값만 읽어온다
class MyPage extends ConsumerWidget {
  Widget build(BuildContext context, WidgetRef ref) {
    final result = ref.watch(expensiveProvider);  // 프로바이더 재계산 X
    return Text('$result');                        // UI만 갱신
  }
}
```

---

## 11.6 TickerMode 통합 — 보이지 않는 위젯의 최적화

`ConsumerStatefulElement`에는 독특한 최적화가 내장되어 있다:

```dart
void _updateTickerMode() {
  final isActive = _tickerModeNotifier!.value;
  if (isActive != _isActive) {
    _isActive = isActive;
    for (final sub in _dependencies.values) {
      if (isActive) {
        sub.resume();    // 위젯이 다시 보이면 구독 재개
      } else {
        sub.pause();     // 위젯이 안 보이면 구독 일시정지
      }
    }
  }
}
```

`TickerMode`는 Flutter가 위젯의 가시성을 추적하는 메커니즘이다.
예를 들어, `TabBarView`에서 현재 보이지 않는 탭의 위젯들은 `TickerMode.of(context) == false`가 된다.

Riverpod은 이를 활용하여:
- 보이지 않는 위젯의 프로바이더 구독을 **pause** → 불필요한 리빌드 방지
- 다시 보일 때 **resume** → 최신 값으로 갱신

이것은 Provider에는 없는 최적화다.

---

## 11.7 autoDispose와 keepAlive — 자동 메모리 관리

### Riverpod의 라이프사이클 후크

Provider의 `InheritedProvider`에서 `dispose`가 위젯 트리의 unmount에 의존했다면,
Riverpod은 **리스너 기반의 자동 해제**를 제공한다:

```dart
// ref.dart
KeepAliveLink keepAlive() {
  _throwIfInvalidUsage();
  final links = _keepAliveLinks ??= [];
  
  late KeepAliveLink link;
  link = KeepAliveLink._(() {
    if (links.remove(link)) {
      if (links.isEmpty) _element.mayNeedDispose();
    }
  });
  links.add(link);
  return link;
}
```

```dart
// 사용 예
final myProvider = Provider.autoDispose((ref) {
  final link = ref.keepAlive();
  
  // 5초 후 자동 해제 예약
  final timer = Timer(Duration(seconds: 5), link.close);
  ref.onDispose(timer.cancel);
  
  return expensiveComputation();
});
```

### 라이프사이클 후크 전체 그림

```
Provider 생성 ──→ ref.onDispose(() { ... })  ──→ cleanup 등록
     │
     ├─ ref.onAddListener(() { ... })      ──→ 새 리스너 추가 시
     ├─ ref.onRemoveListener(() { ... })   ──→ 리스너 제거 시
     ├─ ref.onCancel(() { ... })           ──→ 마지막 리스너 제거 시
     └─ ref.onResume(() { ... })           ──→ 다시 리스닝 시작 시

autoDispose 흐름:
  모든 리스너 제거 → onCancel → keepAliveLink 없으면 → dispose → onDispose
```

Provider에서는 이런 세밀한 라이프사이클 제어가 불가능했다.
`ChangeNotifierProvider`의 `dispose`는 위젯이 트리에서 제거될 때만 호출된다.

---

## 11.8 순환 의존성 감지

Riverpod은 프로바이더 간 **순환 의존성을 컴파일 타임에 감지**할 수 없지만,
**런타임에 BFS 탐색으로 감지**한다:

```dart
// ref.dart
void _debugAssertCanDependOn(ProviderListenableOrFamily listenable) {
  // ...
  final queue = Queue<ProviderElement>();
  _element.visitChildren(queue.add);

  while (queue.isNotEmpty) {
    final current = queue.removeFirst();
    current.visitChildren(queue.add);

    if (current.origin == dependency) {
      final dependencyLoop = _buildDependencyLoop(current, current.origin);
      throw CircularDependencyError._(dependencyLoop);
    }
  }
}
```

현재 프로바이더의 **자식들을 BFS로 탐색**하면서, 그 중에 의존하려는 대상이 있으면
순환 의존성으로 판단하고 `CircularDependencyError`를 던진다.

Provider에서는 이런 감지 메커니즘이 없었다. 순환 의존이 생기면 무한 루프나 스택 오버플로우로 이어졌다.

---

## 11.9 Provider.of vs container.read — 상태 접근 메커니즘 비교

### Provider: 위젯 트리 탐색

```dart
// provider/lib/src/provider.dart
static T of<T>(BuildContext context, {bool listen = true}) {
  final inheritedElement = _inheritedElementOf<T>(context);
  if (listen) {
    context.dependOnInheritedWidgetOfExactType<_InheritedProviderScope<T?>>();
  }
  final value = inheritedElement?.value;
  return value as T;
}
```

→ `BuildContext` 필수, `InheritedWidget` 트리 탐색

### Riverpod: Container 직접 접근

```dart
// flutter_riverpod/lib/src/core/consumer.dart
@override
StateT read<StateT>(ProviderListenable<StateT> provider) {
  _assertNotDisposed();
  return ProviderScope.containerOf(this, listen: false).read(provider);
}
```

→ `ProviderScope.containerOf()`로 container를 한 번 얻은 후, container의 HashMap에서 O(1) 조회

### containerOf()의 listen 분기

```dart
// flutter_riverpod/lib/src/core/provider_scope.dart
static ProviderContainer containerOf(BuildContext context, {bool listen = true}) {
  _UncontrolledProviderScope? scope;
  if (listen) {
    scope = context.dependOnInheritedWidgetOfExactType<_UncontrolledProviderScope>();
  } else {
    scope = context.getElementForInheritedWidgetOfExactType<_UncontrolledProviderScope>()
        ?.widget as _UncontrolledProviderScope?;
  }
  return scope!.container;
}
```

이것은 Ch10에서 본 Provider의 `Provider.of(listen:)` 패턴과 **정확히 같은 분기**다!
차이는 **무엇을 얻느냐**이다:
- Provider: `_InheritedProviderScope<T>`의 **값 자체**
- Riverpod: `_UncontrolledProviderScope`의 **`ProviderContainer` 참조**

Riverpod은 InheritedWidget에서 container만 얻고, 이후 모든 상태 접근은 container 내부에서 처리한다.

---

## 11.10 Family와 Modifier — 컴파일 타임 안전성

### Provider의 한계

```dart
// Provider: 같은 타입 2개를 구분할 수 없다
Provider<String>(create: (_) => 'Hello'),
Provider<String>(create: (_) => 'World'),
// → Provider.of<String>(context)는 가장 가까운 것만 반환
```

### Riverpod의 해결

```dart
// Riverpod: 각 provider가 고유한 전역 변수
final helloProvider = Provider((_) => 'Hello');
final worldProvider = Provider((_) => 'World');
// → ref.watch(helloProvider), ref.watch(worldProvider) 정확히 구분

// Family: 매개변수화된 프로바이더
final userProvider = FutureProvider.family<User, int>((ref, userId) async {
  return fetchUser(userId);
});
// → ref.watch(userProvider(42))  // userId=42인 User
// → ref.watch(userProvider(99))  // userId=99인 User — 다른 인스턴스
```

Family는 내부적으로 `ProviderPointerManager.familyPointers`에서 관리된다:

```dart
class ProviderPointerManager {
  final familyPointers = <Family, ProviderDirectory>{};
  // Family + argument 조합으로 고유한 ProviderElement 생성
}
```

---

## 11.11 Notifier 패턴 — ChangeNotifier를 넘어서

### Provider의 ChangeNotifier

```dart
// ChangeNotifier: mutable state + notifyListeners()
class CounterModel extends ChangeNotifier {
  int _count = 0;
  int get count => _count;
  
  void increment() {
    _count++;
    notifyListeners();  // 수동 호출 필수
  }
}
```

### Riverpod의 Notifier

```dart
// Notifier: immutable state assignment
class Counter extends Notifier<int> {
  @override
  int build() => 0;  // 초기 상태
  
  void increment() {
    state = state + 1;  // state setter가 자동으로 notifyListeners
  }
}

final counterProvider = NotifierProvider<Counter, int>(Counter.new);
```

Riverpod의 `Notifier`는:
1. **`build()` 메서드**가 초기 상태와 의존성을 선언적으로 정의
2. **`state` setter**가 `ref.notifyListeners()`를 자동 호출 — `notifyListeners()` 깜빡 방지
3. **`ref` 접근** — Notifier 안에서 다른 프로바이더를 `ref.watch`/`ref.read` 가능

---

## 11.12 Notifier의 자유도 — Provider vs Riverpod 설계 철학

"왜 Provider의 `ChangeNotifier`는 `notifyListeners()`를 수동으로 호출하고,
Riverpod의 Notifier는 `state = ...`만으로 자동 알림이 되는가?"

이 차이는 단순한 편의성이 아니라, **프레임워크의 설계 철학이 반영된 의도적인 결정**이다.

### Provider — ChangeNotifier: "개발자에게 완전한 제어권"

```dart
// flutter/lib/src/foundation/change_notifier.dart

mixin class ChangeNotifier implements Listenable {
  List<VoidCallback?> _listeners = _emptyListeners;
  int _count = 0;

  // ↓ 개발자가 직접 호출해야 함 — @protected
  @protected
  @visibleForTesting
  void notifyListeners() {
    if (_count == 0) return;

    _notificationCallStackDepth++;
    final int end = _count;
    for (var i = 0; i < end; i++) {
      try {
        _listeners[i]?.call();          // 등록된 콜백을 순회하며 호출
      } catch (exception, stack) {
        FlutterError.reportError(...);
      }
    }
    _notificationCallStackDepth--;
  }
}
```

**특징**: `notifyListeners()`는 `@protected` — 상속한 클래스 내부에서만 호출 가능하며,
**개발자가 언제 호출할지 결정해야 한다**.

```dart
// Provider에서의 일반적인 사용 패턴
class CounterModel extends ChangeNotifier {
  int _count = 0;
  int get count => _count;

  void increment() {
    _count++;
    notifyListeners();    // ← ⚠️ 이것을 빼먹으면? UI가 갱신되지 않음!
  }

  void batchUpdate(List<int> values) {
    for (final v in values) {
      _count += v;
    }
    notifyListeners();    // ← 배치의 마지막에만 호출 → 효율적이지만 실수 가능
  }
}
```

### Riverpod — AnyNotifier: "상태 변경 = 알림, 예외 없음"

```dart
// riverpod/lib/src/core/provider/notifier_provider.dart

abstract class AnyNotifier<StateT, ValueT> {
  $ClassProviderElement? _element;

  @visibleForTesting
  @protected
  set state(StateT newState) {
    final ref = $ref;
    ref._throwIfInvalidUsage();
    ref._element.setValueFromState(newState);   // ← 자동 알림!
  }
}
```

`set state`의 내부 호출 체인:

```
notifier.state = newValue
  → ref._element.setValueFromState(newState)     // element.dart:465
    → set value(AsyncValue<ValueT> value)         // element.dart:452
      → _value = value;
      → if (_didBuild) _notifyListeners(result, previousResult);   // ← 자동!
```

```dart
// riverpod/lib/src/core/element.dart (ProviderElement)

set value(AsyncValue<ValueT> value) {
  final previousResult = this.value;
  final result = _value = value;

  if (_didBuild) {
    _notifyListeners(result, previousResult);    // ← 개발자 관여 없이 자동 실행
  }
}
```

**개발자가 `state = newValue`를 쓰는 순간, 알림이 자동으로 이루어진다.**
콜백 호출을 빼먹을 수 없다.

### 자유도 비교표

| 항목 | Provider (ChangeNotifier) | Riverpod (AnyNotifier) |
|------|--------------------------|----------------------|
| **알림 메커니즘** | `notifyListeners()` 수동 호출 | `set state` 내부에서 자동 |
| **알림 시점 제어** | ✅ 개발자가 결정 | ❌ 상태 변경 즉시 (제어 불가) |
| **배치 업데이트** | ✅ 여러 변경 후 한 번만 알림 가능 | ❌ 각 `state =` 마다 알림 발생 |
| **알림 누락 위험** | ⚠️ `notifyListeners()` 호출 잊으면 버그 | ✅ 알림 누락 불가능 |
| **가변 객체 직접 변경** | ⚠️ 리스트 직접 수정 후 notify 가능 | ❌ 새 객체로 교체해야 상태 변경 감지 |
| **상태 타입 제약** | 없음 (어떤 타입이든 가능) | `==` 비교 기반, 불변 데이터 권장 |

### 높은 자유도의 양날의 검 — 실제 예시

```dart
// ──── Provider: 자유도가 높아서 좋은 경우 ────
class FormModel extends ChangeNotifier {
  String _name = '';
  String _email = '';
  bool _hasUnsavedChanges = false;

  void updateName(String name) {
    _name = name;
    _hasUnsavedChanges = true;
    // 아직 notifyListeners() 호출 안 함 → UI 갱신 안 함
  }

  void updateEmail(String email) {
    _email = email;
    _hasUnsavedChanges = true;
    // 아직 notifyListeners() 호출 안 함
  }

  void commitChanges() {
    notifyListeners();   // ← 모든 변경을 한 번에 반영!
    _hasUnsavedChanges = false;
  }
}

// ──── Provider: 자유도가 높아서 위험한 경우 ────
class TodoModel extends ChangeNotifier {
  final List<Todo> _todos = [];

  void addTodo(Todo todo) {
    _todos.add(todo);
    // ⚠️ notifyListeners()를 빼먹었다! → UI가 갱신되지 않는 버그
  }

  void removeTodo(int index) {
    _todos.removeAt(index);
    notifyListeners();    // 여기선 기억했지만...
  }
}
```

```dart
// ──── Riverpod: 낮은 자유도가 좋은 경우 ────
@riverpod
class TodoList extends _$TodoList {
  @override
  List<Todo> build() => [];

  void addTodo(Todo todo) {
    state = [...state, todo];
    // 자동으로 UI 갱신! 빼먹을 수 없다.
  }
}

// ──── Riverpod: 낮은 자유도가 불편한 경우 ────
@riverpod
class FormState extends _$FormState {
  @override
  ({String name, String email}) build() => (name: '', email: '');

  void updateName(String name) {
    state = (name: name, email: state.email);
    // ← 즉시 UI 갱신 → 중간 상태가 노출됨
    // 배치 업데이트가 어렵다
  }

  void updateAll(String name, String email) {
    // 한 번에 교체해야 배치 효과
    state = (name: name, email: email);
  }
}
```

### 가변 객체(Mutable Object) 처리의 차이

이 차이가 가장 극명하게 드러나는 지점:

```dart
// Provider: 가변 객체 직접 변경 가능 (위험하지만 편리)
class CartModel extends ChangeNotifier {
  final List<Item> items = [];

  void addItem(Item item) {
    items.add(item);         // ← 리스트를 직접 변경
    notifyListeners();       // ← 수동 알림
  }
}

// Riverpod: 불변성 강제 (안전하지만 객체 재생성 비용)
@riverpod
class Cart extends _$Cart {
  @override
  List<Item> build() => [];

  void addItem(Item item) {
    state = [...state, item];  // ← 새 리스트 생성! (기존 리스트를 변경하면 감지 안 됨)
  }
}
```

**왜 Riverpod은 불변성을 강제하는가?**

Riverpod의 `ProviderElement.value` setter를 다시 보면:

```dart
set value(AsyncValue<ValueT> value) {
  final previousResult = this.value;
  final result = _value = value;
  if (_didBuild) {
    _notifyListeners(result, previousResult);   // previous와 next를 비교!
  }
}
```

`_notifyListeners`는 이전 값과 새 값을 콜백에 전달한다.
가변 객체를 직접 수정하면 **previous와 next가 같은 레퍼런스**를 가리켜
`updateShouldNotify`의 `==` 비교가 무의미해진다.

Riverpod의 `updateShouldNotify` 기본 구현:

```dart
// element.dart
bool updateShouldNotify(StateT previous, StateT next) {
  return !identical(previous, next);  // ← 레퍼런스 비교!
}
```

**가변 객체를 직접 변경하면 `identical(previous, next) == true`가 되어 알림이 발생하지 않는다.**
이것이 Riverpod이 불변 데이터 패턴을 강제하는 기술적 근거다.

### 설계 철학 정리

```
Provider (ChangeNotifier)    ──────────────────────────    Riverpod (AnyNotifier)
                             자 유 도 스 펙 트 럼
높은 자유도 ◀━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━▶ 낮은 자유도

"언제 알릴지 개발자가 결정"                        "상태 변경이 곧 알림"
"어떤 타입이든 상태로 사용"                        "불변 데이터+새 객체 교체"
"실수 가능성 → 유연성"                            "실수 불가능 → 예측 가능성"
```

| 선택 기준 | Provider가 유리 | Riverpod이 유리 |
|----------|----------------|----------------|
| 초보자 | ❌ 알림 빼먹는 실수 빈번 | ✅ 실수할 여지 없음 |
| 배치 업데이트 | ✅ 여러 변경 후 한번만 | ❌ 중간 렌더링 가능 |
| 가변 데이터 (List, Map) | ✅ 직접 조작 후 notify | ❌ 매번 복사 필요 |
| 대규모 팀 | ❌ 규칙 위반 추적 어려움 | ✅ 프레임워크 레벨 강제 |
| 디버깅 | ❌ 업데이트 누락 찾기 어려움 | ✅ 상태 변경 추적 용이 |
| 테스트 | ❌ 알림 호출 여부 별도 검증 필요 | ✅ 상태만 검증하면 됨 |

**결론**: Provider는 "최대한의 자유를 주되, 올바르게 사용하는 책임을 개발자에게 맡긴다" — 이것은 Flutter 코어 팀의 철학과 일맥상통한다 (`ChangeNotifier`는 Flutter 프레임워크 자체의 기반 클래스).
Riverpod은 "올바른 사용법만 가능하게 만들어, 실수 자체를 불가능하게 한다" — Remi Rousselet의 반복되는 설계 원칙으로, **Pit of Success(성공의 구덩이)** 패턴이라 불린다.

---

## 11.13 전체 비교 테이블

| 특성 | Provider | Riverpod |
|------|----------|----------|
| **상태 저장** | InheritedWidget (위젯 트리) | ProviderContainer (독립 그래프) |
| **상태 접근** | `Provider.of<T>(context)` | `ref.watch(provider)` |
| **식별 방식** | 제네릭 타입 `T` | 전역 변수 참조 |
| **같은 타입 다수** | ❌ 불가 | ✅ 각각 고유 변수 |
| **매개변수화** | 불가 (직접 구현 필요) | ✅ Family |
| **컴파일 타임 안전** | ❌ 런타임 ProviderNotFoundException | ✅ 존재하지 않는 provider는 컴파일 에러 |
| **BuildContext 필수** | ✅ 항상 필요 | ❌ Ref는 BuildContext 불필요 |
| **autoDispose** | ❌ 위젯 생명주기에 의존 | ✅ 리스너 기반 자동 해제 |
| **순환 의존 감지** | ❌ | ✅ BFS 런타임 감지 |
| **구독 일시정지** | ❌ | ✅ TickerMode 통합 |
| **테스트** | testWidgets 필수 | ProviderContainer로 순수 Dart 테스트 |
| **updateShouldNotify** | 항상 false (push 모델) | container != oldContainer |
| **위젯 리빌드 트리거** | `markNeedsBuild()` via InheritedElement | `markNeedsBuild()` via ProviderSubscription |
| **프로바이더 리빌드** | 지원하지 않음 | `invalidateSelf()` → build() 재실행 |

---

## 11.14 마이그레이션 가이드: Provider → Riverpod

### 단계별 전환

**1단계: ProviderScope 추가**

```dart
// Before (Provider)
void main() {
  runApp(
    MultiProvider(
      providers: [...],
      child: MyApp(),
    ),
  );
}

// After (Riverpod)
void main() {
  runApp(
    ProviderScope(    // ← ProviderContainer를 자동 관리
      child: MyApp(),
    ),
  );
}
```

**2단계: ChangeNotifier → Notifier**

```dart
// Before
class TodoList extends ChangeNotifier {
  List<Todo> _todos = [];
  List<Todo> get todos => _todos;
  
  void add(Todo todo) {
    _todos = [..._todos, todo];
    notifyListeners();
  }
}

final todoListProvider = ChangeNotifierProvider((_) => TodoList());

// After
class TodoList extends Notifier<List<Todo>> {
  @override
  List<Todo> build() => [];
  
  void add(Todo todo) {
    state = [...state, todo];  // 자동 알림
  }
}

final todoListProvider = NotifierProvider<TodoList, List<Todo>>(TodoList.new);
```

**3단계: Consumer 위젯 전환**

```dart
// Before (Provider)
class TodoPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final todos = context.watch<TodoList>().todos;
    return ListView(children: todos.map(TodoTile.new).toList());
  }
}

// After (Riverpod)
class TodoPage extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final todos = ref.watch(todoListProvider);
    return ListView(children: todos.map(TodoTile.new).toList());
  }
}
```

**4단계: ProxyProvider → ref.watch**

```dart
// Before (Provider): 프로바이더 간 의존성
ProxyProvider<AuthModel, UserRepository>(
  update: (_, auth, __) => UserRepository(auth.token),
)

// After (Riverpod): 선언적 의존성
final userRepositoryProvider = Provider((ref) {
  final token = ref.watch(authProvider).token;
  return UserRepository(token);
  // token이 변경되면 자동으로 재생성
});
```

---

## 11.15 언제 어떤 것을 선택할까?

### Provider를 유지하는 경우
- 기존 대규모 프로젝트에서 안정적으로 동작 중
- 팀원 대부분이 Provider에 익숙하고, 단순한 상태 관리만 필요
- InheritedWidget의 구조를 깊이 이해하고 있어 직접 확장 가능

### Riverpod을 선택하는 경우
- **같은 타입의 프로바이더 다수** 필요 (예: 다중 API 클라이언트)
- **프로바이더 간 복잡한 의존성** 그래프
- **autoDispose**가 필요한 복잡한 메모리 관리
- **테스트 용이성** — `ProviderContainer`로 위젯 없이 순수 단위 테스트
- **컴파일 타임 안전성** — 존재하지 않는 프로바이더 참조 시 컴파일 에러
- riverpod_generator를 활용한 **코드 생성** 기반 개발

---

## 11.16 면접 Q&A

### Q1: Provider와 Riverpod의 근본적인 아키텍처 차이는?

**정답:**
Provider는 `InheritedWidget` 기반으로 **상태 자체가 위젯 트리에 존재**한다. 
`_InheritedProviderScope`라는 InheritedWidget의 Element(`_InheritedProviderScopeElement`)가 
상태 저장, 의존성 추적, 알림 전파를 모두 담당한다.

반면 Riverpod은 **위젯 트리와 독립된 `ProviderContainer`**에 상태가 존재한다.
`ProviderContainer` 내부의 `ProviderPointerManager`가 각 Provider마다 `ProviderElement`를 관리하며,
위젯 트리의 `ProviderScope`는 `_UncontrolledProviderScope`(InheritedWidget)를 통해 
이 container의 **참조만 전달**하는 역할을 한다.

이 차이로 인해:
- Riverpod은 `BuildContext` 없이 상태 접근 가능 (`ProviderContainer.read()`)
- 같은 타입의 Provider를 전역 변수로 구분 가능 (타입 기반 조회의 한계 해결)
- 위젯 없이 순수 Dart 테스트 가능

### Q2: Riverpod의 `ref.watch`와 Provider의 `context.watch`는 내부적으로 어떻게 다른가?

**정답:**
**Provider의 `context.watch<T>()`:**
```dart
T watch<T>(BuildContext context) => Provider.of<T>(context);
// → context.dependOnInheritedWidgetOfExactType<_InheritedProviderScope<T>>()
// → Element가 InheritedElement의 _dependents에 등록됨
// → 값 변경 시 notifyClients() → didChangeDependencies() → markNeedsBuild()
```

**Riverpod의 `ref.watch()` (프로바이더 간):**
```dart
StateT watch<StateT>(ProviderListenable<StateT> listenable) {
  sub = _element.listen(listenable,
    (prev, value) => invalidateSelf(asReload: true),
  );
  return sub.readSafe().valueOrProviderException;
}
```
→ 값 변경 시 `invalidateSelf()` → **프로바이더의 build()가 처음부터 재실행**

**Riverpod의 `WidgetRef.watch()` (위젯에서):**
```dart
StateT watch<StateT>(ProviderListenable<StateT> target) {
  return _dependencies.putIfAbsent(target, () {
    final sub = container.listen(target, (_, _) => markNeedsBuild());
    return sub;
  }).readSafe().valueOrProviderException;
}
```
→ `container.listen(target, markNeedsBuild)` → 위젯 리빌드

핵심 차이: `ref.watch`는 **프로바이더를 무효화(invalidate)**하고, `WidgetRef.watch`는 **위젯을 리빌드**한다.

### Q3: Riverpod의 `ConsumerStatefulElement.build()`에서 구독 관리 패턴을 설명하라.

**정답:**
매 빌드마다 이전 빌드의 구독을 `_oldDependencies`에 보존하고, 새 빌드에서 `watch()`가 호출될 때 
`putIfAbsent`로 같은 프로바이더를 다시 구독하면 **기존 구독을 재활용**한다.

```
build() 시작
  ① _oldDependencies = _dependencies   // 이전 구독 보존
  ② _dependencies = {}                  // 새 Map 시작
  ③ super.build() 실행                  // watch() 호출들
     → putIfAbsent에서 _oldDependencies 확인 → 있으면 재활용
  ④ finally: _oldDependencies 남은 것들 close()  // 더 이상 안 쓰는 구독 정리
```

이 패턴 덕분에:
- 조건부 `watch`가 가능 (빌드마다 다른 프로바이더를 watch 가능)
- 더 이상 watch하지 않는 프로바이더는 자동 구독 해제
- 기존 구독은 재활용하여 불필요한 재구독 방지

Provider에서는 `dependOnInheritedWidgetOfExactType`로 등록한 의존성이 
**Element가 unmount될 때까지 유지**되므로, 이런 동적 구독 관리가 불가능하다.

### Q4: `autoDispose`와 `keepAlive`의 내부 메커니즘은?

**정답:**
`autoDispose` 프로바이더는 **리스너가 0개가 되면 dispose**된다.
`keepAlive()`는 `KeepAliveLink`를 생성하여 이를 방지한다:

```dart
KeepAliveLink keepAlive() {
  final links = _keepAliveLinks ??= [];
  late KeepAliveLink link;
  link = KeepAliveLink._(() {
    if (links.remove(link)) {
      if (links.isEmpty) _element.mayNeedDispose();
    }
  });
  links.add(link);
  return link;
}
```

모든 `KeepAliveLink`가 `close()`되어야만 `_element.mayNeedDispose()`가 호출된다.
이를 활용한 "캐시 타이머" 패턴:

```dart
final dataProvider = Provider.autoDispose((ref) {
  final link = ref.keepAlive();
  final timer = Timer(Duration(minutes: 5), link.close);
  ref.onDispose(timer.cancel);
  return fetchData();
});
```

→ 리스너가 없어도 5분간 캐시 유지 → 5분 후 자동 해제 → 재접근 시 fresh fetch

### Q5: Riverpod에서 `InheritedWidget`의 역할은 정확히 무엇인가?

**정답:**
Riverpod에서 `InheritedWidget`(`_UncontrolledProviderScope`)은 **`ProviderContainer` 참조를 
위젯 트리에 전달**하는 데만 사용된다.

```dart
final class _UncontrolledProviderScope extends InheritedWidget {
  final ProviderContainer container;
  @override
  bool updateShouldNotify(_UncontrolledProviderScope oldWidget) {
    return container != oldWidget.container;
  }
}
```

Provider에서는 `_InheritedProviderScope.updateShouldNotify`가 **항상 false**를 반환하며
push 모델로 알림을 전파했다. 반면 Riverpod에서는:

1. `updateShouldNotify`는 `container != oldWidget.container` — container가 교체될 때만 전파
2. 상태 변경 알림은 `ProviderContainer.listen()` → `ProviderSubscription` → `markNeedsBuild()`로 전달
3. InheritedWidget의 `notifyClients()` / `_dependents` 메커니즘은 **사용하지 않음**

즉, Provider는 InheritedWidget의 상태 전파 메커니즘을 커스터마이징하여 사용했고,
Riverpod은 InheritedWidget을 **오직 container 참조 공유를 위한 서비스 로케이터**로만 축소했다.

---

## 다음 장 예고

Ch12에서는 **Riverpod Generator — 빌드 타임 코드 생성**을 분석한다.
`@riverpod` 어노테이션 하나가 어떻게 타입-세이프한 프로바이더 코드로 변환되는지,
`build_runner` → `RiverpodGenerator` → 5개 템플릿의 파이프라인을 소스코드 수준에서 추적한다.
이 장에서 배운 `ProviderElement`, `Ref`, `create()`가 Generator의 출력과 어떻게 연결되는지 확인할 것이다.

> Ch13에서는 Bloc 패턴과 반응형 프로그래밍, Stream 기반 이벤트-상태 변환을 다룬다.
