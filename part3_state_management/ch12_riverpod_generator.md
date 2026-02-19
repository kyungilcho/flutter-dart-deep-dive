# Chapter 12: Riverpod Generator — 빌드 타임 코드 생성

> **"개발자가 작성하는 코드는 의도를 선언하고, 기계가 생성하는 코드는 구현을 담당한다."**

Ch11에서 Riverpod의 **런타임 동작** — `Ref.watch`, `ProviderElement`, `ProviderContainer`의 실행 흐름을 추적했다.
이번 챕터에서는 관점을 바꾼다: **빌드 타임에 코드가 어떻게 생성되는가**.

`@riverpod` 어노테이션 하나로 수십 줄의 타입-세이프한 프로바이더 코드가 자동 생성되는 과정을,
실제 소스코드를 기반으로 한 단계씩 추적한다.

---

## 12.1 왜 코드 생성인가 — 수동 Riverpod의 남은 한계

Ch11에서 배운 수동(manual) Riverpod은 이미 강력하지만, 몇 가지 반복적인 고통이 남아있다:

### 수동 방식의 보일러플레이트

```dart
// 수동 방식: 카운터 프로바이더 하나를 만들기 위한 코드
class Counter extends Notifier<int> {
  @override
  int build() => 0;
  
  void increment() => state++;
}

// ↓ 이 부분이 항상 필요하고, 타입을 직접 맞춰야 한다
final counterProvider = NotifierProvider<Counter, int>(Counter.new);
```

### Family를 사용하면 어떻게 될까?

```dart
// 수동 방식: 파라미터를 받는 Family 프로바이더
class TodoList extends FamilyNotifier<List<Todo>, String> {
  @override
  List<Todo> build(String userId) => [];
  // ...
}

// ↓ 제네릭 파라미터를 3개나 수동으로 맞춰야 한다
final todoListProvider = NotifierProvider.family<TodoList, List<Todo>, String>(
  TodoList.new,
);

// 사용
ref.watch(todoListProvider('user-123'));
```

### 문제점 정리

| 문제 | 설명 |
|------|------|
| **타입 중복 선언** | `Notifier<int>`에서 이미 `int`를 선언했는데, `NotifierProvider<Counter, int>`에서 또 반복 |
| **Family 타입 폭발** | 파라미터가 2개 이상이면 Record나 별도 클래스를 만들어야 함 |
| **이름 규칙** | `Counter` → `counterProvider`의 네이밍을 수동으로 맞춰야 함 |
| **autoDispose 선택** | `AutoDisposeNotifier` vs `Notifier` 타입을 직접 골라야 함 |

**Riverpod Generator는 이 모든 반복을 `@riverpod` 어노테이션 하나로 해결한다.**

---

## 12.2 전체 파이프라인 개요

`@riverpod` 어노테이션이 `.g.dart` 파일로 변환되기까지의 흐름:

```
┌─────────────────────────────────────────────────────────────┐
│                    개발자가 작성하는 코드                        │
│                                                             │
│  @riverpod                                                  │
│  String greeting(Ref ref) => 'Hello';                       │
│                                                             │
│  part 'greeting.g.dart';   // ← 생성될 파일 선언               │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              dart run build_runner build                     │
│                                                             │
│  1. build_runner가 build.yaml 설정을 읽음                      │
│  2. riverpod_generator의 builder를 찾아 실행                   │
│  3. Dart Analyzer가 소스를 AST(추상 구문 트리)로 파싱             │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│             riverpod_generator 내부 처리                      │
│                                                             │
│  4. RiverpodGenerator.generateForUnit()                     │
│  5. _RiverpodGeneratorVisitor가 각 선언문을 방문               │
│  6. FunctionalProvider / ClassBasedProvider 판별              │
│  7. 5개 템플릿으로 코드 생성:                                    │
│     - ProviderVariableTemplate  (전역 변수)                   │
│     - ProviderTemplate          (Provider 클래스)             │
│     - HashFnTemplate            (해시 함수)                   │
│     - FamilyTemplate            (Family 클래스)               │
│     - NotifierTemplate          (Notifier 베이스 클래스)       │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                   greeting.g.dart 출력                       │
│                                                             │
│  // GENERATED CODE - DO NOT MODIFY BY HAND                  │
│  part of 'greeting.dart';                                   │
│                                                             │
│  @ProviderFor(greeting)                                     │
│  final greetingProvider = GreetingProvider._();              │
│                                                             │
│  final class GreetingProvider extends $FunctionalProvider... │
│  String _$greetingHash() => r'abc123...';                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 12.3 builder.dart — 진입점 분석

모든 것은 이 14줄 파일에서 시작된다:

```dart
// riverpod_generator/lib/builder.dart

import 'package:build/build.dart';
import 'package:source_gen/source_gen.dart';

import 'src/json_generator.dart';
import 'src/riverpod_generator.dart';

