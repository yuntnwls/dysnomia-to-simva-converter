# Dysnomia → SIMVA 스크립트 통합 분석 보고서 (Master)

> **최종 수정일**: 2026-02-26  
> **본 문서는 Dysnomia(C#) 환경에서 SIMVA(Python) 환경으로의 성공적인 전환을 위한 구조 분석, API 매핑, 전략 비교 및 기술적 허들 해결책을 총망라한 마스터 보고서입니다.**

---

## 1. 프로젝트 구조 및 파일 분석

### 1.1 Dysnomia 디렉토리 및 가변 구조 분석

Dysnomia 프로젝트는 고정된 인프라 폴더(`_common`)와 프로젝트에 따라 가변적인 테스트케이스 폴더 구조를 가집니다.

#### [패턴 A: 표준 고정 구조]
주로 `Testcases_Common` 폴더를 통해 헬퍼 메서드가 체계적으로 분리된 형태입니다.
```
DysnomiaScript/
├── _common/                       # 공용 인프라 (고정, 프로젝트 공통)
│   ├── Functions/                 # 확장 메서드 정의 (10개 파일: Set, Get, Wait, Is, Turns, Keeps 등)
│   └── Predefined/
│       ├── Quantities/            # 신호 상수/값 정의 (CommonQuantities.cs 등)
│       └── Variables/             # 신호 변수 선언 (변환의 핵심 키)
├── Testcases_Common/              # 공용 헬퍼 메서드 (특정 샘플에만 존재)
│   ├── Methods/                   # 공통 동작 함수 (11개 파일: Methods_BDC.cs 등)
│   └── Check_Output_Signal/       # 출력 검증 함수
└── Testcases/                     # 실제 TC 파일 (구조 불고정)
    └── Domain/                    # (옵션) 도메인별 분류
        └── Feature.cs
```

#### [패턴 B: 평탄화 구조]
`Testcases_Common` 없이 모든 TC와 헬퍼가 `Testcases` 폴더 내에 혼재하는 형태입니다.
```
DysnomiaScript/
├── _common/                       # (동일)
└── Testcases/                     # TC + 헬퍼 함수 전부 혼재
    ├── Feature_A.cs
    ├── Feature_B.cs
    ├── Methods_Feature_A.cs       # 헬퍼가 TC와 같은 폴더에 위치
    └── CommonMethods.cs
```

#### [구조 가변성에 따른 변환기 대응 전략]
| 유형 | 분석 포인트 | 대응 전략 |
|------|--------|-----------|
| **폴더 구조** | TC 파일 위치가 매번 다름 | 파일 탐색 시 `[Testcase]` 어트리뷰트로 기능을 식별하여 폴더 무관하게 수집 |
| **헬퍼 위치** | 별도 폴더 혹은 TC 폴더 내 위치 | 메서드 호출 시 심볼 탐색(Symbol Resolving)을 통해 메서드 정의를 추적하여 참조 |
| **partial class** | 동일 클래스가 여러 파일에 분산 구현 | 변환 시 동일 네임스페이스/클래스명을 가진 모든 `.cs` 파일을 수집하여 병합 처리 (Mixin 전략) |

### 1.2 SIMVA 디렉토리 구조
```
SimvaScript/
├── sils_test.py                     # TestSuite 실행 파일 (핵심 실행 단위)
├── run.bat                          # 실행 스크립트
├── testcases/                       # 핵심 TC 및 공통 함수 폴더
│   ├── functions/                   # precondition, cleanup, method, outputcheck 등 레이어 분리
│   ├── config.py / quantity.py      # 설정 및 상수 관리
│   └── sils_testcase_*.py           # 실제 변환된 TC 모듈
└── references/
    ├── signals.py                   # 전체 신호 변수 풀 (ECU별 클래스 구분)
    └── profiles.py                  # ECU 프로파일
```

---

## 2. 테스트케이스(TC) 파일 구조 상세

### 2.1 TC 클래스 및 메서드 구조 비교

| 구분 | Dysnomia (C#) | SIMVA (Python) |
|:---|:---|:---|
| **기본 단위** | `public partial class Testcases` 내 메서드 | 모듈 내 독립 함수 (`def`) |
| **TC 식별** | `[Testcase]` 어트리뷰트 | 함수명 (`TC_...`) |
| **사전 조건** | `[Precondition("이름")]` | `testcases/functions/precondition.py` 내 함수 호출 |
| **사후 정리** | `[Cleanup("이름")]` | `testcases/functions/cleanup.py` 내 함수 호출 |

---

## 3. 상세 API 및 매커니즘 매핑

### 3.1 기본 및 검증 API 전체 목록

| 분류 | Dysnomia 확장 메서드 | 설명 | SIMVA 대응 API |
|:---|:---|:---|:---|
| **기본** | `var.Set(q)` | 신호 값 쓰기 | `simva.set_signal(sig, val)` |
| | `var.Get()` | 신호 값 읽기 | `simva.get_signal(sig)` |
| | `Wait(ms)` | 대기 (ms 단위) | `simva.wait(ms / 1000.0)` |
| **Is 계열** | `var.Is(q)` | 현재값 == q 확인 | `simva.is_eq(sig, val)` |
| | `var.Is2(q1, q2)` | q1 또는 q2 확인 | `is_eq(sig, v1) or is_eq(sig, v2)` |
| | `var.Is2v(v1, v2, q1, q2)` | 2개 변수 조건 확인 | 복합 래퍼 필요 |
| **Turns 계열** | `var.Turns(q, ms)` | ms 내 q로 전환 대기 | `simva.turn_eq(sig, val, ms/1000)` |
| | `var.Turns_Not(q, ms)` | q가 아닌 값으로 전환 | `simva.turn_ne(sig, val, ms/1000)` |
| | `v1.Turns2v(v2, q, ms)` | v1 또는 v2가 전환 대기 | 복합 래퍼 필요 |
| **Keeps 계열** | `var.Keeps(q, ms)` | ms 동안 q 유지 확인 | `simva.keep_eq(sig, val, ms/1000)` |
| | `var.Keeps_Not(q, ms)` | ms 동안 q 아님 유지 | `simva.keep_ne(sig, val, ms/1000)` |

> **중간 정리 (시간 단위)**: Dysnomia는 밀리초(ms) 단위, SIMVA는 초(sec) 단위입니다. 변환 시 `/1000.0` 처리가 필수입니다.

---

## 4. 상수 및 신호 관리 시스템

### 4.1 Quantities (상수 정의: CommonQuantities.cs)
- **C# 원천**: `_common/Predefined/Quantities/CommonQuantities.cs` (약 2000줄 이상)
- **Python 분리 전략**:
    1.  **`quantity.py`**: `[IntQuantity]`, `[FloatQuantity]` 등 신호 값 성격의 상수.
    2.  **`config.py`**: 어트리뷰트가 없거나 시간/딜레이 설정 성격의 상수.
- **변환 방식**: `CommonQuantities.cs` 내의 모든 `static` 필드를 추출하여 성격에 따라 두 모듈로 배분 생성하여 누락을 방지함.

### 4.2 Variables (신호 변수)
- **매핑 정합성**: C#의 변수 이름(예: `DRV_Door_BDC`)을 SIMVA의 실제 신호 경로(`signals.BDC.DRV_Door`)로 연결하는 매핑 DB 구축이 변환의 성패를 좌우함.

---
## 5. 변환 기술 비교: LLM vs Rule-based (하이브리드 전략)

### 5.1 Pass 1: Syntax Transpilation (100% 로직 기반)
- **과정**: C# AST를 분석하여 Python AST로 재구성하는 **Deterministic Transpiler** 방식.
- **핵심**: 클래스/메서드 구조 변환, `self` 인자 주입 등 순수 문법(Syntax) 처리.
- **범위 최적화**: 모든 C# 문법이 아닌 **테스트 스크립트에 사용되는 핵심 문법(클래스, 반복, 제어, 메서드 호출)으로만 지원 범위를 타겟팅**하여 개발 복잡도를 낮추고 신뢰성을 높입니다. 한계를 벗어나는 구문은 Pass 3(LLM)에 위임합니다.
- **이점**: 표준 문법을 로직으로 처리하여 **문법 에러율 0%**를 보장합니다.

### 5.2 Pass 2: API Mapping (규칙 로직 기반)
- **과정**: 1단계 결과물에서 Dysnomia API 노드를 SIMVA API 호출로 치환.
- **핵심**: 밀리초 $\rightarrow$ 초 단위 변환, `Set` $\rightarrow$ `set_signal` 등 명확한 API 치환 규칙 적용.
- **이점**: 도메인 특화 규칙을 기계적으로 적용하여 데이터 정합성을 확보합니다.

### 5.3 Pass 3: Semantic Intelligence (LLM 기반)
- **과정**: RAG(Vector DB) 및 LLM 추론을 통한 최종 정밀 교정.
- **핵심**: 신호 명칭의 의미적 매핑 해결, 복잡한 비즈니스 관용구(람다 등) 의역.
- **이점**: AI의 유연함으로 10%의 특수 케이스 및 의미적 완성도를 해결합니다.

---

## 6. 공통 헬퍼 메서드 (Methods) 분석

### 6.1 규모 및 ECU별 전문화
- **파일 목록**: `Methods_BDC.cs` (약 3700줄), `Methods_PSU.cs`, `Methods_SCM.cs`, `Method_EVENT.cs` 등 ECU별로 독립된 파일에 수천 줄의 로직이 구현됨.

### 6.2 주요 로직 패턴 및 변환 난이도
| 패턴 | 설명 | 난이도 | 대응 방식 |
|:---|:---|:---|:---|
| **조건부 설정** | `if(EXIST_PDC == 1) ...` | ⭐⭐⭐ | 런타임 환경 설정 기반 분기 보존 |
| **재시도 루프** | `for` 문 내 `Wait` + `break` | ⭐⭐⭐ | Pythonic 루프(`for i in range`)로 변환 |
| **복합 검증** | `Turns2v` 등 복수 변수 감시 | ⭐⭐⭐⭐ | 공용 래퍼 함수(Utility) 설계 도입 |

---

## 7. 기술적 허들 및 해결 전략

- **Mixin 전략**: C#의 `partial class` 파일을 각각의 **Mixin 클래스**로 변환 후 다중 상속을 통해 파이썬에서 하나의 `Testcases` 클래스로 통합하여 함수 구조를 보존함.
- **Self 주입**: 모든 멤버 함수에 `self` 인자를 자동 삽입하고, 클래스 멤버 호출부도 `self.xxx()` 형태로 보정함 (AST 전처리).
- **Import 자동화**: `using static` 구문을 분석하여 파이썬의 `from references import signals` 등으로 상단 자동 생성함.

---

## 8. SIMVA 실행 매커니즘: Test Suite (`sils_test.py`)

- **실행 단위**: SIMVA는 `TestSuite` 객체를 통해 개별 TC 함수를 등록하고 실행함.
- **자동화 전략**: 변환 엔진은 TC 변환 완료 후 `sils_test.py` 파일 내에 `suite.add(TC_NAME)` 구문을 자동으로 대입/업데이트하여 즉시 실행 가능한 환경을 제공함.

---

## 9. 최종 권고: 가장 효율적이고 정확한 전략

저희의 심층 분석 결과, 가장 **효율적(Efficient)**이면서도 **정확한(Accurate)** 결과를 낼 수 있는 최적의 해법은 **[하이브리드 전략]**입니다.

1.  **정확성 (Accuracy)**: 로직(Pass 1)이 파이썬의 문법적 뼈대와 시간 단위 변환을 100% 보장하고, LLM(Pass 2)이 문맥에 맞는 신호 매핑을 완성합니다.
2.  **효율성 (Efficiency)**: 90%의 단순 구문을 로직이 즉시 처리하여 LLM 호출 비용을 최소화하고 변환 속도를 극대화합니다.

---
*Generated by Antigravity Analysis Engine*
