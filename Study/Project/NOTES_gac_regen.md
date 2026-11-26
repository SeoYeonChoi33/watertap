# GAC regen costing & optimization: 진행 노트

> 다른 컴퓨터에서 이어서 작업할 때 이 파일부터 읽기.
> Claude에게: "Study/Project/NOTES_gac_regen.md 읽고 이어서 하자"

## 0. 작업 방식
- **코드는 내가 직접 짠다.** Claude는 가이드, 리뷰, 설명만 한다. 내 파일을 대신 수정하지 않는다. scratch 폴더에서 테스트하는 건 괜찮다.
- 줄 번호를 말하기 전에 항상 파일을 새로 읽는다. 내가 실시간으로 수정하고 있기 때문이다.
- 커밋과 push는 내가 요청할 때만 한다.
- 브랜치: `WaterTAP-Study`. remote가 `https://github.com/SeoYeonChoi33/watertap.git`로 이전됨 (`git remote set-url origin ...`으로 바꿔도 됨).
- conda 환경: `watertap-dev`

## 1. 전체 방향
- 변수와 식은 **WaterTAP 원본을 직접 수정해서** 넣는다. 고칠 파일은 `watertap/costing/unit_models/gac.py`이고, 필요할 때만 `watertap/unit_models/gac.py`도 고친다.
- flowsheet(`Study/Project/gac_optimization.py`)는 최대한 깔끔하게 둔다. build, fix, 범위, objective, 반복문만 넣는다.
  - flowsheet는 d425f08b 버전(최적화 이전)으로 되돌려 둔 상태이고, 새로 짤 예정이다.
  - 예전 최적화 코드(unit_width/unit_area, j 반복문 등)는 커밋 6c984079, 7960c2d8에 남아 있다.
- 원본을 직접 고치므로 upstream 업데이트는 `git fetch upstream && git merge upstream/main`으로 받는다.
- 기존 테스트가 깨질 수 있다: `watertap/unit_models/tests/test_gac.py`, `watertap/costing/unit_models/tests/test_gac.py`

### 어디에 무엇을 넣나
| 종류 | 위치 |
|---|---|
| 항상 성립하는 관계식, 비용 식 | unit / costing model |
| 변수 범위, fix/unfix, objective, 정수 반복문 | flowsheet |
| Q에 따라 고르는 값 (h 8/6 ft, redundant, C_labor) | flowsheet. **feed를 fix한 뒤에** 정한다 |
| regen_frac | flowsheet에서 옵션별로 fix (on-site 0.9, off-site 0.7, disposal은 식에서 안 씀) |

## 2. 현재 진행 상황 (regen costing)
**지금 할 일:** regen 옵션 추가와 옵션별 costing. 최적화는 보류.

- [x] **1단계** `regen_option` 전달 경로 (커밋 6ca168e9)
  - `cost_gac(blk, contactor_type, regen_option="watertap")` → `cost_gac_{pressure,gravity}(blk, regen_option="watertap")` → `_cost_gac(blk, parameter_blk, regen_option)`
  - 허용 값: `"watertap"`(기본, WaterTAP 원래 방식), `"onsite"`, `"offsite_nonhaz"`, `"offsite_haz"`, `"disposal_nonhaz"`, `"disposal_haz"`. 이 외의 값은 `ConfigurationError`로 막는다.
  - reference flowsheet `python watertap/flowsheets/gac/gac.py`가 LCOW **0.5636**으로 그대로 나오는 것 확인.
- [ ] **2단계** 파라미터 블록에 계수 넣기 ← **지금 여기**
- [ ] **3단계** `_cost_gac` 안에 공통 중간값 (`pyo.Expression` 추천)
- [ ] **4단계** 옵션별 식. 순서: offsite_nonhaz → disposal_nonhaz → haz 두 개 → onsite
- [ ] 테스트용 flowsheet: 설계 고정(DOF 0. 예: EBCT 600 s, h 8 ft, j=1, 1 MGD)으로 5개 옵션 비교

