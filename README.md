# Training Log — 개인 헬스 기록 웹앱

광고 없이 혼자 쓰는 근력운동 기록 앱. **의존성/빌드 없는 단일 HTML 파일**(`index.html`) 하나로 동작하며, 데이터는 브라우저 `localStorage`에 저장된다. 아이폰에서 GitHub Pages로 호스팅해 "홈 화면에 추가"로 앱처럼 쓴다. UI는 한국어, 모바일 우선.

> 이 문서는 새로 합류하는 개발자/AI 에이전트가 코드를 빠르게 파악하도록 쓴 핸드오프 문서입니다. 구현 세부는 항상 코드가 최종 기준입니다.

---

## 1. 기본 원칙 (꼭 지킬 것)

- **단일 파일**: 모든 HTML/CSS/JS를 `index.html` 하나에 인라인. 외부 라이브러리·CDN·빌드 스텝 **없음**. 차트/그래픽도 순수 SVG/CSS/Canvas로 직접 그린다. 시스템 폰트만 사용.
- **저장소는 localStorage뿐**: 서버·DB 없음. 기기별로 데이터가 독립(동기화 없음). 백업/이전은 JSON 내보내기/불러오기(⋯ 메뉴).
- **한국어 UI, 모바일 우선**. 큰 탭 타깃, 하단 5탭 내비.
- 수정 후엔 반드시 로컬 프리뷰로 검증(아래 6번). 콘솔 에러 0 확인.

---

## 2. 실행 & 배포

### 로컬에서 열기/미리보기
정적 서버로 서빙하면 된다(파일 직접 열기 `file://`도 대체로 되지만 서버 권장):
```bash
cd ~/Applications && python3 -m http.server 8091
# 브라우저에서 http://localhost:8091/index.html
```

### 배포 (아이폰용)
- GitHub 저장소 **`apriots6/Workout-log`**(public)에서 **GitHub Pages**로 서빙.
- 업데이트 = 저장소의 `index.html`을 **같은 이름으로 덮어쓰기 커밋** → 1~2분 뒤 아이폰에서 새로고침.
- 아이폰: Safari로 Pages URL 접속 → 공유 → "홈 화면에 추가".
- iOS는 로컬 `file://` 앱에 영구 저장을 못 하므로 호스팅이 필요.

### 버전 확인 (업데이트 반영 체크)
- `<script>` 최상단의 `var APP_VERSION="v1.3";` 상수. 헤더 "Training Log" 옆 배지로 표시된다.
- **배포할 때마다 이 값을 올릴 것**(v1.1 → v1.2 …). 아이폰에서 배지 번호가 바뀌면 업데이트가 반영된 것.

---

## 3. 데이터 모델 (localStorage, 접두어 `tl.`)

`var K = {settings, exercises, sessions, routines, active}` — 실제 키는 `tl.settings` 등.

```
tl.settings  → { unit:"kg", incrementKg:2.5, incrementLb:5, bodyweightKg:70 }
               unit 은 집계 표시용(항상 "kg"로 강제). 무게 단위는 종목별로 관리.
tl.exercises → [{ name, bodyPart, best1RM(kg), mode }]   // 종목 카탈로그 + 최고 추정1RM
tl.sessions  → [ 완료 세션 ]  (아래 구조)
tl.routines  → [{ id, name, bodyPart, items:[{exercise, bodyPart, targetSets, targetReps}] }]
tl.active    → 진행 중 세션 1개(없으면 null). 새로고침 복구용.
```

세션/활성 세션 구조:
```
{ id, date("YYYY-MM-DD"), startTs, endTs, durationSec,
  routineId?, routineName?,           // 루틴에서 시작한 경우만
  entries:[
    { exercise, bodyPart, unit("kg"|"lb"), mode("weight"|"assist"), memo,
      sets:[ { weight, reps, rpe(null 가능), done } ] }
  ],
  totalVolume }
```

- **부위(bodyPart)**: 가슴/등/어깨/하체/팔/코어/유산소.
- **볼륨** = Σ(유효부하 × reps). 단위 혼재 시 kg로 정규화해 집계.
- 저장 세션의 sets 는 **완료(done) + 유효 세트만** 필터링되어 들어감.

---

## 4. 화면 구조 (하단 5탭)

1. **운동(today)** — 세션 없으면 시작 화면(운동 시작 / 루틴에서 시작 + 이번 주 주간 스트립 + 그날 기록 인라인 + 달력). 세션 중이면 경과타이머 + 종목 카드들 + 종목추가 + 운동종료. `renderToday()`.
2. **회복(recovery)** — 부위별 회복도(근육 도식 + 목록). `renderRecovery()`.
3. **기록(history)** — 날짜별 세션 카드. `renderHistory()`.
4. **루틴(routines)** — 루틴 목록/생성/편집. `renderRoutines()`, `openRoutineEditor()`.
5. **리포트(report)** — 주간 부위별 볼륨/요일 차트/주간 평균. `renderReport()`.

