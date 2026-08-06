# 여행 준비 Trip Prep

## 프로젝트 개요
일정 · 준비물 · 장보기를 여행 하나로 묶어 관리하는 모바일 우선 PWA.
단일 HTML 파일(`index.html`)로 구성. 소유자(조상현)가 혼자 사용.
핵심 아이디어: **메뉴를 고르면 재료가 자동으로 합쳐진 장보기 목록이 나온다.**

## 라이브 URL / 저장소
- **라이브:** https://sh4sh-ux.github.io/trip-prep/
- **GitHub:** git@github.com:sh4sh-ux/trip-prep.git (SSH)
- **로컬:** `/Users/sanghyeon/Claude/Trip Prep/`
- **브랜치:** main
- **로컬 미리보기:** Dutch Pay 쪽 `.claude/launch.json`의 `trip-prep` 설정 (python http.server :8765)

## 파일 구조
```
index.html            — 앱 전체 (로직·데이터·스타일 포함)
history.html          — Dropbox OAuth 콜백 리다이렉터
mockup-travel.html    — 초기 디자인 목업 (참고용, 배포와 무관)
sw.js                 — 서비스워커 (HTML 네트워크 우선 / 정적자원 캐시 우선)
manifest.webmanifest  — PWA 매니페스트
favicon.png / icons/  — 아이콘 (PIL로 생성, 파란 배경 흰 캐리어)
CLAUDE.md             — 이 파일
```

## 현재 버전
**v0.31** (2026-08-06) — 메뉴 고르기 정리: 담은 메뉴 위로 모으고, 안 담은 메뉴는 `담을 메뉴 더 보기 N개`로 접기

## 탭 구조
- **오늘** — D-day 히어로, 콜드체인 배너, Day 칩 전환, 날짜별 일정 타임라인, 그 날 메뉴
  - 출발일(Day 1)에 `🏠 출발 전 집 체크`, 마지막 날에 `🧳 떠나기 전 숙소 체크` 자동 표시 (`quickCheckCard`, `t.chk`)
- **준비물** — 카테고리 아코디언 체크리스트, 진행률, 순서 바꾸기 모드(▲▼)
- **장보기** — `메뉴 고르기`(행 리스트: ○ 담기 · 이름 눌러 묶음 보기) / `장볼 것`(매대별 아코디언 + 전체 접기/펼치기)
  - **메뉴 고르기 정리(v0.31)** — 담은 메뉴가 위로 모이고, 안 담은 메뉴는 `담을 메뉴 더 보기 N개` 토글(`#menuMoreToggle`, `UI.menuMoreOpen`)로 접힘(옅은 배경 `menu-more-body`). **담은 게 하나도 없으면 자동으로 전부 펼쳐** 첫 선택을 돕는다. sec-note에 담은 개수 표시
  - **메뉴별 묶음 보기**(`sheetMenuBundle`) — 한 메뉴에 필요한 것을 조리도구/소모품/재료 3그룹으로.
    장보기의 '매대별 합산'과 반대 시선("이 메뉴 한 상 차리려면?"). **진입: 장보기 메뉴 행의 이름 탭(`data-menubundle`), 또는 오늘 탭의 그 날 메뉴 행 탭(`data-openmenu`).** 왼쪽 동그라미(`data-menu`)는 담기/빼기 토글.
    체크는 준비물(도구 done)·장보기(재료 bought) 상태를 반영·토글. 소모품 분류는 `CONSUMABLES` 집합
    - **묶음 시트에서 바로 CRUD**(`bundleItemForm`) — 각 행에 ×(삭제·되돌리기 토스트), 이름 탭(편집), 그룹마다 `+ 재료/도구 추가`.
      재료 추가·편집은 이름만 넣어도 `ingHint`로 분류·단위 자동(이름 입력 시 분류 칩 자동 선택). 저장/취소하면 묶음으로 복귀. 변경은 `menu.ing`/`menu.gear`에 반영하고 `saveToLibrary`로 내 사전 동기화
- ⋯ 메뉴 — Dropbox / 여행 정보 수정 / 여행 목록 / JSON 백업·불러오기 / 여행 삭제
- 여행 이름 탭 → 여행 목록 시트 (전환 + 연필로 정보 수정)

