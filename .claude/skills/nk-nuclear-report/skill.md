---
name: nk-nuclear-report
description: "북한 핵활동 일일 보고서를 자동 생성하는 파이프라인 오케스트레이터. 4단계(수집→태깅→분석→보고서)로 구성되며, 각 단계의 산출물을 sources/ 폴더에 파일로 저장하여 전 과정을 추적할 수 있다. '북한 핵 보고서', 'NK nuclear report', '일일 보고서 생성', 'daily report' 요청 시 사용."
---

# NK Nuclear Report — 4단계 파이프라인 오케스트레이터

## 아키텍처

파이프라인 패턴. 각 단계는 이전 단계의 파일 산출물을 읽고, 자신의 산출물을 파일로 저장한다.

```
Phase 1        Phase 2        Phase 3         Phase 4
수집(Collect) → 태깅(Tag)   → 분석(Analyze) → 보고서(Report)
    │              │              │               │
    ▼              ▼              ▼               ▼
search-        sources.       analysis.md     reports/YYYY/MM/
results.json   json           report-         YYYY-MM-DD.md
                              basis.md        + Wiki 발행
```

모든 중간 산출물은 `sources/YYYY-MM-DD/` 에 저장된다.
개별 소스는 `items/` 하위에 파일별로 분리하여 토큰 효율을 확보한다.
이전 소스 비교 시 경량 `index.json`만 읽고, 필요한 항목만 개별 열람한다.

## 실행 흐름

### Phase 0: 준비
1. 대상 날짜 결정 (입력값 또는 오늘 날짜)
2. `mkdir -p sources/YYYY-MM-DD/`
3. `mkdir -p reports/YYYY/MM/`
4. 이전 7일의 `sources/*/index.json` 파일 목록 조회
5. 이전 7일의 `reports/YYYY/MM/*.md` 파일 목록 조회

### Phase 1: 수집 (Collect)
**에이전트:** nk-collector
**산출물:** `sources/YYYY-MM-DD/search-results.json`

1. `config/search-sites.json` 읽기
2. WebSearch(빌트인)로 다국어 검색 수행:
   - 한국어 (최소 3회): "북핵 최신 뉴스", "북한 핵실험", "북한 미사일 발사"
   - 영어 (최소 3회): "North Korean nuclear news today", "DPRK missile launch", "North Korea sanctions update"
   - 일본어 (최소 1회): "北朝鮮 核実験 最新ニュース"
   - 중국어 (최소 1회): "朝鲜核试验 最新消息"
3. Cheliped Browser로 사이트별 검색:
   - 검색 엔진: `search` 커맨드 (Google, Naver, Bing, Baidu, Yahoo Japan, DuckDuckGo)
   - 커스텀 사이트: `goto` + `extract` (38 North, NK News, Reuters, 연합뉴스, NHK, ACA)
4. 수집된 URL 단위로 1차 중복 제거
5. **결과를 `sources/YYYY-MM-DD/search-results.json`에 저장**

### Phase 2: 태깅 (Tag)
**에이전트:** nk-tagger
**입력:** `search-results.json` + 이전 7일 `index.json`
**산출물:** `sources/YYYY-MM-DD/index.json` + `sources/YYYY-MM-DD/items/src-XXX.json`

1. `search-results.json` 읽기
2. 이전 7일의 `sources/*/index.json`만 읽기 (경량 — 토큰 절약)
3. 각 소스에 태그 부여:
   - `new`: 이전 소스에 없는 완전 신규
   - `reported`: 이전에 이미 보고된 동일 내용
   - `update`: 기존 사건의 후속 보도 (유의미한 새 정보 포함)
4. 각 태그에 근거(tag_reason) 기록 → 개별 items 파일에 저장
5. `reported`/`update` 태그에는 관련 보고서 날짜와 항목명 기록
6. **`sources/YYYY-MM-DD/index.json`에 경량 인덱스 저장** (id, title, url, tag, related_report만)
7. **`sources/YYYY-MM-DD/items/src-XXX.json`에 개별 소스 상세 저장** (snippet, tag_reason 등 포함)

### Phase 3: 분석 (Analyze)
**에이전트:** nk-analyst
**입력:** `index.json` + `new`/`update` 항목의 개별 `items/src-XXX.json` + 이전 보고서(`reports/`)
**산출물:** `sources/YYYY-MM-DD/analysis.md`, `sources/YYYY-MM-DD/report-basis.md`

1. `index.json`에서 전체 태깅 현황 파악
2. `new`/`update` 태그 항목만 `items/src-XXX.json` 개별 열람 (`reported` 항목은 읽지 않음)
3. 이전 7일 보고서 읽기
4. 연관관계 분석:
   - 신규 소스의 중요도 평가 (높음/중간/낮음)
   - 카테고리 분류 (핵실험/미사일/우라늄농축/외교·제재/군사력/기타)
   - 기존 보도 추적 가치 판단
   - 주제별 흐름 분석 (최근 7일 동향 + 오늘 새 정보)