탭 전환은 `switchTab(tab)`. 모달은 `openSheet(html, locked)` / `closeSheet()` 하나의 바텀시트를 재사용(스택 불가).

---

## 5. 핵심 로직

### 5-1. 무게 단위 (종목별)
- 종목마다 `en.unit`("kg"/"lb")를 기억(머신마다 다름). 전역 토글 아님.
- `toKg(w,unit)` / `fromKg(w,unit)`로 변환. 집계·1RM은 항상 kg 기준.

### 5-2. 추정 1RM & 추천 (`recForEntry`, index.html 근처 700~)
- **e1RM**: RPE 있으면 RIR 기반 %1RM 차트(`RPE_CHART`, Zourdos/Helms), 없으면 Epley `w*(1+reps/30)`. `estimate1RM()`.
- **RPE = 10 − 더 할 수 있는 횟수**. 입력 가능 범위 6~10(차트 범위). 비워도 됨.
- **부위별 목표 반복수** `repRangeFor(bodyPart)`: 대근육(가슴/등/하체/코어) **8~12**, 소근육(어깨/팔) **10~15**. `isSmallMuscle()`.
- **증량 단위 자동 추론** `detectInc(en)`: 그 종목에 실제로 입력한 무게들의 GCD로 판단(5kg 스텝→5, 덤벨 2kg→2, lb 머신→5/10). 추천 무게는 이 단위로 반올림.
- **수행 가능 횟수 역산** `repsAtWeight(e1,w)`: 무게로 몇 회까지 가능한지(실패 지점) 계산(RPE10 열 보간 + 고반복은 Epley 역산).
- **다음 세트 추천(파란 💡)**: 기준 세트(완료한 마지막 세트) + 내 최고 1RM(`best1RM`)로 예측.
  - **웜업 보정**: 첫 세트가 내 최고 대비 확연히 약하면(`< best1RM*0.9`) 직전 세트가 아니라 **best1RM 기준**으로 본세트 무게를 추천. (1RM 80인 사람이 웜업 40×10 → 40×8이 아니라 ~55×11)
  - 예측 횟수가 목표 범위 밖이면 증량 단위 한 칸씩 조정(거리 최소화). 어시스트는 방향 반전.
  - `FATIGUE_FACTOR=0.97`로 다음 세트 피로 약간 반영.
- **최고 기록 갱신(주황 🔥)**: 이번 세트 e1RM이 저장된 best1RM을 넘으면.
  - **3대운동(`isBig3`)**: `벤치프레스`/`스쿼트`/`데드리프트` **정확히 이 세 이름만** → "1RM 갱신! X × 3회 도전".
  - **그 외 전부(파생운동 포함)**: "최고 기록! X × (목표하한)회 도전". X = 목표 하한 반복수의 최고 무게(올림).

### 5-3. 어시스트(체중보조) 종목 — `mode:"assist"`
- 딥스 어시스트/풀업 어시스트처럼 **입력 무게 = 덜어주는 보조량**. 유효부하 = `max(0, bodyweight − 입력무게)`. 무게 0 = 순수 체중(유효 세트).
- 모든 부하/볼륨/e1RM은 **`setKg(en,st)`** 헬퍼 하나를 거친다(일반=toKg, 어시스트=체중−보조). 여기저기서 이 함수 사용.
- 종목 카드의 "계산 방식: 일반/어시스트" 토글. 이름에 "어시스트/assist" 있으면 기본 어시스트(`assistDefault`).

### 5-4. 회복도 (`recoveryData`, `renderRecovery`)
- 부위별 회복% = 마지막 운동 후 경과시간 ÷ (부위별 회복시간 × **볼륨 강도 계수**). ≥90% 초록 / ≥70% 노랑 / 그 미만 빨강.
- **볼륨 반영**: 그날 볼륨을 그 부위 평소 볼륨과 비교(`sessionLoadByGroup`)해 회복시간을 0.6~1.6배 조정(빡센 날일수록 느리게).
- **팔은 회복도에서만 이두/삼두로 분리**(`recGroupsFor`: 컬→이두, 익스텐션/푸시다운/딥스 등→삼두, 애매하면 둘 다). 회복 그룹 = 가슴/등/어깨/하체/**이두/삼두**/코어. (종목 분류·루틴·리포트는 여전히 '팔')

