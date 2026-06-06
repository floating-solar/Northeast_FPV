# Solar_Estimation.py 분석 설명

## 개요

`Solar_Estimation.py`는 수상 태양광(FPV) 후보 수계별로 NREL PVWatts v8 시뮬레이션을 실행하여 연간 발전량(kWh)을 산출하는 스크립트입니다.

---

## 필요한 입력 파일

| 파일 | 형식 | 설명 |
|------|------|------|
| 위치·용량 CSV | `.csv` | 수계별 위경도, 시스템 용량, 기상 파일 ID가 담긴 입력 파일 |
| SAM 설정 JSON | `.json` | PVWatts v8 시뮬레이션 파라미터 (`SAM_config.json`) |
| 기상 파일 폴더 | `.csv` (NREL PSM3 형식) | `PySAM_weatherFiles.py`로 다운로드한 위치별 기상 데이터 파일 |

### 위치·용량 CSV 필수 컬럼

| 컬럼명 | 설명 |
|--------|------|
| `fpv_id` | 수계(FPV 후보지) 고유 ID |
| `water_lat` | 수계 중심 위도 |
| `water_lon` | 수계 중심 경도 |
| `fpv_system_size_kw` | PV 시스템 용량 (kWdc) |
| `weather_files` | 해당 위치의 기상 파일명 (확장자 포함) |

---

## 분석 절차

### 1단계 — 입력 데이터 로드

위치·용량 CSV를 읽어 수계별로 다음 정보를 리스트로 수집합니다.

- `FPV_ID`: 수계 고유 ID
- `Kw_list`: 시스템 용량 (kWdc)
- `weather_files`: 기상 파일명
- `Lat` / `Long`: 위경도

### 2단계 — PVWatts v8 모델 초기화

`PySAM.Pvwattsv8`로 새 시뮬레이션 인스턴스를 생성하고, `SAM_config.json`에서 기본 파라미터를 불러와 모델에 적용합니다.

주요 고정 파라미터 (`SAM_config.json` 기준):
- 시스템 용량: 100 MW (수계별로 덮어씌워짐)
- 배열 유형: 고정 경사각 (Fixed Tilt)
- 경사각: 11°, 방위각: 남향(180°)
- DC:AC 비율: 1.15
- 시스템 손실: 약 14%

### 3단계 — 수계별 반복 시뮬레이션

CSV의 각 행(수계)에 대해 아래 과정을 반복합니다.

1. **기상 파일 경로 설정** — 기상 파일 폴더 경로와 파일명을 결합하여 `solar_resource_file`에 지정
2. **시스템 용량 설정** — `fpv_system_size_kw` 값을 `system_capacity`에 적용 (JSON 기본값 덮어씌움)
3. **시뮬레이션 실행** — `system_model.execute()` 호출
4. **결과 수집** — `Outputs.ac_annual` (연간 AC 발전량, kWh)을 리스트에 저장

### 4단계 — 결과 저장

수집된 결과를 DataFrame으로 조합하여 CSV로 출력합니다.

출력 CSV 컬럼:

| 컬럼명 | 설명 |
|--------|------|
| `fpv_id` | 수계 고유 ID |
| `water_lat` | 위도 |
| `water_lon` | 경도 |
| `Year1_Energy_kWh` | 연간 AC 발전량 (kWh) |
| `weather_files` | 사용된 기상 파일명 |
| `System_Size_kW` | 시스템 용량 (kWdc) |

---

## 실행 전 경로 설정

스크립트 내 아래 3곳의 플레이스홀더(`' '`)를 실제 경로로 채워야 합니다.

```python
# 1. 위치·용량 CSV 경로
kW_filename = open('<CSV 폴더 경로>/<파일명.csv>', 'r')

# 2. SAM 설정 JSON 경로
with open('<SAM_config.json 경로>', 'r') as f:

# 3. 기상 파일 폴더 경로
weather_path = ("<기상 파일 폴더 경로>/%s") % weather_files[i]
```

