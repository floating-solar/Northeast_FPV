# CLAUDE.md

이 파일은 이 저장소에서 코드 작업 시 Claude Code(claude.ai/code)에게 제공하는 안내 문서입니다.

## 프로젝트 개요

*Gallaher et al. (2024): 미국 북동부 수상 태양광 경관의 모델링된 지속가능성 트레이드오프* 논문을 위한 연구 코드입니다. 미국 북동부 14개 주 + DC 전역의 수상 태양광(FPV) 발전 잠재량을 2단계 파이프라인으로 모델링합니다: GIS 기반 적합지 선별 → 에너지 시뮬레이션.

## 파이프라인 실행 방법

스크립트는 순서대로 실행해야 합니다. 별도의 빌드 시스템이나 테스트 스위트는 없으며, 각 스크립트는 독립적인 처리 단계입니다.

**1단계 — 전처리 (ArcGIS Pro + `arcpy` 필요):**
```
Pre_Processing/FPV_NID_Preprocessing.py   # NID 댐 데이터 정제 및 지오코딩
Pre_Processing/NIW_Data_Cleaning.py        # 주별 국가 습지 목록(NIW) 정제
Pre_Processing/Local_Roads.py              # FIPS 코드별 DOT 지방 도로 추출/병합
Pre_Processing/NIW_NID_Join.py             # 공간 결합: NHD 수계 + NID + TNC + 도로/송전선
```

**2단계 — 에너지 분석 (PySAM + NREL API 키 필요):**
```
Pre_Processing/PySAM_weatherFiles.py       # 위치별 NREL PSM3 기상 파일 다운로드
FPV_Analysis/Solar_Estimation.py           # 수계별 PVWatts v8 실행, 연간 kWh 출력
```

## 실행 전 설정

**모든 파일 경로는 플레이스홀더(`" "`)로 되어 있으며, 스크립트 실행 전 반드시 실제 경로로 채워야 합니다.** `## path to...` 또는 `## set working directory` 형태의 주석을 찾아 수정하세요.

스크립트별 주요 경로 설정:
- `FPV_NID_Preprocessing.py`: NID 원본 CSV 경로, ArcGIS 지오데이터베이스 작업 공간
- `NIW_Data_Cleaning.py`: 이전 단계 결과 폴더, 주별 습지 피처 클래스
- `Local_Roads.py`: 지방 도로 GIS 데이터 폴더, 연구 지역 경계 파일, 출력 폴더
- `NIW_NID_Join.py`: 작업 공간 지오데이터베이스, 출력 위치, 지방 도로 지오데이터베이스 경로
- `PySAM_weatherFiles.py`: NREL API 키 + 이메일, 기상 파일 저장 디렉토리, 입력 CSV 경로
- `Solar_Estimation.py`: 위치 + 시스템 용량이 담긴 입력 CSV, SAM 설정 JSON 경로, 기상 파일 폴더, 출력 CSV 경로

## 아키텍처

### 데이터 흐름

```
NID (댐)         ──┐
NHD (수계)       ──┤  FPV_NID_Preprocessing.py  →  Northeast_NID_Proj (GIS)
                   │  NIW_NID_Join.py             →  Northeast_Selection_2 (최종 적합 수계)
NIW (습지)       ──┤  NIW_Data_Cleaning.py        →  Northeast_Wetlands (GIS, 제외 마스크로 사용)
지방 도로        ──┤  Local_Roads.py              →  Northeast_local_Roads (GIS)
TNC 호수/연못    ──┘
        │
        ▼ (위경도 + system_size_kw + 기상 파일 ID가 담긴 CSV)
        │
PySAM_weatherFiles.py  →  NREL PSM3 .csv 기상 파일 (위치별 1개)
        │
Solar_Estimation.py    →  수계별 Year1_Energy_kWh가 담긴 출력 CSV
```

### 적합지 선별 기준 (`NIW_NID_Join.py`에 구현)
- 수계 면적 ≥ 2.5 에이커 (NHD 기준)
- 습지/늪, 하구, 플라야 유형 제외
- NID 댐 레코드로부터 0.06 마일 이내
- 송전선 또는 변전소로부터 1 마일 이내
- 지방 도로로부터 0.5 마일 이내

### 에너지 시뮬레이션 (`SAM_config.json`)
PVWatts v8 템플릿 파라미터: 100 MW 시스템, 고정 경사각 배열 (11°, 남향), DC:AC 비율 1.15, 손실 약 14%. `solar_resource_file`과 `system_capacity`는 `Solar_Estimation.py` 실행 시 위치별로 덮어씁니다.

## 주요 의존성

- **arcpy** (ArcGIS Pro Python 환경) — 모든 `Pre_Processing/` 스크립트에 필요
- **PySAM** — NREL System Advisor Model Python 래퍼; `pip install NREL-PySAM`
- **pandas**, **numpy** — 데이터 처리
- **NREL API 키** — `PySAM_weatherFiles.py` 사용을 위해 https://developer.nrel.gov 에서 등록 필요

## 좌표계

- 원본 입력: WGS 1984 (EPSG:4326)
- NID/NHD 처리: NAD 1983 UTM Zone 18N (EPSG:26918)
- 습지 처리: USA Contiguous Albers Equal Area Conic (ESRI:102004)

## 연구 지역

14개 주 + DC: 메인, 뉴햄프셔, 버몬트, 뉴욕, 매사추세츠, 로드아일랜드, 코네티컷, 뉴저지, 펜실베이니아, 델라웨어, 메릴랜드, 웨스트버지니아, 버지니아, 워싱턴 D.C.
