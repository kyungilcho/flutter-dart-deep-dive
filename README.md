# 🔬 Flutter/Dart Deep Dive — 소스코드 기반 심층 분석

> 전환자 → 중급자 → 중급자+ 까지, **실제 프레임워크 소스코드**를 분석하며 배우는 Flutter/Dart

[![Flutter](https://img.shields.io/badge/Flutter-3.41.1-02569B?logo=flutter)](https://flutter.dev)
[![Dart](https://img.shields.io/badge/Dart-3.8-0175C2?logo=dart)](https://dart.dev)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

---

## 📌 이 프로젝트는 무엇인가?

Flutter/Dart의 공식 문서에서 다루지 않는 **내부 동작 원리**를 소스코드 레벨에서 분석한 이론서이다.

- 🔍 **실제 소스코드 인용** — 추측이 아닌 코드 기반 설명
- 🎯 **면접 Q&A** — 각 챕터마다 시니어급 면접 질문/답변 포함
- 📊 **단계적 학습** — 기본 → 중급 → 심화 순서로 구성

---

## 📖 목차

### Part 1. Dart 언어 기초~심화

| Ch | 제목 | 상태 |
|----|------|------|
| 01 | [Dart 타입 시스템 — Sound Null Safety의 내부](part1_dart/ch01_type_system.md) | ✅ |
| 02 | 컬렉션과 제네릭 — 공변성과 타입 추론 | 📝 |
| 03 | 클래스 심화 — mixin, extension, sealed class | 📝 |
| 04 | [비동기 프로그래밍 — Event Loop과 Zone](part1_dart/ch04_async_programming.md) | ✅ |

### Part 2. Flutter 프레임워크 핵심 원리

| Ch | 제목 | 상태 |
|----|------|------|
| 05 | [위젯 기초 — 불변성과 3-트리 아키텍처](part2_flutter/ch05_widget_fundamentals.md) | ✅ |
| 06 | [State와 생명주기 — Element의 상태 머신](part2_flutter/ch06_state_lifecycle.md) | ✅ |
| 07 | [BuildContext와 InheritedWidget — O(1) 의존성 주입](part2_flutter/ch07_buildcontext_inherited.md) | ✅ |
| 08 | [레이아웃 시스템과 RenderObject — 직접 다루기](part2_flutter/ch08_layout_system.md) | ✅ |
| 09 | [렌더링 파이프라인 — VSync에서 GPU까지](part2_flutter/ch09_rendering_pipeline.md) | ✅ |

### Part 3. 상태 관리의 모든 것

| Ch | 제목 | 상태 |
|----|------|------|
| 10 | [상태 관리, 왜 어려운가](part3_state_management/ch10_state_problem.md) | ✅ |
| 11 | [Provider → Riverpod — 무엇이 달라졌는가](part3_state_management/ch11_provider_vs_riverpod.md) | ✅ |
| 12 | [Riverpod Generator — 빌드 타임 코드 생성](part3_state_management/ch12_riverpod_generator.md) | ✅ |

### Part 4. 아키텍처와 설계 패턴

| Ch | 제목 | 상태 |
|----|------|------|
| 13 | 클린 아키텍처 — 레이어 분리의 실전 | 📝 |
| 14 | 의존성 주입 — GetIt과 Injectable | 📝 |
| 15 | 네비게이션 — GoRouter의 내부 | 📝 |
| 16 | 테스트 — 유닛/위젯/통합 테스트 | 📝 |

### Part 5. 실전 심화

| Ch | 제목 | 상태 |
|----|------|------|
| 17 | 네트워크 — Dio 인터셉터 체인 | 📝 |
| 18 | 로컬 저장소 — Hive와 SQLite | 📝 |
| 19 | 코드 생성 — build_runner와 Freezed | 📝 |

### Part 6. 엔진 심화

| Ch | 제목 | 상태 |
|----|------|------|
| 20 | Impeller 렌더링 엔진 | 📝 |
| 21 | Platform Channel과 FFI | 📝 |
| 22 | Flutter Rust Bridge | 📝 |

> ✅ = 작성 완료 &nbsp;&nbsp; 📝 = 작성 예정

---

## 📁 소스코드 분석 대상

이 프로젝트에서 분석하는 소스코드 목록이다. 로컬에서 참고하려면 별도로 클론이 필요하다.

| 저장소 | 버전/태그 | 분석 대상 |
|--------|-----------|-----------|
| `flutter/flutter` | 3.41.1 | 프레임워크 핵심 (`packages/flutter/`) |
| `flutter/engine` | latest | Impeller 렌더링 엔진 (`impeller/`) |
| `dart-lang/sdk` | latest | Dart 언어 내부 (`sdk/lib/`) |
| `rrousselGit/riverpod` | latest | 상태관리 (`packages/riverpod/`) |
| `felangel/bloc` | latest | Bloc 패턴 (`packages/bloc/`) |
| `rrousselGit/provider` | latest | Provider (`lib/`) |
| `flutter/packages` | latest | GoRouter (`packages/go_router/`) |
| `cfug/dio` | latest | HTTP 클라이언트 (`dio/lib/src/`) |

---

## 📜 License

MIT License — 자유롭게 참고하고 활용할 수 있다.