### 5-5. 루틴
- 시작 시 각 종목의 세트를 **직전 세션의 세트 그대로** 프리필(`lastSessionEntry`, 세트별 무게·횟수 보존). 기록 없으면 targetSets/Reps 폴백.
- 종료 시 루틴에서 시작했으면 **병합 제안**(`mergeRoutineItems`): 오늘 수행한 종목은 **현재 운동 순서**와 세트 수를 반영하고, 새 종목은 추가한다. **오늘 안 한 기존 종목은 삭제하지 않고 뒤에 유지**한다. 단, 운동 중 종목 카드의 **삭제**를 누른 종목은 루틴에서도 제거한다. 변경 없으면 프롬프트 안 뜸.
- 편집기 `openRoutineEditor`: 드래그 정렬, 종목 추가(picker), 변경 없으면 취소 시 확인 없이 닫힘.

### 5-6. 종료 요약 & 이미지 저장 (`buildSummarySVG`, `saveSummaryImage`)
- 운동한 부위 근육 도식(`muscleFigureSVG`, 정면/후면) + 총볼륨/시간/추정칼로리 카드를 SVG로.
- PNG 저장은 라이브러리 없이 SVG→canvas 래스터화. **SVG는 self-contained(리터럴 hex 색상, CSS 변수/외부폰트 금지)** — 아니면 canvas가 taint되어 실패.

### 5-7. 종목 선택 picker (`openExercisePicker`)
- 카테고리 칩 + 검색 + "직접 추가". 타이핑 datalist 아님(모바일 부적합).
- **내가 추가한 종목**(`SEED_EX`에 없는 것)은 **왼쪽 스와이프로 삭제**(iOS식, `enableSwipeDelete`). 기록 있으면 확인 후 삭제(기록 자체는 보존).

---

## 6. 검증(수정 후 필수)

정적 서버(위 2번)로 띄우고 브라우저 콘솔/DOM으로 확인. 흔한 함정:
- localStorage에 시드 후 **새로고침**해야 인메모리 `sessions`/`active`가 갱신됨.
- 테스트 후 `tl.*` 키 정리.
- 추천/회복 로직은 `document.querySelectorAll('.entry .rec')`, `.recrow` 등에서 결과 문자열로 검증 가능.

---

## 7. 주의해야 할 함정 (Gotchas)

- **전역 `.empty` 클래스 재사용 금지**: `.empty{padding:36px}` 이라서 다른 요소에 쓰면 레이아웃 깨짐. 빈/블랭크 상태는 `.zero`/`.blank` 등 다른 이름 사용.
- **드래그 정렬/스와이프**: Pointer Events + document 레벨 리스너로 구현(`setPointerCapture` 쓰면 DOM 이동 시 캡처 유실로 맥에서 깨짐). 터치는 롱프레스, 마우스는 즉시.
- **스와이프 삭제 버튼**: 행에 `position:relative;z-index:1`을 줘야 평소엔 삭제 버튼을 덮어 가림(안 그러면 항상 노출됨).
- **입력창 캐럿**: `input`에 `line-height` 명시 안 하면 iOS에서 커서가 텍스트와 어긋남.
- **바텀시트 + 키보드**: `visualViewport`로 시트를 키보드 위로 올리고, `focusin` 시 입력창을 화면 안으로 스크롤. 시트 열리면 `body.noscroll`로 배경 스크롤 잠금.
- **집계는 항상 kg**. 표시만 종목 단위. 예전 전역 lb 설정이 남아 "0 lb" 뜨던 버그 때문에 로드시 `settings.unit="kg"` 강제.

---

## 8. 자주 만지는 상수/함수 빠른 참조

| 대상 | 위치(대략) | 설명 |
|---|---|---|
| `APP_VERSION` | script 최상단 | 버전 배지. 배포마다 올릴 것 |
| `RPE_CHART` | 상단 | (reps,RPE)→%1RM 표 |
| `repRangeFor` / `isSmallMuscle` | | 부위별 목표 반복수 |
| `isBig3` | | 3대운동 정확 매칭 |
| `detectInc` / `repsAtWeight` | | 증량단위 추론 / 횟수 역산 |
| `recForEntry` | ~714 | 다음 세트 & 최고기록 추천의 핵심 |
| `setKg` / `isAssist` | | 유효부하(어시스트 포함) |
| `recoveryData` / `recGroupsFor` | ~1576 | 회복도(팔=이두/삼두) |
| `mergeRoutineItems` | ~1232 | 루틴 병합(비파괴적) |
| `startSession` / `endSession` | | 세션 시작/종료 |
| `openSheet` / `closeSheet` | | 바텀시트 |
| `SEED_EX` / `BODY_PARTS` | 상단 | 기본 종목/부위 |

---

## 9. 범위 밖 (현재 안 함)
클라우드 동기화/멀티기기 자동동기화, 체중 추이 그래프, CSV 내보내기, 계정/로그인.
