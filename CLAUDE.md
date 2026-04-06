# North Korean Nuclear Activities Monitor

## Project Purpose
북한 핵활동에 대한 일일 자동 모니터링 및 보고서 생성 시스템.
GitHub Actions에서 Claude Code 에이전트가 매일 실행되어 4단계 파이프라인(수집→태깅→분석→보고서)으로 일일 보고서를 생성한다.

## Pipeline Architecture

```
Phase 1        Phase 2        Phase 3         Phase 4
수집(Collect) → 태깅(Tag)   → 분석(Analyze) → 보고서(Report)
    │              │              │               │
    ▼              ▼              ▼               ▼
search-        sources.       analysis.md     YYYY-MM-DD.md
results.json   json           report-basis.md + Wiki
```

### 디렉토리 구조
```
sources/
  YYYY-MM-DD/
    search-results.json   # Phase 1: 검색 원본 (write-only, 이후 Phase에서 재읽기 않음)
    index.json             # Phase 2: 경량 인덱스 (id, title, url, tag만 포함)
    items/                 # Phase 2: 개별 소스 상세 파일
      src-001.json
      src-002.json
      ...
    analysis.md            # Phase 3: 연관관계 분석
    report-basis.md        # Phase 3: 보고서 작성 근거

reports/
  YYYY/MM/
    YYYY-MM-DD.md          # Phase 4: 최종 보고서
```

### 토큰 효율 설계
- **index.json**은 소스당 ~1줄 요약만 포함 (~2KB). 7일치 읽어도 ~14KB
- 이전 소스 비교 시 `index.json`만 읽고, 필요한 항목만 `items/src-XXX.json` 개별 열람
- `search-results.json`은 Phase 1에서 쓰고 이후 재읽기 않음 (디버깅/추적용)

## Phase 1: 수집 (Collect)

### 검색 방법

**Method 1: WebSearch (Built-in)**
Claude Code의 빌트인 WebSearch 도구로 일반 웹검색을 수행한다.

**Method 2: Cheliped Browser (Site-specific)**
Cheliped CLI (`$CHELIPED_CLI`)를 사용하여 특정 검색 사이트에서 직접 검색한다.
검색 사이트 목록은 `config/search-sites.json`에서 관리한다.

**검색 엔진 (search 커맨드):** Google, Naver, Bing, Baidu, Yahoo Japan, DuckDuckGo
```bash
node $CHELIPED_CLI '[{"cmd":"search","args":["검색어","엔진명"]},{"cmd":"extract","args":["all"]},{"cmd":"close"}]'
```

**커스텀 사이트 (goto + extract):** 38 North, NK News, Reuters, 연합뉴스, NHK, Arms Control Association
```bash
node $CHELIPED_CLI '[{"cmd":"goto","args":["URL"]},{"cmd":"wait","args":["2000"]},{"cmd":"extract","args":["all"]},{"cmd":"close"}]'
```

- 각 검색 후 반드시 `close` 커맨드로 세션을 종료한다
- `config/search-sites.json`에서 `enabled: true`인 사이트만 검색한다

### 검색 키워드

#### Korean (한국어)
북핵, 북한 핵활동, 북한 핵실험, 북한 핵무기, 북한 미사일, 북한 ICBM, 북한 핵폐기물, 북한 핵개발, 북한 우라늄 농축, 북한 플루토늄, 조선 핵, 북한 제재

#### English
North Korean nuclear, DPRK nuclear test, North Korea missile launch, DPRK ICBM, North Korea nuclear weapons, Pyongyang nuclear program, North Korea uranium enrichment, DPRK sanctions, North Korea plutonium, Korean peninsula nuclear

#### Japanese (日本語)
北朝鮮 核実験, 北朝鮮 ミサイル, 北朝鮮 核開発, 北朝鮮 ウラン濃縮, 朝鮮半島 核問題

#### Chinese (中文)
朝鲜核试验, 朝鲜核武器, 朝鲜导弹, 朝鲜半岛核问题, 朝鲜铀浓缩, 朝鲜制裁

### 산출물: search-results.json (write-only)
```json
{
  "date": "YYYY-MM-DD",
  "collected_at": "ISO8601",
  "search_count": 12,
  "total_results": 45,
  "searches": [
    {
      "method": "websearch|cheliped",
      "engine": "google|naver|built-in|...",
      "keyword": "검색어",
      "language": "ko|en|ja|zh",
      "timestamp": "ISO8601",
      "result_count": 5,
      "results": [
        { "title": "...", "url": "...", "snippet": "...", "source_name": "..." }
      ]
    }
  ]
}
```
이 파일은 Phase 1에서 쓰고 이후 Phase에서 재읽기 않는다. 디버깅/추적 전용.