---

## 데이터 흐름 요약

```
위치·용량 CSV
    │  (fpv_id, lat, lon, system_size_kw, weather_files)
    ▼
SAM_config.json ──► PVWatts v8 모델 초기화
    │
    ▼
수계별 반복:
  기상 파일(.csv) → solar_resource_file 설정
  system_size_kw  → system_capacity 설정
  system_model.execute()
  ac_annual 수집
    │
    ▼
출력 CSV (Year1_Energy_kWh 포함)
```

---

## 주요 의존성

- `PySAM` (`NREL-PySAM`) — PVWatts v8 시뮬레이션 엔진
- `pandas` — DataFrame 생성 및 CSV 출력
- `json` — SAM 설정 파일 파싱
- `csv` — 입력 CSV 읽기

---

## 심화 설명

### 1. `water_lat` / `water_lon` — 위경도의 역할

`water_lat`(위도)과 `water_lon`(경도)은 NHD 수계 폴리곤의 **중심점 좌표**입니다.

`Solar_Estimation.py` 내에서 이 두 컬럼은 **시뮬레이션 계산에 직접 사용되지 않습니다.** 위경도 기반 위치 데이터는 이미 이전 단계인 `PySAM_weatherFiles.py`에서 활용되었습니다.

```
water_lat / water_lon
    │
    ▼ (PySAM_weatherFiles.py 단계)
NREL API 요청 → 해당 위치의 PSM3 기상 파일 다운로드
    │
    ▼ (Solar_Estimation.py 단계)
weather_files 컬럼의 파일명으로 기상 파일 로드
water_lat / water_lon → 출력 CSV에 참조 좌표로만 기록
```

| 단계 | water_lat / water_lon 활용 |
|------|---------------------------|
| `PySAM_weatherFiles.py` | NREL API에 위경도를 전달하여 해당 지점의 기상 파일 다운로드 |
| `Solar_Estimation.py` | 시뮬레이션에 직접 사용되지 않음; 출력 CSV의 위치 추적용으로만 저장 |

즉, 위경도 정보는 **이미 기상 파일 파일명에 인코딩**되어 있으며 (`weather_files` 컬럼), `Solar_Estimation.py`는 파일명을 통해 간접적으로 위치 정보를 활용합니다.

---

### 2. `fpv_system_size_kw` — 시스템 용량의 의미와 계산 방법

`fpv_system_size_kw`는 해당 수계에 설치 가능한 FPV 시스템의 **DC 설계 용량 (kWdc)** 입니다. PVWatts v8의 `system_capacity` 입력값으로 사용됩니다.

#### 계산 원리

수체 면적(에이커)에 GCR과 패널 출력 밀도를 곱하여 산출합니다.

```
fpv_system_size_kw =
    Acres × 4,046.86 m²/acre   ← 단위 변환
    × GCR (0.3)                 ← 패널 설치 커버리지 (SAM_config.json)
    × 패널 출력 밀도 (W/m²)     ← 모듈 사양
    ÷ 1,000                     ← W → kW 단위 변환
```

`SAM_config.json`의 GCR = 0.3, 표준 모듈(~155 W/m²) 기준 근사값:

```
fpv_system_size_kw ≈ Acres × 188 kW/acre
```

#### 이 연구에서의 적용