/// Builds generators for `build_runner` to run
Builder riverpodBuilder(BuilderOptions options) {
  return SharedPartBuilder([
    RiverpodGenerator(options.config),   // ← 핵심: 프로바이더 코드 생성
    JsonGenerator(),                      // ← JSON 직렬화 관련 코드 생성
  ], 'riverpod');
}
```

### 핵심 개념

- **`Builder`**: `build_runner`가 인식하는 코드 생성기의 인터페이스
- **`SharedPartBuilder`**: `part of` 파일을 생성하는 빌더. 여러 Generator의 출력을 하나의 `.g.dart`에 합침
- **`'riverpod'`**: 이 문자열이 파일명의 일부가 됨 → `xxx.g.dart`
- **`BuilderOptions`**: `build.yaml`에서 읽어온 설정 (프로바이더 이름 접미/접두사 등)

### build.yaml와의 연결

```yaml
# build.yaml
targets:
  $default:
    builders:
      riverpod_generator:
        options:
          # 프로바이더 이름에 접미사/접두사 추가 가능
          provider_name_prefix: ""
          provider_name_suffix: "Provider"  # 기본값
```

이 설정은 `BuilderOptions.config`로 전달되어 `BuildYamlOptions`에 파싱된다.

---

## 12.4 @riverpod 어노테이션 — 무엇을 선언하는가

### 어노테이션 클래스의 소스코드

```dart
// riverpod_annotation/lib/src/riverpod_annotation.dart

@Target({TargetKind.classType, TargetKind.function})  // ← 클래스와 함수에만 붙일 수 있음
@sealed
final class Riverpod {
  const Riverpod({
    this.keepAlive = false,      // true면 autoDispose 비활성화
    this.dependencies,           // 의존하는 프로바이더 목록 (scoping용)
    this.retry,                  // 실패 시 재시도 로직
    this.name,                   // 생성될 프로바이더 변수명 직접 지정
  });

  final String? name;
  final Duration? Function(int retryCount, Object error)? retry;
  final bool keepAlive;
  final List<Object>? dependencies;
}

/// 소문자 `@riverpod` — 기본값으로 생성
@Target({TargetKind.classType, TargetKind.function})
const riverpod = Riverpod();     // keepAlive: false (= 자동으로 autoDispose)
```

### 두 가지 사용 방식

```dart
// 1. 소문자 (기본값) — autoDispose 활성화, 의존성 없음
@riverpod
String greeting(Ref ref) => 'Hello';

// 2. 대문자 (커스터마이즈) — keepAlive, dependencies 등 설정 가능
@Riverpod(keepAlive: true, dependencies: [other])
String greeting(Ref ref) => 'Hello';
```

### @Target의 의미

`@Target({TargetKind.classType, TargetKind.function})`는 이 어노테이션이
**함수**와 **클래스**에만 붙을 수 있음을 Dart 분석기에 알려준다.
이것이 Generator가 두 가지 경로로 분기하는 근거:

| 대상 | Generator 분류 | 생성 결과 |
|------|---------------|----------|
| **함수** | `FunctionalProviderDeclaration` | `$FunctionalProvider` + `$Family` |
| **클래스** | `ClassBasedProviderDeclaration` | `$NotifierProvider` + `$Notifier` + `$Family` |

---

## 12.5 RiverpodGenerator — AST 방문자 패턴

### Generator의 핵심 구조

```dart
// riverpod_generator/lib/src/riverpod_generator.dart

@immutable
class RiverpodGenerator extends ParserGenerator<Riverpod> {
  // ① build.yaml 옵션을 받아 파싱
  RiverpodGenerator(Map<String, Object?> mapConfig)
    : options = BuildYamlOptions.fromMap(mapConfig);

  final BuildYamlOptions options;

  @override
  String generateForUnit(List<CompilationUnit> compilationUnits) {
    // ② 출력 버퍼 준비
    final buffer = AnalyzerBuffer.part(
      compilationUnits.first.declaredFragment!.element,
      header: '// ignore_for_file: type=lint, type=warning',
    );

    // ③ 에러 수집 → 생성 → 에러 보고
    final errors = <RiverpodAnalysisError>[];
    try {
      errorReporter = errors.add;
      _generate(compilationUnits, buffer);
    } finally {
      errorReporter = previousErrorReporter;
    }

    for (final error in errors) {
      throw RiverpodInvalidGenerationSourceError(
        error.message, astNode: error.targetNode,
      );
    }

    return buffer.toString();
  }
}
```

### _generate — 선언문 분류

```dart
void _generate(List<CompilationUnit> units, AnalyzerBuffer buffer) {
  final visitor = _RiverpodGeneratorVisitor(tmpBuffer, options);

  for (final member in units.expand((e) => e.declarations)) {
    final provider = member.provider;         // ← AST에서 provider 정보 추출

    switch (provider) {
      case ClassBasedProviderDeclaration():    // @riverpod class XXX extends _$XXX
        visitor.visitClassBasedProviderDeclaration(provider);
      case FunctionalProviderDeclaration():    // @riverpod String xxx(Ref ref)
        visitor.visitFunctionalProviderDeclaration(provider);
      default:
        continue;                             // @riverpod이 아닌 선언은 무시
    }
  }
}
```

### Visitor의 코드 생성 흐름

```dart
class _RiverpodGeneratorVisitor {
  final StringBuffer buffer;
  final BuildYamlOptions options;