### 2단계: 파라미터 블록 (`_build_gac_cost_param_block`, `energy_consumption_coeff` 아래)
- 파라미터 블록은 contactor type마다 **하나만 만들어지고 공유**된다 (`m.fs.costing.gac_gravity`).
- `regen_option`을 모르는 곳이라 여기서는 분기할 수 없다. 옵션 5개에 필요한 값을 **전부 만들어 두고**, 어느 걸 쓸지는 `_cost_gac`에서 if문으로 고른다.
- 데코레이터(`register_costing_parameter_block`)가 블록 안의 **모든 Var를 자동으로 fix**한다. 그래서 DOF가 늘지 않고, flowsheet에서 `.fix(값)`으로 값을 바꿀 수 있다.
- 기존 WaterTAP regen 식과 Var(`regen_frac`, `regen_unit_cost`, `makeup_unit_cost`, `gac_regen_cost`, `gac_makeup_cost`)는 **지우지 않는다.**
  - `gac.py` reference flowsheet, `gac_ui.py`, `dye_desalination.py`, 테스트가 이것들을 쓴다.
  - `"watertap"` 옵션 분기로 남겨 두고, 비교 기준으로도 쓴다.
  - `gac_regen_cost`와 `gac_makeup_cost`는 이름을 재사용하고, 옵션에 따라 constraint 식만 다르게 만든다.

**방식**
- 단가, 거리, 무게: 단위를 붙인 Var
- 회귀식: 계수 `_coeff`(결과 단위를 붙임)와 지수 `_exp`(무차원)를 따로 만든다
- 기존 개념과 겹치면 `_epa`를 붙이고, haz 구분은 `_haz` / `_nonhaz`를 맨 뒤에 붙인다

| 이름 | 값 | units |
|---|---|---|
| `makeup_unit_cost_epa_coeff` / `_exp` | 4.6333 / −1.2427e-5 | USD_2021/kg / kg⁻¹ |
| `regen_unit_cost_epa_coeff` / `_exp` | 4.3914 / −2.9154e-5 | USD_2021/kg / kg⁻¹ |
| `tank_cost_coeff` / `_exp` | 308.94 / 0.5015 | USD_2021 (연도 확인 필요) / 무차원 |
| `tank_volume_factor` | 1.33 | 무차원 |
| `furnace_cost_coeff` / `_exp` | 97378.2 / 0.3834 | USD_2021 (연도 확인 필요) / 무차원 |
| `furnace_capacity_factor` | 1.177 | 무차원 (= 1.0 / 0.85 가동률) |
| `onsite_material_cost_coeff` / `_exp` | 8451.4 / 0.2017 | USD_2021/yr (연도 확인 필요, 2000 추정) / 무차원 |
| `onsite_labor_hours_coeff` / `_exp` | 2.2515 / 0.7554 | hr/yr / 무차원 |
| `onsite_elec_use_coeff` / `_exp` | 9.1217 / 1.0475 | kWh/yr / 무차원 |
| `onsite_gas_use_coeff` | 28.161 | therm/yr |
| `labor_unit_cost` | 45.51 (flowsheet에서 Q 구간 보고 fix) | USD_2021/hr |
| `onsite_elec_unit_cost` | 0.11 | USD_2021/kWh |
| `gas_unit_cost` | 0.5786 | USD_2021/therm |
| `transport_unit_cost_nonhaz` / `_haz` | 0.212 / 0.283 | USD_2021/(ton·mi) |
| `transport_distance_nonhaz` / `_haz` | 10 / 200 | mi |
| `transport_min_charge` | 3392 | USD_2021 (연도 확인 필요) / shipment |
| `max_waste_ship` | 18 | ton |
| `disposal_unit_cost_nonhaz` / `_haz` | 110.04 / 363.41 | USD_2021/ton |
| (재사용) `bed_mass_max_ref` | 18143.7 kg (= 40,000 lb) | 기존 |

- Pyomo 단위: `ton`(short ton = 907.18 kg), `mi`, `lb`, `gal`, `kWh`, `therm`, `hr`, `year`, `day`, `ft`. 통화는 `USD_2000`~`USD_2023`만 있다 (2024 이후 없음).
- 확인 방법: 다 넣은 뒤 `python watertap/flowsheets/gac/gac.py`를 돌려서 LCOW 0.5636이 그대로 나오면 된다.