## 데이터 구조
```js
DB = {
  v:1, cur:<tripId>,
  library:[ {name,emoji,ing:[{n,q,u,c}],gear:[..],people,at} ],  // 내 사전 (여행 간 재사용)
  packTemplate:[{cat,name}],      // 내 기본 준비물 (없으면 앱 기본 PACK_TPL 사용)
  packTemplateType:'domestic',    // 템플릿 저장 당시 여행 유형
  catOrder:[..],                  // 준비물 카테고리 순서 (전역 설정)
  trips:[{
    id,name,stay,type,dateStart,dateEnd,people,createdAt,
    pack:[{id,cat,name,qty,auto,gearAuto?,done}],   // 배열 순서 = 표시 순서
    menus:[{id,name,emoji,day,picked,ing:[{n,q,u,c}],gear:[..]}],
    bought:[<"이름|단위" key>], extras:[{n,q,u,c}],
    overrides:{key:{q,u,c}}, excluded:[key],        // 장보기 개별 수정·제외
    plans:[{id,day,time,end,title,note}],
    chk:{dep:[label],ret:[label]}                   // 출발/귀가 체크 상태
  }]
}
// localStorage key: 'tripprep_v1'
```

## 핵심 설계 원칙
- **준비물 수량 자동 계산** — 옷은 여행 길이, 칫솔·수건·수영복은 인원수 (`NAME_RULE`)
  자동 항목은 `auto` 필드에 근거 표시(`1박2일`, `6명`). 사용자가 수량을 직접 고치면 `auto=''`
  → 여행 인원·날짜 수정 시 `auto`가 남은 항목만 재계산 (직접 수정·체크 상태 보존)
- **여행 유형(국내/해외/캠핑/출장)별 항목** — `only:[...]` / `NAME_ONLY`
  해외→여권·변환어댑터, 국내→자동차키, 출장→수영용품 카테고리 제외
- **내 기본 준비물** — 준비물 탭 하단 버튼으로 현재 목록을 `packTemplate`으로 저장
  이후 새 여행은 이 목록으로 생성. 유형이 다르면 유형 전용 항목 자동 보충/제외
  (저장 당시 유형 `packTemplateType`에 없던 항목만 보충 — 사용자가 지운 항목 부활 방지)
  아이스박스·조리도구는 자동 연동 카테고리라 템플릿에서 제외
- **아이스박스 자동 연동** — 장보기에 콜드(정육·수산·냉장) 항목 생기면 붙고 없어지면 제거 (`syncIcebox`)
- **조리도구 자동 연동** — 선택된 메뉴들의 `gear` 합집합이 준비물 '조리도구'로 (`syncGear`, 공유 도구는 1개)
  `GEAR_MAP`에 내장 레시피별 도구. 버너 쓰는 메뉴는 부탄가스를 장보기 재료로 자동 추가 (소모품은 장보기 = A안)
- **sync 함수는 render()에서 호출되며 변경 시 즉시 save** (반영 지연 버그 방지)
- **레시피 사전 1인분 기준** — 불러올 때 인원수 환산(`scaleIng`), `=` 접두사는 고정 수량(양념·기름)
- **재료 입력 스마트 보정**(`parseIng`+`ingHint`) — 이름만 써도 분류·단위 자동. `이름`/`이름 300g`/`이름, 2개`/`이름, 2개, 채소` 모두 허용.
  분류·단위 힌트는 내장 사전 `ING_DICT`(RECIPES 학습) 우선, 없으면 내 사전(`DB.library` 과거 입력)에서 학습. 직접 지정한 유효 분류가 있으면 그것 우선