  // ① 모든 프로바이더 공통 — 4개 템플릿 실행
  void visitGeneratorProviderDeclaration(GeneratorProviderDeclaration provider) {
    final allTransitiveDependencies = _computeAllTransitiveDependencies(provider);

    ProviderVariableTemplate(provider, options).run(buffer);   // 전역 변수
    ProviderTemplate(provider, options, ...).run(buffer);       // Provider 클래스
    HashFnTemplate(provider).run(buffer);                       // 해시 함수
    FamilyTemplate(provider, options, ...).run(buffer);         // Family 클래스
  }

  // ② 클래스 기반 → 추가로 NotifierTemplate 실행
  void visitClassBasedProviderDeclaration(ClassBasedProviderDeclaration provider) {
    visitGeneratorProviderDeclaration(provider);   // 공통 4개
    NotifierTemplate(provider).run(buffer);         // + Notifier 베이스 클래스
  }

  // ③ 함수 기반 → 공통 4개만
  void visitFunctionalProviderDeclaration(FunctionalProviderDeclaration provider) {
    visitGeneratorProviderDeclaration(provider);   // 공통 4개만
  }
}
```

```
                visitGeneratorProviderDeclaration (공통)
                ┌─────────────────────────────────────┐
                │ ProviderVariableTemplate            │
                │ ProviderTemplate                    │
                │ HashFnTemplate                      │
                │ FamilyTemplate                      │
                └──────────────────┬──────────────────┘
                                   │
            ┌──────────────────────┼──────────────────────┐
            │                      │                      │
   FunctionalProvider         ClassBasedProvider           │
   (추가 없음)                 + NotifierTemplate          │
```

---

## 12.6 5개 템플릿 — 각각 무엇을 생성하는가

소스코드의 실제 테스트 케이스를 기반으로 분석한다.

### 입력 코드 (sync.dart)

```dart
// 단순 함수 기반
@riverpod
String public(Ref ref) {
  return 'Hello world';
}

// 클래스 기반
@riverpod
class PublicClass extends _$PublicClass {
  @override
  String build() {
    return 'Hello world';
  }
}
```

### 출력 코드 분석 (sync.g.dart)

#### ① ProviderVariableTemplate — 전역 변수

```dart
// 생성 코드
@ProviderFor(public)                          // ← 원본 함수와의 연결 마커
final publicProvider = PublicProvider._();     // ← 전역에서 접근하는 프로바이더 인스턴스
```

**역할**: `ref.watch(publicProvider)`에서 사용하는 전역 변수를 생성한다.
Family가 아닌 경우 `Provider._()`, Family인 경우 `Family._()`를 생성한다.

```dart
// 소스: ProviderVariableTemplate.run()
void run(StringBuffer buffer) {
  buffer.writeln('@ProviderFor(${provider.name})');
  
  switch (provider) {
    case _ when provider.providerElement.isFamily:
      buffer.writeln('final $providerName = ${provider.familyTypeName}._();');
    case _:
      buffer.writeln('final $providerName = $providerType._();');
  }
}
```

#### ② ProviderTemplate — Provider 클래스

함수 기반과 클래스 기반에서 다른 코드를 생성한다:

**함수 기반 (`@riverpod String public(Ref ref)`):**

```dart
final class PublicProvider
    extends $FunctionalProvider<String, String, String>   // ← 타입 자동 추론
    with $Provider<String> {                              // ← 동기 Provider mixin
  
  PublicProvider._(): super(
    from: null,           // Family가 아니므로 null
    argument: null,       // 파라미터 없음
    retry: null,          // 재시도 미설정
    name: r'publicProvider',
    isAutoDispose: true,  // @riverpod는 기본 autoDispose
    dependencies: null,
    $allTransitiveDependencies: null,
  );

  @override
  String debugGetCreateSourceHash() => _$publicHash();

  @$internal
  @override
  $ProviderElement<String> $createElement($ProviderPointer pointer)
      => $ProviderElement(pointer);

  @override
  String create(Ref ref) {
    return public(ref);    // ← 원본 함수를 그대로 호출!
  }
}
```

**클래스 기반 (`@riverpod class PublicClass`):**

```dart
final class PublicClassProvider
    extends $NotifierProvider<PublicClass, String> {  // ← Notifier 타입 자동 매핑
  
  PublicClassProvider._(): super(
    from: null, argument: null, retry: null,
    name: r'publicClassProvider',
    isAutoDispose: true,
    dependencies: null,
    $allTransitiveDependencies: null,
  );

  @$internal
  @override
  PublicClass create() => PublicClass();  // ← Notifier 인스턴스 생성
}
```

**핵심 차이:**
- 함수 기반: `create(Ref ref)` → 원본 함수 호출
- 클래스 기반: `create()` → Notifier 인스턴스 생성

#### ③ HashFnTemplate — 소스 해시

```dart
String _$publicHash() => r'5413a75a31e3e1e4135ee1bf8f57f4a9d52f4e62';
```

**용도**: 소스코드가 변경되었는지 감지하는 해시.
Hot Reload 시 Provider를 재초기화할지 결정하는 데 사용된다.
코드가 변경되면 해시가 바뀌고, Riverpod은 이를 감지해 해당 Provider를 무효화한다.

#### ④ FamilyTemplate — Family 클래스 (파라미터가 있을 때만)

파라미터가 있는 프로바이더에서만 생성된다:

```dart
// 입력
@riverpod
String family(Ref ref, int first, {String? second, required double third}) {
  return '...';
}
```

```dart
// 생성된 Family 클래스
final class FamilyFamily extends $Family
    with $FunctionalFamilyOverride<String, (int, {String? second, double third})> {
  
  FamilyFamily._(): super(
    name: r'familyProvider',
    dependencies: null,
    isAutoDispose: true,
  );

  // ← 파라미터가 함수 시그니처가 된다!
  FamilyProvider call(int first, {String? second, required double third})
    => FamilyProvider._(
      argument: (first, second: second, third: third),  // ← Record로 변환
      from: this,
    );
}
```

**핵심**: 수동 방식에서는 Family 파라미터를 하나의 타입으로 제한했지만,
Generator는 **함수 파라미터 → Dart 3 Record** 변환으로 여러 파라미터를 자연스럽게 지원한다.

```dart
// 사용법
ref.watch(familyProvider(42, second: 'hello', third: 3.14));
//        ↑ familyProvider는 FamilyFamily 인스턴스
//        .call()이 호출되어 FamilyProvider 인스턴스 반환
```

#### ⑤ NotifierTemplate — Notifier 베이스 클래스 (클래스 기반만)

NotifierTemplate은 5개 템플릿 중 **클래스 기반에서만 추가 실행**되는 유일한 템플릿이다.
개발자가 `extends _$XXX`로 상속하는 **추상 베이스 클래스**를 생성하며,
Riverpod 런타임과 개발자 코드를 연결하는 핵심 접착제 역할을 한다.

##### 템플릿 소스코드 — 무엇을 결정하는가

```dart
// riverpod_generator/lib/src/templates/notifier.dart