- `Acres` 필드는 `NIW_NID_Join.py` 21~22행에서 NHD 수체 폴리곤에 대해 `CalculateGeometryAttributes`(측지 면적, 에이커 단위)로 계산됩니다.
- 최소 적합 면적 기준: **2.5 에이커 이상** (약 10,117 m²)
- **`fpv_system_size_kw`는 이 저장소의 스크립트 어디에도 직접 계산되지 않습니다.** `Northeast_Selection_2` GIS 피처 클래스를 CSV로 내보낸 뒤 ArcGIS Field Calculator 또는 별도 과정에서 `Acres` 필드를 기반으로 산출된 값입니다.
- 결론적으로 **NHD 수체 폴리곤의 면적(Acres) 하나만 있으면** `fpv_system_size_kw`를 유도할 수 있으며, 이것이 `Solar_Estimation.py`에서 수체별로 달라지는 유일한 시스템 파라미터입니다 (기상 파일 경로와 함께).
- 산출된 `fpv_system_size_kw` 값은 `SAM_config.json`의 기본값(100,000 kW = 100 MW)을 **수계별로 덮어씌워** 각 수체의 실제 규모에 맞는 시뮬레이션을 수행합니다.

---

### 3. `SAM_config.json` 파라미터 상세 설명

PVWatts v8 시뮬레이션의 고정 파라미터를 담은 JSON 템플릿입니다. `Solar_Estimation.py` 실행 시 `solar_resource_file`과 `system_capacity`는 수계별로 덮어씌워집니다.

#### 기상 및 자원 관련

| 파라미터 | 값 | 설명 |
|---------|-----|------|
| `solar_resource_file` | 경로 문자열 | 기상 파일 경로. 템플릿에는 Phoenix AZ 파일이 기본값으로 설정되어 있으나, **실행 시 각 수계의 기상 파일로 반드시 덮어씌워짐** |
| `albedo` | `[0.2 × 12]` | 월별 지표면 반사율 (12개월). `0.2` = 일반 지표면의 20% 반사. `use_wf_albedo`가 1이면 기상 파일 값이 우선 적용됨 |
| `use_wf_albedo` | `1` | 기상 파일 내 albedo 데이터 사용 여부 (`1` = 기상 파일 값 우선 사용, `0` = 위 `albedo` 배열 사용) |

#### 시스템 용량

| 파라미터 | 값 | 설명 |
|---------|-----|------|
| `system_capacity` | `100000` (kWdc) | DC 설계 용량. 템플릿 기본값은 100 MW이나, **실행 시 `fpv_system_size_kw`로 수계별 덮어씌워짐** |

#### 모듈 및 배열 구성

| 파라미터 | 값 | 설명 |
|---------|-----|------|
| `module_type` | `0` | 모듈 유형. `0` = Standard (결정질 실리콘, 효율 약 15%), `1` = Premium (고효율 단결정), `2` = Thin film |
| `bifaciality` | `0` | 양면 수광 모듈(Bifacial) 계수. `0` = 단면(Monofacial) 모듈 사용 |
| `array_type` | `0` | 배열 방식. `0` = Fixed Open Rack (고정 경사각, 개방형 랙). FPV 특성상 트래커 없이 고정 설치 |
| `tilt` | `11` (°) | 패널 경사각. 북동부 위도 대비 낮은 경사각은 FPV 설치 환경(수면 위 낮은 지지대)을 반영 |
| `azimuth` | `180` (°) | 방위각. `180°` = 정남향 (북반구 최대 일사 수광 방향) |
| `gcr` | `0.3` | Ground Coverage Ratio (지상 커버리지 비율). 전체 설치 면적 중 패널이 차지하는 비율 30%. 음영 손실과 발전량 사이의 트레이드오프를 조절 |

#### 손실 파라미터

| 파라미터 | 값 | 설명 |
|---------|-----|------|
| `losses` | `14.076` (%) | 시스템 총 손실률. 배선 손실, 불일치 손실, 가용성 손실 등 복합 손실의 합산값. PVWatts 기본 손실 구성 요소를 합산한 결과 |
| `soiling` | `[0 × 12]` | 월별 패널 오염 손실 (%). 수상 태양광은 비에 의한 자연 세정 효과로 오염 손실이 낮아 모두 `0`으로 설정 |
| `en_snowloss` | `0` | 적설 손실 계산 활성화 여부. `0` = 비활성화 (수면 위 설치로 적설 자정 효과 가정) |