5. **`sources/YYYY-MM-DD/analysis.md`에 연관관계 분석 저장**
6. 보고서 포함/제외 결정:
   - 포함 항목: 소스 ID, 제목, 태그, 카테고리, 포함 근거
   - 제외 항목: 소스 ID, 제목, 제외 근거
   - 보고서 구성 방향: 요약 방향, 분석 초점, 추적 항목
7. **`sources/YYYY-MM-DD/report-basis.md`에 작성 근거 저장**

### Phase 4: 보고서 (Report)
**에이전트:** nk-reporter
**입력:** `sources.json` + `analysis.md` + `report-basis.md`
**산출물:** `reports/YYYY/MM/YYYY-MM-DD.md` + Wiki 페이지

1. `report-basis.md`의 포함 결정에 따라 보고서 작성
2. `analysis.md`의 연관관계로 추적 항목 표기
3. `sources.json`에서 정확한 출처 정보 인용
4. 보고서 형식 (CLAUDE.md 표준):
   - YAML frontmatter (title, date, sources_count, new_items, updated_items)
   - 요약 (3-5문장)
   - 주요 뉴스 (출처 URL, 일시, 내용, 상태: 신규/업데이트)
   - 분석 및 평가
   - 국제 반응
   - 동향 요약 테이블
   - 출처 목록
5. **`reports/YYYY/MM/YYYY-MM-DD.md`에 보고서 저장**
6. 메인 리포 커밋:
   ```bash
   git add sources/ reports/
   git commit -m "report: daily NK nuclear update (YYYY-MM-DD)"
   git push
   ```
7. Wiki 발행:
   - Wiki 리포 clone
   - 보고서 복사 (YAML frontmatter 제거)
   - Home.md, _Sidebar.md, Monthly 인덱스 업데이트
   - Wiki push

## 데이터 흐름 요약

```
sources/YYYY-MM-DD/
├── search-results.json   ← Phase 1 (write-only, 이후 재읽기 않음)
├── index.json             ← Phase 2 (경량 인덱스 ~2KB, 비교용)
├── items/                 ← Phase 2 (개별 소스 상세)
│   ├── src-001.json
│   ├── src-002.json
│   └── ...
├── analysis.md            ← Phase 3 (연관관계)
└── report-basis.md        ← Phase 3 (작성 근거)

reports/YYYY/MM/
└── YYYY-MM-DD.md          ← Phase 4 (최종 보고서)
```

### 토큰 효율
- Phase 2: 이전 7일 `index.json`만 읽음 (~14KB) ← 기존 sources.json 7일치 대비 1/10
- Phase 3: `new`/`update` 항목만 개별 `items/` 파일 열람 ← `reported` 항목은 읽지 않음
- Phase 4: report-basis.md의 포함 항목만 `items/` 파일 열람

## 에러 핸들링

| Phase | 에러 | 전략 |
|-------|------|------|
| 1 | 검색 전체 실패 | 빈 search-results.json 생성, Phase 2~4 계속 진행 |
| 1 | 일부 검색 실패 | 성공한 결과만으로 진행, search-results.json에 에러 기록 |
| 2 | 이전 sources.json 없음 (첫 실행) | 전체 결과를 `new`로 태깅 |
| 3 | 이전 보고서 없음 | 연관관계 없이 신규 소스만으로 분석 |
| 4 | 포함 항목 0건 | "특이사항 없음" 보고서 생성 |
| 4 | Wiki clone 실패 | 메인 리포 커밋은 유지, Wiki 발행만 스킵 |

## 테스트 시나리오

### 정상 흐름
1. workflow_dispatch로 수동 트리거
2. `sources/YYYY-MM-DD/` 디렉토리에 4개 파일 생성 확인
3. `search-results.json`에 최소 8개 검색 기록 존재
4. `sources.json`에 태그와 tag_reason이 모든 항목에 존재
5. `analysis.md`에 신규/추적/제외 섹션 존재
6. `report-basis.md`에 포함/제외 테이블 존재
7. 보고서에 출처 URL 최소 1개 포함
8. git commit 메시지 형식 확인

### 2일 연속 실행 (중복 제거 검증)
1. 첫째날: 모든 소스가 `new` 태그
2. 둘째날: 첫째날과 동일한 소스는 `reported` 태그
3. 둘째날 보고서에 첫째날 뉴스 반복 없음
4. 후속 보도만 `update` 태그로 보고서에 포함

### 검색 결과 없음
1. 모든 검색 0건 반환
2. `search-results.json`에 빈 results 배열
3. `sources.json`에 items 0건
4. "특이사항 없음" 보고서 정상 생성