class NotifierTemplate extends Template {
  final ClassBasedProviderDeclaration provider;

  @override
  void run(StringBuffer buffer) {
    // ① 클래스 이름 결정
    final notifierBaseName = provider.isPersisted
        ? '_\$${provider.name.public}Base'    // 특수: persisted notifier
        : '_\$${provider.name.public}';        // 일반: _$MyNotifier

    // ② 부모 클래스를 createdType에 따라 분기
    final baseClass = switch (provider.providerElement.createdType) {
      SupportedCreatedType.future =>           // build()가 Future<T> 반환
        '\$AsyncNotifier<${provider.valueTypeNode}>',
      SupportedCreatedType.stream =>           // build()가 Stream<T> 반환
        '\$StreamNotifier<${provider.valueTypeNode}>',
      SupportedCreatedType.value =>            // build()가 T 반환 (동기)
        '\$Notifier<${provider.valueTypeNode}>',
    };

    // ③ 파라미터 → getter 변환 (아래에서 상세 분석)
    // ④ build() 시그니처 생성
    // ⑤ runBuild() 생성
  }
}
```

##### 세 가지 부모 클래스

| `build()` 반환 타입 | 생성되는 부모 클래스 | 설명 |
|---------------------|---------------------|------|
| `String` (동기) | `$Notifier<String>` | 상태를 즉시 반환 |
| `Future<String>` | `$AsyncNotifier<String>` | 비동기 데이터 로드 |
| `Stream<String>` | `$StreamNotifier<String>` | 스트림 구독 |

Generator는 개발자의 `build()` 반환 타입만 보고 자동으로 올바른 부모를 선정한다.
수동 방식에서는 `Notifier`, `AsyncNotifier`, `StreamNotifier` 중 하나를 직접 골라야 했다.

##### 단순 예시 — 파라미터 없는 Notifier

```dart
// 입력
@riverpod
class PublicClass extends _$PublicClass {
  @override
  String build() => 'Hello world';
}
```

```dart
// 생성된 _$PublicClass
abstract class _$PublicClass extends $Notifier<String> {
  // 파라미터 없음 → _$args, getter 없음

  String build();           // ← 추상 메서드 (개발자가 @override로 구현)

  @$mustCallSuper
  @override
  void runBuild() {
    final ref = this.ref as $Ref<String, String>;
    final element = ref.element as $ClassProviderElement<
      AnyNotifier<String, String>, String, Object?, Object?
    >;
    element.handleCreate(ref, build);   // ← build 메서드 참조를 직접 전달
  }
}
```

##### Family 예시 — 파라미터 → getter 변환의 핵심

파라미터가 있으면 Generator의 진짜 능력이 드러난다:

```dart
// 입력
@riverpod
class FamilyClass extends _$FamilyClass {
  @override
  String build(int first, {String? second, required double third,
               bool fourth = true, List<String>? fifth}) {
    return '$first, $second, $third, $fourth, $fifth';
  }
}
```

```dart
// 생성된 _$FamilyClass — 파라미터가 getter로 변환된다!
abstract class _$FamilyClass extends $Notifier<String> {