## 3. regen 옵션 비용 식 (EPA WBS 기반, USD2021)
공통 기호:
- ṁ_gac: `gac_usage_rate` (kg/s). 연간 양으로 변환해서 쓴다.
- freq = 1/`operational_time`
- V_bed = `bed_volume / num_contactors_op` (contactor 하나의 GAC 부피)
- V_storage[gal] = 1.33 × V_bed(gal로 변환)
- regen_capacity[lb/day] = ṁ_gac(lb/day) × 1.177

### On-site (f_regen 0.9)
- **CAPEX:**
  - furnace `97378.2 × cap^0.3834`
  - tank 3개 `3 × 308.94 × V_storage^0.5015`
- **OPEX:**
  - makeup `(1−f)·ṁ·C_unit,gac`
  - material `8451.4·cap^0.2017`
  - labor `2.2515·cap^0.7554 [hr/yr] × C_labor`
  - 전기 `9.1217·cap^1.0475 [kWh/yr] × 0.11`
  - gas `28.161·cap [therm/yr] × 0.5786`

### Off-site non-haz (f 0.7)
- **CAPEX:** tank 3개
- **OPEX:** makeup `(1−f)·ṁ·C_unit,gac` + regen `f·ṁ·C_regen`. 운송비는 regen 단가에 포함돼서 0.

### Off-site haz (f 0.7)
non-haz와 같고, 운송비가 추가된다:
`MAX(shipments × 3392, f·ṁ[ton/yr] × 200 mi × 0.283)`

### Disposal non-haz
- **CAPEX:** tank 2개
- **OPEX:**
  - makeup `ṁ·C_unit,gac`
  - 처분 `ṁ[ton/yr] × 110.04`
  - 운송 `ṁ[ton/yr] × 10 mi × 0.212`

### Disposal haz
- **CAPEX:** tank 2개
- **OPEX:**
  - makeup `ṁ·C_unit,gac`
  - 처분 `ṁ[ton/yr] × 363.41`
  - 운송 `MAX(shipments × 3392, ṁ[ton/yr] × 200 mi × 0.283)`

### 단가 회귀식
- **C_unit,gac** = `4.6333·e^(−1.2427e-5·x)`, x = min((1−f)·ṁ[kg/yr], 18143.7 kg)
- **C_regen** = `4.3914·e^(−2.9154e-5·x)`, x = min(f·ṁ[kg/yr], 18143.7 kg)
- x는 **연간 양**이 맞다 (확인함). 40,000 lb 이상에서는 규모의 경제가 더 이상 작용하지 않는다.
- 1 MGD에서는 on-site makeup을 빼면 모두 상한에 걸려서, 단가가 사실상 상수가 된다.

### C_labor (flowsheet에서 Q로 골라 fix)
| Q (MGD) | 유효 단가 ($/hr) |
|---|---|
| Q < 0.741 | 45.51 (외삽) |
| 0.741–2.1521 | 45.51 |
| 2.1521–7.3651 | 49.16 |
| 7.3651–22.6141 | 51.93 |
| 22.6141–75.07 | 59.22 |
| > 75.07 | 59.22 (외삽) |

### Shipments
- spent_media(1회 양):
  - off-site: `f·ṁ[ton/yr] / freq`
  - disposal: `ṁ[ton/yr] / freq`
- shipments = `ceil(spent_media / 18 ton) × freq`

### 미분할 수 없는 항목 처리
- **min:** `smooth_min` (WaterTAP도 초기 충전 단가에 이미 쓰고 있음)
- **max:** `smooth_max`. 1 MGD haz에서는 최소 운송비 쪽이 약 7배 크다.
- **ceil:** 정수 k를 fix하고 `spent_media ≤ 18 ton × k`, `shipments = k × freq`로 쓴다. k = 1, 2, 3을 반복해서 풀고 가장 싼 걸 고른다. EBCT 범위 안에서 ceil 값이 1→2로 바뀐다 (off-site는 EBCT 약 1180 s, disposal은 약 830 s에서).

### 단위 주의
- `a·x^b`는 `(x / (lb/day))**b`처럼 밑을 무차원으로 만든 뒤 지수를 붙인다.
- **전기:** WaterTAP의 `cost_flow("electricity")`는 flowsheet 단가 0.07을 쓴다. on-site furnace 전기(0.11)는 `cost_flow`가 아니라 OPEX 식에 직접 넣는다.
- 초기 충전 CAPEX(`adsorbent_cost`)는 **WaterTAP 식을 그대로** 둔다. 바꾸는 건 regen뿐이고, 단가식 차이는 나중에 sensitivity analysis에서 다룬다.

