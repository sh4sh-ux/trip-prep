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
**v0.4** (2026-08-05) — Dropbox 동기화 + PWA 배포 준비

## 탭 구조
- **오늘** — D-day, 콜드체인 배너, Day 1·2·3 전환, 날짜별 일정, 그 날 메뉴
- **준비물** — 카테고리별 체크리스트, 진행률
- **장보기** — `메뉴 고르기` / `장볼 것` 세그먼트
- ⋯ 메뉴 — Dropbox / 여행 정보 수정 / 여행 목록 / JSON 백업·불러오기 / 여행 삭제

## 데이터 구조
```js
DB = {
  v:1, cur:<tripId>,
  library:[ {name,emoji,ing:[{n,q,u,c}],people,at} ],   // 내 사전 (여행 간 재사용)
  trips:[{
    id,name,dest,stay,type,dateStart,dateEnd,people,createdAt,
    pack:[{id,cat,name,qty,auto,done}],
    menus:[{id,name,emoji,day,picked,ing:[{n,q,u,c}]}],
    bought:[<"이름|단위" key>], extras:[{n,q,u,c}],
    plans:[{id,day,time,title,note}]
  }]
}
// localStorage key: 'tripprep_v1'
```

## 핵심 설계 원칙
- **준비물 수량은 자동 계산** — 옷은 여행 길이(`n박n+1일`), 칫솔·수건·수영복은 인원수
  자동 계산된 항목에는 근거를 그대로 노출 (`2박3일 기준 자동`, `4명 기준 자동`)
- **여행 유형(국내/해외/캠핑/출장)에 따라 항목이 달라짐** — `only:[...]`로 제어
  해외→여권·변환어댑터, 국내→자동차키, 출장→수영용품 카테고리 통째로 제외
- **아이스박스 카테고리는 자동 연동** — 장보기에 콜드(정육·수산·냉장) 항목이 생기면 붙고,
  없어지면 사라짐 (단, 이미 체크한 항목이 있으면 유지) → `syncIcebox()`
- **레시피 사전은 1인분 기준** — 불러올 때 여행 인원만큼 환산(`scaleIng`)
  양념·기름은 `=` 접두사로 고정 수량 처리 (인원 늘어도 한 통)
- **내 사전이 내장 레시피를 이김** — 이름이 같으면 사용자가 고친 버전이 우선
  앱 업데이트로 내장 사전이 바뀌어도 사용자 수정본은 보존
- 삭제는 전부 '되돌리기' 토스트 제공
- 저장 버튼 없음 — 변경 즉시 `save()` (Dropbox는 3초 디바운스 후 업로드)

## 주요 함수
| 함수 | 역할 |
|------|------|
| `render()` | 전체 재렌더 (탭 3개를 모두 그리고 CSS로 표시 전환) |
| `viewToday()` / `viewPack()` / `viewMarket()` | 탭별 HTML 생성 |
| `genPack(trip)` | 여행 유형·박수·인원으로 준비물 자동 생성 |
| `mergeMarket(trip)` | 선택된 메뉴들의 재료를 `이름\|단위` 기준으로 합산 |
| `syncIcebox(trip)` | 콜드 항목 유무에 따라 아이스박스 카테고리 자동 추가/제거 |
| `searchRecipes(q)` | 내 사전 + 내장 레시피 검색 (내 사전 우선) |
| `scaleIng(ing,people)` | 1인분 재료 → 인원수 환산 (고정 항목 제외) |
| `dbxPull/dbxPush` | Dropbox 동기화 |
| `migrate(db)` | 구버전 데이터 보정 (카테고리 이름 변경 등) |

## 레시피 사전
- `RECIPES` 배열 — `[메뉴명, 아이콘, '재료,수량단위,분류|...']`, **104종**
- 수량 앞 `=` 는 인원수 무관 고정 (예: `고추장,=1통,소스·양념`)
- 장보기 분류: 채소 / 정육 / 수산 / 냉장 / 소스·양념 / 기타
- 콜드체인 대상: 정육 · 수산 · 냉장

## Dropbox 연동
- 저장 경로: `/01_Personal/Trip-Prep/trip-prep_data.json` (한글 경로 회피 — 더치페이 v1.21 교훈)
- 리다이렉트 URI: `https://sh4sh-ux.github.io/trip-prep/history.html`
- App Key는 하드코딩하지 않고 사용자가 앱에서 입력 → localStorage 저장
- PKCE(S256) + refresh token 방식
- 원격이 더 최신이면 로컬을 덮어쓰되, 직전 상태를 `tripprep_backup_presync`에 보관

## 배포 시 주의
- 코드 변경 후 `BUILD_TIME`, `VERSION` 갱신
- **`sw.js`의 `CACHE` 버전도 같이 올릴 것** — 안 올리면 구버전이 계속 나옴
- 커밋은 변경된 파일만 명시적으로 추가 (`git add -A` 사용 금지)

## 작업 규칙
- **실행 전 반드시 생각/계획을 먼저 말하고 확인 후 진행**
- 시각적 변경은 미리보기로 확인받은 뒤 배포
- 버전은 변경마다 올림