  // ▼ ① Record로 저장된 argument를 lazy하게 캐스팅
  late final _$args = ref.$arg as (int, {
    String? second,
    double third,
    bool fourth,
    List<String>? fifth,
  });

  // ▼ ② 각 파라미터가 getter로 노출 — Notifier 내부에서 this.first로 접근 가능!
  int get first => _$args.$1;              // positional → Record.$1
  String? get second => _$args.second;      // named → Record.fieldName
  double get third => _$args.third;
  bool get fourth => _$args.fourth;
  List<String>? get fifth => _$args.fifth;

  // ▼ ③ build()는 모든 파라미터를 받는 추상 메서드
  String build(int first, {
    String? second,
    required double third,
    bool fourth = true,
    List<String>? fifth,
  });

  // ▼ ④ runBuild()가 _$args에서 꺼낸 값으로 build()를 호출
  @$mustCallSuper
  @override
  void runBuild() {
    final ref = this.ref as $Ref<String, String>;
    final element = ref.element as $ClassProviderElement<
      AnyNotifier<String, String>, String, Object?, Object?
    >;
    element.handleCreate(
      ref,
      () => build(         // ← 파라미터 없는 경우와 다름: 클로저로 감쌈
        _$args.$1,          // first
        second: _$args.second,
        third: _$args.third,
        fourth: _$args.fourth,
        fifth: _$args.fifth,
      ),
    );
  }
}
```

##### 파라미터 → getter 변환의 소스코드

이 변환은 `NotifierTemplate.run()` 내부에서 일어난다:

```dart
// notifier.dart — 파라미터를 getter로 변환하는 핵심 로직

final parametersAsFields = provider.parameters.map((p) {
  return '${p.typeDisplayString} get ${p.name!.lexeme} => ${
    switch (provider.parameters) {
      [_] => r'_$args;',                              // 파라미터 1개: _$args 그 자체
      _ => '_\$args.${p.isPositional                   // 파라미터 2개+: Record 필드 접근
            ? '\$${++paramOffset}'                     //   positional → .$1, .$2...
            : p.name!.lexeme                           //   named → .fieldName
          };',
    }
  }';
}).join();
```

| 파라미터 | 생성되는 getter | Record 접근 | 설명 |
|---------|----------------|-------------|------|
| `int first` (positional) | `int get first => _$args.$1;` | Record.$1 | 위치 기반 접근 |
| `String? second` (named) | `String? get second => _$args.second;` | Record.second | 이름 기반 접근 |
| 파라미터 1개일 때 | `int get id => _$args;` | 타입 그대로 | Record 없이 직접 접근 |

##### runBuild()의 분기 — 파라미터 유무

```dart
// notifier.dart — runBuild() 내 build 호출 방식 결정

final buildVarUsage = switch (provider.parameters) {
  []       => 'build',                        // 파라미터 없음 → 메서드 참조
  [_, ...] => '() => build($paramsPassThrough)', // 파라미터 있음 → 클로저로 감쌈
};

// 생성:
// element.handleCreate(ref, build);                    ← 파라미터 없을 때
// element.handleCreate(ref, () => build(_$args.$1));   ← 파라미터 있을 때
```

**왜 클로저인가?** `handleCreate`는 `T Function()` 타입의 콜백을 받는다.
파라미터가 없으면 `build` 자체가 `() => T`이므로 직접 전달할 수 있지만,
파라미터가 있으면 `(int, String) => T`이므로 `() => build(args...)`로 감싸야 한다.

##### Ch11과의 연결점 — element.handleCreate

`runBuild()`에서 호출하는 `element.handleCreate`는 Ch11에서 분석한
`$ClassProviderElement`의 메서드다. 이 호출이 Notifier의 **라이프사이클 시작점**이 된다:

```
개발자 코드                     Generator 생성 코드              Riverpod 런타임 (Ch11)
─────────                     ──────────────                  ─────────────────
@riverpod                     _$FamilyClass                   $ClassProviderElement
class FamilyClass              ├── _$args (Record)             ├── handleCreate()
  extends _$FamilyClass        ├── getters (first, second...)  │     → build() 호출
  │                            ├── build() (abstract)          │     → state 초기화
  ├── build(int first, ...)    └── runBuild()                  ├── mount()
  └── addItem()                    → element.handleCreate()    └── listenSelf()
```

##### 추상 build 패턴 — 왜 이렇게 설계했는가

```dart
abstract class _$PublicClass extends $Notifier<String> {
  String build();    // 추상!
}