## Phase 2: 태깅 (Tag)

이전 7일의 `sources/*/index.json`만 읽어 URL·제목을 비교하고, 각 소스에 태그를 부여한다.

### 태그 정의
| 태그 | 의미 | 조건 |
|------|------|------|
| `new` | 신규 | 이전 소스에 동일 URL·유사 제목 없음 |
| `reported` | 보고됨 | 이전에 이미 보고된 동일 내용 |
| `update` | 후속보도 | 기존 사건의 후속이지만 유의미한 새 정보 포함 |

### 중복 판단 기준
1. URL 일치 → `reported`
2. 제목 핵심 키워드 80%+ 일치 → `reported`
3. 동일 사건 + 새로운 수치/성명/결과 → `update`

### 산출물 1: index.json (경량 인덱스)
태깅 시 이전 소스 비교용. 소스당 1줄 요약만 포함하여 7일치 읽어도 토큰 부담 최소화.
```json
{
  "date": "YYYY-MM-DD",
  "total": 25,
  "new": 8,
  "reported": 15,
  "update": 2,
  "items": [
    {
      "id": "src-001",
      "title": "뉴스 제목",
      "url": "https://...",
      "tag": "new",
      "related_report": null
    }
  ]
}
```

### 산출물 2: items/src-XXX.json (개별 소스 상세)
각 소스의 전체 정보를 개별 파일로 저장. 필요할 때만 열람.
```json
{
  "id": "src-001",
  "title": "뉴스 제목",
  "url": "https://...",
  "snippet": "기사 요약 또는 발췌",
  "source_name": "연합뉴스",
  "language": "ko",
  "discovered_date": "YYYY-MM-DD",
  "tag": "new",
  "related_report": null,
  "related_item": null,
  "tag_reason": "이전 7일 소스에 동일/유사 항목 없음"
}
```

## Phase 3: 분석 (Analyze)

### 산출물 1: analysis.md (연관관계 분석)
- 신규 소스 분석: 중요도(높음/중간/낮음), 카테고리, 이전 보고서 연관
- 기존 보도 추적: 원본 보고서, 변경/추가 정보, 추적 가치
- 제외 소스: 원본 참조, 제외 근거
- 주제별 흐름 분석: 최근 7일 동향 + 오늘 새 정보

### 산출물 2: report-basis.md (보고서 작성 근거)
- 포함 항목 테이블: 소스 ID, 제목, 태그, 카테고리, 포함 근거
- 제외 항목 테이블: 소스 ID, 제목, 제외 근거
- 보고서 구성 방향: 요약 방향, 분석 초점, 추적 항목

## Phase 4: 보고서 (Report)

### Report Structure
보고서는 `reports/YYYY/MM/YYYY-MM-DD.md` 경로에 저장된다.

필수 구조:
1. YAML frontmatter (title, date, sources_count, new_items, updated_items)
2. 요약 (3-5문장)
3. 주요 뉴스 (출처 URL 필수, 상태: 신규/업데이트)
4. 분석 및 평가
5. 국제 반응
6. 동향 요약 테이블
7. 출처 목록

### 업데이트 항목 표기
```
- **상태:** 업데이트 (← YYYY-MM-DD "이전 항목 제목")
```

## Wiki Publishing Rules
- GitHub Wiki는 YAML frontmatter(`---`)를 지원하지 않는다
- Wiki에 복사할 보고서에서는 YAML frontmatter 블록을 반드시 제거한다
- 메인 리포의 `reports/` 파일에는 frontmatter를 그대로 유지한다

## Commit Convention
- 보고서 커밋: `report: daily NK nuclear update (YYYY-MM-DD)`
- 구조/설정 변경: `chore: 설명`
- `git add sources/ reports/` — sources 디렉토리도 함께 커밋한다

## Rules
- 출처 URL 없는 정보는 보고서에 포함하지 않는다
- 보고서 언어는 한국어가 기본이며, 원문 인용은 원어 보존한다
- 동일 뉴스가 여러 매체에서 보도된 경우 대표 1개만 포함하되 출처 목록에 모두 기재한다
- 보고서가 비어있더라도 파일은 생성한다 (특이사항 없음 보고서)
- 파이프라인 중간 산출물(search-results.json, sources.json, analysis.md, report-basis.md)은 항상 생성한다