#### 인버터 및 기타

| 파라미터 | 값 | 설명 |
|---------|-----|------|
| `dc_ac_ratio` | `1.15` | DC:AC 비율 (Inverter Loading Ratio). DC 설계 용량이 AC 용량보다 15% 크게 설계. 인버터 클리핑을 최소화하며 용량 이용률을 높임 |
| `inv_eff` | `96` (%) | 인버터 변환 효율 96% |
| `batt_simple_enable` | `0` | 간이 배터리 저장장치 모델 활성화 여부. `0` = 배터리 없음 (발전 즉시 계통 연계 가정) |
| `constant` | `0` | 추가 상수 손실 (%). 별도의 고정 손실 없음 |

---

### 4. 기상 파일 (NREL PSM3)에서 사용되는 데이터

기상 파일은 `PySAM_weatherFiles.py`에서 NREL PSM3 (Physical Solar Model v3) API로 다운로드한 시간별 기상 데이터입니다. PVWatts v8이 1년(8,760시간) 시뮬레이션에 사용하는 주요 변수는 다음과 같습니다.

#### 핵심 입력 변수

| 변수 | 단위 | 설명 |
|------|------|------|
| **GHI** (Global Horizontal Irradiance) | W/m² | 수평면 전일사량. 직달 + 확산 일사의 합. 경사면 일사량(POA) 계산의 기반 |
| **DNI** (Direct Normal Irradiance) | W/m² | 법선면 직달일사량. 태양 방향에 수직인 면에 도달하는 직사광 |
| **DHI** (Diffuse Horizontal Irradiance) | W/m² | 수평면 확산일사량. 대기 산란을 통해 도달하는 간접광 |
| **대기온도** (Dry Bulb Temperature) | °C | 모듈 온도 계산에 사용. 온도가 높을수록 결정질 실리콘 효율 저하 |
| **풍속** (Wind Speed) | m/s | 모듈 냉각 효과 계산. 풍속이 높을수록 모듈 온도 낮아져 효율 향상 |
| **Albedo** (지표면 반사율) | 무차원 | `use_wf_albedo = 1` 설정 시 기상 파일의 월별 반사율 사용. 수면 반사 특성 반영 |

#### PVWatts v8 내부 계산 흐름

```
기상 파일 (시간별)
    │
    ▼ ① 경사면 일사량 계산
GHI + DNI + DHI + 경사각(11°) + 방위각(180°) + albedo
    → POA (Plane of Array Irradiance, W/m²)
    │
    ▼ ② 모듈 온도 계산
POA + 대기온도 + 풍속
    → Tcell (모듈 셀 온도, °C)
    │
    ▼ ③ DC 발전량 계산
POA × system_capacity × 온도 보정 계수
    → DC 출력 (kWdc)
    │
    ▼ ④ AC 발전량 계산
DC 출력 × 인버터 효율(96%) × DC:AC 비율(1.15) × 손실(14.076%)
    → AC 출력 (kWac)
    │
    ▼ ⑤ 연간 합산
8,760시간 AC 출력 합산
    → ac_annual (Year1_Energy_kWh)
```

#### 수상 태양광(FPV) 특유의 기상 효과

수면 위 설치라는 FPV의 특성은 기상 파일 활용 방식에 다음과 같은 영향을 줍니다.

- **Albedo**: 수면의 반사율(약 0.06~0.10)은 일반 지표면(0.20)보다 낮지만, `use_wf_albedo = 1`로 기상 파일의 실측 albedo를 사용
- **온도**: 수면의 증발 냉각 효과로 모듈 온도가 지상 설치 대비 낮아져 효율이 소폭 향상될 수 있으나, 본 모델에서는 별도 FPV 보정 없이 표준 PVWatts 모델 적용
- **적설**: `en_snowloss = 0`으로 수면 위 설치 환경의 자연 제설 효과 반영