class PublicClass extends _$PublicClass {
  @override
  String build() => 'Hello world';   // 개발자가 구현
}
```

이 패턴의 장점:
1. **컴파일 타임 안전성**: `build()`를 구현하지 않으면 컴파일 에러
2. **시그니처 강제**: Family 파라미터가 `build()`의 시그니처에 포함되므로 타입 불일치 방지
3. **런타임 분리**: 개발자는 `build()`만 신경쓰고, `runBuild()`의 존재를 몰라도 됨

---

## 12.7 Family 파라미터 변환 — Record의 활용

Generator의 가장 영리한 부분을 소스코드로 살펴보자.

### 파라미터 개수에 따른 분기

```dart
// riverpod_generator.dart — ProviderNames 확장

String get argumentRecordType {
  switch (parameters) {
    case [_]:                    // 파라미터 1개: 타입 그대로
      return parameters.first.typeDisplayString;
    case []:                     // 파라미터 0개: Never
      return 'Never';
    case [...]:                  // 파라미터 2개 이상: Record로 변환
      return '(${buildParamDefinitionQuery(parameters, asRecord: true)})';
  }
}
```

### 실제 변환 예시

| 원본 코드 | argumentRecordType | 설명 |
|-----------|-------------------|------|
| `fn(Ref ref)` | `Never` | 파라미터 없음 → Family 아님 |
| `fn(Ref ref, int id)` | `int` | 1개 → 타입 그대로 |
| `fn(Ref ref, int a, {String? b})` | `(int, {String? b})` | 2개+ → Record |

**왜 Record인가?**

Dart 3의 Record는 **불변이고, ==와 hashCode가 자동 구현**되므로
Provider 캐싱의 키로 사용하기에 완벽하다.
수동 방식에서는 이 키를 개발자가 직접 만들어야 했다.

---

## 12.8 keepAlive와 autoDispose — 어노테이션에서 코드로

### 어노테이션의 결정

```dart
@riverpod                          // keepAlive: false (기본값)
String greeting(Ref ref) => '...';

@Riverpod(keepAlive: true)         // keepAlive: true
String greeting(Ref ref) => '...';
```

### 코드에서의 반영

```dart
// ProviderTemplate._writeConstructor()
PublicProvider._(): super(
  // ...
  isAutoDispose: ${!provider.annotation.element.keepAlive},  // ← keepAlive의 역
);
```

**수동 방식과의 비교:**

```dart
// 수동 — 다른 부모 클래스를 선택해야 함
class Counter extends Notifier<int> { ... }           // keepAlive
class Counter extends AutoDisposeNotifier<int> { ... } // autoDispose

// Generator — 어노테이션 파라미터 하나로 결정
@Riverpod(keepAlive: true)   // → isAutoDispose: false
@riverpod                    // → isAutoDispose: true (기본)
```

Generator는 **타입 계층 선택의 복잡함을 어노테이션 하나로 추상화**했다.

---

## 12.9 dependencies — 스코핑과 전이적 의존성

### dependencies가 코드에 미치는 영향

```dart
@Riverpod(dependencies: [other])
String dependent(Ref ref) {
  return ref.watch(otherProvider);
}
```

Generator는 이 정보를 두 가지로 변환한다:

```dart
final class DependentProvider extends $FunctionalProvider<...> {
  DependentProvider._(): super(
    // ① 직접 의존성
    dependencies: <ProviderOrFamily>[otherProvider],
    
    // ② 전이적 의존성 (other가 의존하는 것까지 포함)
    $allTransitiveDependencies: <ProviderOrFamily>[
      DependentProvider.$allTransitiveDependencies0,  // otherProvider
      DependentProvider.$allTransitiveDependencies1,  // otherProvider의 의존성
    ],
  );
  
  // 전이적 의존성을 정적 필드로 저장
  static final $allTransitiveDependencies0 = otherProvider;
  static final $allTransitiveDependencies1 = 
      OtherProvider.$allTransitiveDependencies0;
}
```

### 전이적 의존성 계산 — 재귀적 탐색

```dart
// _RiverpodGeneratorVisitor._computeAllTransitiveDependencies()

List<String>? _computeAllTransitiveDependencies(
  GeneratorProviderDeclaration provider,
) {
  final dependencies = provider.annotation.dependencyList?.values;
  if (dependencies == null) return null;

  final allTransitiveDependencies = <String>[];

  // 재귀적으로 모든 전이적 의존성을 수집
  Iterable<GeneratorProviderDeclarationElement>
  computeAllTransitiveDependencies(
    GeneratorProviderDeclarationElement provider,
  ) sync* {
    final deps = provider.annotation.dependencies;
    if (deps == null) return;

    final uniqueDependencies = <GeneratorProviderDeclarationElement>{};
    for (final transitiveDependency in deps) {
      if (!uniqueDependencies.add(transitiveDependency)) continue;
      yield transitiveDependency;
      yield* computeAllTransitiveDependencies(transitiveDependency);
    }
  }
  // ...
}
```

이 코드가 **빌드 타임에** 의존성 그래프를 탐색해서, 런타임에는 미리 계산된 결과를 사용한다.
린타임 성능에 영향을 주지 않으면서 스코핑의 정확성을 보장하는 설계다.

---

## 12.10 수동 vs Generator — 1:1 소스코드 대조

### 시나리오 1: 단순 프로바이더

```dart
// ────── 수동 방식 ──────
final greetingProvider = Provider<String>((ref) {
  return 'Hello world';
});