### 아직 안 정한 것
- **USD 연도:** furnace, tank, material, 최소 운송비 3392. 일단 USD_2021로 가정한다.
- **maintenance factor(CAPEX의 3%)와 on-site O&M 중복:** furnace에는 3%를 빼고 tank에는 둔다는 쪽으로 기울어 있음. 교수님과 상의할 예정.
  - 방법 후보: `m.fs.costing.maintenance_labor_chemical_factor.fix(0)` + costing 안에서 `0.03 × (contactor + adsorbent + other)`
- base_currency 연도는 보류. 나중에 USD2023으로 통일할 수도 있다.

## 4. 최적화 (보류. regen costing을 끝낸 뒤에)
- **변수:** EBCT (456–1560 s), n_op = j (정수)
- **gravity:** h(bed_length) 고정. Q ≥ 1 MGD면 8 ft, 1 MGD 미만이면 6 ft. 직렬 i = 1.
- **contactor 개수:** `bed_area = j × unit_area`, `unit_area = width²`. 총 개수 = j + redundant (Q < 10 MGD면 1, Q ≥ 10 MGD면 2).
- **n_op는 공정 성능에 영향이 없다** (velocity, kf, BV 모두 그대로). costing에만 영향을 준다. 최적 n_op는 Q × EBCT, redundant, 비용 곡선, width 범위로 정해지고, 1/n과 kf는 최적 EBCT를 거쳐 간접적으로만 영향을 준다.
- **정수 처리:** j 반복문. 20번 푸는 데 약 2초라 충분히 빠르다. 반복 전에 가능한 j 범위를 미리 계산해서 infeasible한 j는 건너뛴다.
  - j_min = ceil(Q·EBCT_min / (h·A_max)), j_max = floor(Q·EBCT_max / (h·A_min))
- **sweep 범위 후보:** 1–75 MGD, j 상한 약 30

### bed(contactor) 비용식 문제 → 나중에 직접 회귀식을 새로 구할 예정
- WaterTAP gravity: `f(V) = 75131.3 + 735.55·V − 1.01827·V²` (USD2020, V = contactor 1기 부피 m³)
  - V ≈ 361 m³에서 최대이고, 그보다 크면 오히려 싸진다. V ≈ 813 m³ 이상이면 음수가 된다.
  - WaterTAP 문서에는 유효 범위가 없다. "상용 가능한 크기가 되도록 사용자가 개수를 정하라"는 말뿐이다.
  - EPA WBS gravity basin은 하나의 식이 아니다. 콘크리트 부피 × 단가에 부속 장비 회귀식을 더한 것인데, WaterTAP이 이걸 하나로 묶어서 회귀했다.
- **크기 상한이 없으면** 약 10–30 MGD부터 최적이 j = 1로 쏠린다 (비현실적).
- **width 상한:** 30 ft(EPA 설계 기준, V ≤ 204 m³) 또는 약 40 ft(h = 8 ft에서 V = 361 m³, 비용 곡선 꼭짓점). 50 ft는 비추천.
- **실제 사례:** 240 MGD 정수장, 50 ft 정사각형 12기, depth 11.5 ft → V_unit 814 m³. WaterTAP 식으로는 비용이 약 0이 되어 재현할 수 없다.
- **EPA WBS 결과의 depth 9.7 ft vs 8 ft 고정:** 원인을 확인해야 한다. 후보는 depth = EBCT × loading rate / 7.48 이거나, basin 전체 높이일 가능성.
- 참고 그림/스크립트 (다른 컴퓨터에는 없음): Claude scratch 폴더의 `bed_capex_heatmap.py`

## 5. 기타 메모
- min_ebct 제약은 넣지 않는다. placeholder solute로는 infeasible하다 (surface diffusion 항만으로 약 2684 s > 1560 s). PFAS 데이터를 넣을 때 다시 본다.
- GAC config 철자 `cphsdm_calaculation_method`는 WaterTAP 원본 철자 그대로가 맞다.
- WaterTAPCosting: LCOW = (CAPEX × 0.1 + OPEX) / (Q × 0.9 × 1년)