- **내 사전이 내장 레시피를 이김** — 같은 이름이면 사용자 수정본 우선. 메뉴 저장·수정 시 자동으로 library에 반영
- **장보기 매대**: 채소/정육/수산/냉장/소스·양념/기타/**집에서 챙기기**(🏠, 사는 게 아니라 가져가는 것)
  콜드체인 대상: 정육·수산·냉장. 합산 출처 표시(`합산 · 삼겹살 · 떡볶이`), 개별 수정은 `overrides`, 제외는 `excluded`
- **공용 양념 최댓값 합산** — 레시피의 `=`(고정 수량) 재료 이름을 모은 `STAPLES` 집합.
  이 이름들은 여러 메뉴에 걸쳐도 합산하지 않고 **최댓값(1통이면 충분)**만 필요 → 고추장 3메뉴여도 1통.
  화면엔 `합산` 대신 `공용` 표시. `STAPLES`는 앱 시작 시 `RECIPES`에서 자동 생성(어느 레시피서든 인원 비례로 쓰이는 이름은 오탐 방지 위해 제외)
- **일정** — 제목·장소 중 하나만 있으면 저장(장소만 넣으면 제목처럼 표시). 종료 시간 선택.
  숙박 모드: 체크인/체크아웃 두 일정을 두 날짜에 자동 배치 (기본 15:00/11:00)
- **날짜·시간 피커는 앱 자체 시트** (`openPicker`) — 달력 + 시간 휠(스크롤)+직접입력. 네이티브 date/time input 사용 안 함
- **장소 검색** — Daum 우편번호(차용증 앱과 동일, lazy load). 해외 여행이면 버튼 숨김
- 삭제는 전부 '되돌리기' 토스트. 저장 버튼 없음 — 변경 즉시 save (Dropbox는 3초 디바운스)
- **토스트는 시트가 열려 있으면 화면 위로** (`positionToast`) — 하단 버튼 가림 방지 (v0.9 버그의 교훈)

## 주요 함수
| 함수 | 역할 |
|------|------|
| `render()` | 전체 재렌더 (탭 3개 모두 그리고 CSS로 전환, sync 후 필요 시 save) |
| `viewToday/viewPack/viewMarket` | 탭별 HTML |
| `genPack(t)` | 준비물 생성 (packTemplate 있으면 그것 + 유형 보정) |
| `savePackTemplate(t)` | 내 기본 준비물 저장 |
| `mergeMarket(t)` | 메뉴 재료 합산(공용 양념 `STAPLES`는 최댓값) + overrides/excluded 적용 |
| `syncIcebox / syncGear` | 아이스박스·조리도구 자동 연동 (변경 시 true 반환) |
| `searchRecipes(q)` | 내 사전 + 내장 레시피 검색 |
| `sheetPlan(id)` | 일정 추가/수정 (일반/숙박 모드) |
| `sheetEditBuy(key)` | 장보기 항목 편집 (이름·수량·분류·연결 메뉴) |
| `sheetImportMenus()` | 지난 여행에서 메뉴 복사 |
| `sheetMenuBundle(id)` | 메뉴별 묶음 보기 (조리도구·소모품·재료를 한 화면에) |
| `openPicker(opts)` | 날짜·시간 피커 (sheet2 레이어) |
| `openPlaceSearch(cb)` | Daum 장소 검색 오버레이 |
| `dbxPull/dbxPush` | Dropbox 동기화 |
| `migrate(db)` | 구버전 데이터 보정 (CAT_RENAME 등) |

## 레시피 사전
- `RECIPES` — `[메뉴명, 아이콘, '재료,수량단위,분류|...']`, **104종**, 1인분 기준
- 카테고리 이름 변경 시 반드시 `CAT_RENAME`에 옛→새 매핑 추가 (기존 데이터 마이그레이션)

## Dropbox 연동
- 저장 경로: `/01_Personal/Trip-Prep/trip-prep_data.json` (한글 경로 회피)
- 리다이렉트 URI: `https://sh4sh-ux.github.io/trip-prep/history.html`
- App Key는 하드코딩 안 함 — 사용자가 앱에서 입력(⋯ → Dropbox), PKCE(S256)+refresh token
- 원격이 최신이면 로컬 덮어쓰되 직전 상태를 `tripprep_backup_presync`에 보관

## 배포 시 주의
- 코드 변경마다 `VERSION`·`BUILD_TIME` 갱신 + **`sw.js`의 `CACHE` 버전도 같이 올릴 것**
- 커밋은 변경된 파일만 명시적으로 추가 (`git add -A` 금지)
- 브라우저 검증 시 테스트 데이터는 `localStorage.removeItem('tripprep_v1')`로 정리 후 커밋

## 작업 규칙
- **실행 전 반드시 생각/계획을 먼저 말하고 확인 후 진행**
- 시각적 변경은 미리보기로 확인받은 뒤 배포
- 버전은 변경마다 올림
