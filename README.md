# Dysnomia to SIMVA Auto Converter

Dysnomia(C#) 환경에서 작성된 차량용 소프트웨어 테스트 스크립트를 SIMVA(Python) 환경의 테스트 스크립트로 자동으로 변환해주는 지능형 변환 엔진입니다.

## 🌟 프로젝트 개요 (Overview)

이 프로젝트는 수만 라인에 달하는 레거시 C# 테스트케이스와 헬퍼 메서드들을 단기간에 Python 기반의 SIMVA 플랫폼으로 이관하기 위해 시작되었습니다. 

단순한 정규식 치환을 넘어, 코드의 **문법적 무결성을 보장하는 AST(Abstract Syntax Tree) 분석**과 **도메인 지식을 이해하고 의역하는 LLM(Large Language Model)** 기술을 융합한 **하이브리드 아키텍처**를 특징으로 합니다.

## 🧬 핵심 아키텍처: 3-Pass Hybrid Pipeline

저희 변환 엔진은 인간 엔지니어의 통찰이 담긴 **"언어 변환 선행(Language-First)"** 다단계 파이프라인을 채택하여 가장 효율적이고 정확한 변환을 수행합니다.

1. **Pass 1: Syntax Transpilation (로직 기반 구문 변환)**
   - **역할**: C#의 구문 트리를 파싱하여 Python의 구문 트리로 변환합니다.
   - **특징**: 클래스, 메서드, `self` 주입 등 파이썬의 핵심 뼈대를 에러율 0%로 완벽하게 구축합니다 (타겟화된 핵심 문법 한정 지원).
2. **Pass 2: API Mapping (로직 기반 도메인 변환)**
   - **역할**: 1단계 결과물에서 Dysnomia 전용 API(`Wait(ms)`, `Set` 등)를 식별하여 SIMVA 엔진 API(`simva.wait(sec)`, `simva.set_signal`)로 기계적 치환을 수행합니다.
3. **Pass 3: Semantic Intelligence (LLM 기반 의미 보정)**
   - **역할**: 로직이 처리하기 힘든 모호한 신호명 매핑이나, 복잡한 비즈니스 헬퍼 로직(람다식 등)을 AI가 '의역(Refinement)'합니다.
   - **환각 방지**: RAG(Vector DB)를 통해 실제 `signals.py`의 심볼을 참조하므로, 존재하지 않는 신호를 만들어내는 치명적 오류를 원천 차단합니다.

## 📂 문서 구조 (Documentation)

자세한 기술 및 분석 문서는 `docs/` 디렉토리에 정의되어 있습니다.

- **`docs/01_analysis/` (분석 및 기술 전략)**
  - [`analysis_report.md`](docs/01_analysis/analysis_report.md): 프로젝트 구조 분해, API 전수 매핑, 핵심 헬퍼 메서드 분석 등을 통합한 **마스터 분석 보고서**
  - [`strategy_comparison.md`](docs/01_analysis/strategy_comparison.md): LLM vs Logic 기반 변환 전략의 장단점 비교 및 우리가 채택한 **3단계 하이브리드 파이프라인의 핵심 매커니즘 설명**
- **`docs/02_design/` (설계 및 구현 계획)** *(작성 진행 중)*
  - 변환기 시스템 아키텍처 및 세부 모듈 설계도