// ────── Generator 방식 ──────
@riverpod
String greeting(Ref ref) {
  return 'Hello world';
}

// ────── 생성되는 코드 ──────
@ProviderFor(greeting)
final greetingProvider = GreetingProvider._();

final class GreetingProvider extends $FunctionalProvider<String, String, String>
    with $Provider<String> {
  GreetingProvider._(): super(
    from: null, argument: null, retry: null,
    name: r'greetingProvider',
    isAutoDispose: true,
    dependencies: null,
    $allTransitiveDependencies: null,
  );
  
  @override
  String create(Ref ref) {
    return greeting(ref);   // ← 원본 함수 호출
  }
}

String _$greetingHash() => r'...';
```

### 시나리오 2: Notifier + Family

```dart
// ────── 수동 방식 ──────
class TodoList extends FamilyNotifier<List<Todo>, String> {
  @override
  List<Todo> build(String userId) => fetchTodos(userId);
  void addTodo(Todo todo) => state = [...state, todo];
}

final todoListProvider = NotifierProvider.family<TodoList, List<Todo>, String>(
  TodoList.new,
);

// 사용: ref.watch(todoListProvider('user-123'))

// ────── Generator 방식 ──────
@riverpod
class TodoList extends _$TodoList {
  @override
  List<Todo> build(String userId) => fetchTodos(userId);
  void addTodo(Todo todo) => state = [...state, todo];
}

// 사용: ref.watch(todoListProvider('user-123'))  ← 동일!

// ────── 생성되는 코드 (4개 요소) ──────

// 1. 전역 변수 (Family 인스턴스)
@ProviderFor(TodoList)
final todoListProvider = TodoListFamily._();

// 2. Provider 클래스
final class TodoListProvider
    extends $NotifierProvider<TodoList, List<Todo>> {
  TodoListProvider._({
    required TodoListFamily super.from,
    required String super.argument,
  }): super(...);
  
  @$internal @override
  TodoList create() => TodoList();
}

// 3. Family 클래스
final class TodoListFamily extends $Family
    with $ClassFamilyOverride<TodoList, List<Todo>, List<Todo>, List<Todo>, String> {
    
  TodoListProvider call(String userId)
    => TodoListProvider._(argument: userId, from: this);
}

// 4. Notifier 베이스 클래스
abstract class _$TodoList extends $Notifier<List<Todo>> {
  late final _$args = ref.$arg as String;
  String get userId => _$args;          // ← 파라미터가 getter로 제공됨!
  
  List<Todo> build(String userId);
  
  @$mustCallSuper @override
  void runBuild() {
    final element = ref.element as $ClassProviderElement<...>;
    element.handleCreate(ref, () => build(_$args));
  }
}
```

**주목**: 생성된 `_$TodoList`에서 `userId`가 **getter로 자동 제공**된다.
Notifier 내부에서 `this.userId`로 파라미터에 접근할 수 있다.

---

## 12.11 create()에서 원본 코드로 — 연결 지점

Ch11에서 추적한 Riverpod의 런타임 흐름과 Generator가 어떻게 연결되는지:

### 함수 기반

```
ref.watch(greetingProvider)
  → ProviderElement.mount()
    → GreetingProvider.create(Ref ref)    // ← Generator가 생성한 메서드
      → greeting(ref)                     // ← 개발자가 작성한 함수!
```

### 클래스 기반

```
ref.watch(todoListProvider('user-123'))
  → ProviderElement.mount()
    → TodoListProvider.create()           // ← Generator가 생성
      → TodoList()                        // Notifier 인스턴스 생성
    → notifier.runBuild()                 // ← Generator가 생성한 _$TodoList.runBuild()
      → element.handleCreate(ref, () => build(_$args))
        → build('user-123')              // ← 개발자가 작성한 build 메서드!
```

Generator가 생성하는 코드는 "개발자 코드와 Riverpod 런타임 사이의 접착제"다.

---

> **💡 Note**: Notifier의 자유도(자동 알림 vs 수동 알림)에 대한 설계 철학 비교는
> [11.12 Notifier의 자유도 — Provider vs Riverpod 설계 철학](ch11_provider_vs_riverpod.md#1112-notifier의-자유도--provider-vs-riverpod-설계-철학) 참조.

---

## 12.12 Generator의 설계 철학

### 왜 매크로가 아닌 코드 생성인가?

Dart는 현재 [매크로(Macros)](https://dart.dev/language/macros) 기능을 실험 중이다.
매크로가 안정화되면 `build_runner` 없이도 코드 생성이 가능해질 것이다.

하지만 현재 Riverpod이 코드 생성을 선택한 이유:

| 기준 | 매크로 | build_runner |
|------|--------|-------------|
| 안정성 | 실험적 | 프로덕션 레벨 |
| 생태계 | 아직 빈약 | freezed, json_serializable 등 성숙 |
| 디버깅 | `.g.dart` 없음 | `.g.dart`를 직접 읽을 수 있음 |
| 빌드 속도 | (이론적으로) 빠름 | 별도 빌드 단계 필요 |

### "선언적" 코드의 힘

Generator의 진짜 가치는 코드량 감소가 아니라 **의도의 명확성**이다:

```dart
// 개발자의 의도: "userId를 받아서 Todo 목록을 관리하는 상태"
@riverpod
class TodoList extends _$TodoList {
  @override
  List<Todo> build(String userId) => fetchTodos(userId);
}

// 구현의 세부사항: Provider 클래스, Family, 타입 매핑, 해시...
// → 전부 Generator가 처리
```

---

## 12.13 비교 테이블

| 항목 | 수동 Riverpod | Riverpod Generator |
|------|-------------|-------------------|
| **프로바이더 선언** | `Provider<T>((ref) => ...)` | `@riverpod T fn(Ref ref)` |
| **Notifier 선언** | `NotifierProvider<N, T>(N.new)` | `@riverpod class N extends _$N` |
| **타입 추론** | 수동 (3~4개 제네릭) | 자동 (함수 반환 타입에서 추론) |
| **Family 파라미터** | 1개 타입만 가능 | 여러 개, named 파라미터 가능 |
| **autoDispose** | 별도 타입 선택 | `@riverpod` (기본) vs `@Riverpod(keepAlive: true)` |
| **빌드 단계** | 불필요 | `dart run build_runner build` 필요 |
| **디버깅** | 직접 작성한 코드 | `.g.dart` 확인 가능 |
| **의존성 추적** | 없음 | `dependencies` + 전이적 의존성 자동 계산 |
| **Hot Reload 감지** | 없음 | 해시 함수로 변경 감지 |

---

## 12.14 실전 워크플로우

### 프로젝트 설정

```yaml
# pubspec.yaml
dependencies:
  riverpod_annotation: ^2.0.0
  riverpod: ^2.0.0

dev_dependencies:
  riverpod_generator: ^2.0.0
  build_runner: ^2.0.0
```

### 코드 생성 명령

```bash
# 한 번 생성
dart run build_runner build

# 파일 변경 감시 (개발 중)
dart run build_runner watch

# 충돌 시 기존 파일 삭제 후 재생성
dart run build_runner build --delete-conflicting-outputs
```

### 파일 구조

```
lib/
├── providers/
│   ├── auth_provider.dart          // 개발자가 작성
│   ├── auth_provider.g.dart        // Generator가 생성
│   ├── todo_provider.dart          // 개발자가 작성
│   └── todo_provider.g.dart        // Generator가 생성
```

**중요**: `.g.dart` 파일을 **직접 수정하면 안 된다**. `build_runner`를 다시 실행하면 덮어씌워진다.

---

## 12.15 면접 Q&A

### Q1: `@riverpod`와 `@Riverpod(keepAlive: true)`의 차이는?

**A**: 둘 다 같은 `Riverpod` 클래스를 사용한다. 소문자 `@riverpod`는 `const Riverpod()`로 정의된 상수로,
`keepAlive: false`(기본값) → `isAutoDispose: true`가 된다.
대문자 `@Riverpod(keepAlive: true)`는 구독자가 없어도 상태를 유지한다.
Generator는 이 값을 읽어 생성되는 Provider의 `isAutoDispose` 파라미터에 반영한다.

### Q2: Family의 파라미터가 Record로 변환되는 이유는?

**A**: Dart 3의 Record는 **값 기반 동등성(==)과 hashCode가 자동 제공**된다.
Provider의 캐싱은 argument를 키로 사용하는데, Record를 쓰면 별도의
`==`/`hashCode` 구현 없이도 `(userId: 'abc', page: 1)` 같은
복합 키를 안전하게 비교할 수 있다. 단일 파라미터는 Record 오버헤드가
불필요하므로 타입 그대로 사용한다.

### Q3: `.g.dart`에 생성되는 해시 함수의 용도는?

**A**: `_$xxxHash()` 함수는 소스코드의 내용을 기반으로 생성된 SHA-1 해시다.
Riverpod은 Hot Reload 시 이 해시를 비교해서, 프로바이더의 소스코드가 변경되었으면
해당 프로바이더를 무효화(invalidate)하고 다시 빌드한다.
이 메커니즘 덕분에 개발 중 코드를 수정하면 관련 상태만 정확하게 갱신된다.

### Q4: Generator와 수동 Riverpod을 섞어서 사용할 수 있는가?

**A**: 가능하다. 둘 다 같은 `ProviderContainer`와 `Ref`를 사용한다.
Generator가 생성하는 `$FunctionalProvider`나 `$NotifierProvider`는
수동으로 작성하는 `Provider`나 `NotifierProvider`와 같은 부모 클래스를 공유한다.
다만 일관성을 위해 한 프로젝트에서는 한 방식으로 통일하는 것이 권장된다.
