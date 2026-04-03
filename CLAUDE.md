# North Korean Nuclear Activities Monitor

## Project Purpose
북한 핵활동에 대한 일일 자동 모니터링 및 보고서 생성 시스템.
GitHub Actions에서 Claude Code 에이전트가 매일 실행되어 다국어 웹검색 → 중복제거 → 보고서 생성 → 자동 커밋을 수행한다.

## Report Structure
보고서는 `reports/YYYY/MM/YYYY-MM-DD.md` 경로에 저장된다.

각 보고서의 필수 구조:
1. YAML frontmatter (title, date, sources_count, new_items, updated_items)
2. 요약 (3-5문장)
3. 주요 뉴스 (출처 URL 필수)
4. 분석 및 평가
5. 국제 반응
6. 동향 요약 테이블
7. 출처 목록

## Deduplication Rules
- 새 보고서 작성 전 반드시 이전 7일 보고서를 읽는다
- 이미 보고된 사건과 동일한 내용은 제외한다
- 판단 기준: 뉴스 제목 유사도 + 출처 URL 일치
- 기존 사건의 후속 보도는 "업데이트" 표시로 포함 가능
- 특이사항 없으면 "금일 북한 핵활동 관련 특이사항 없음" 기재

## Search Keywords

### Korean (한국어)
북핵, 북한 핵활동, 북한 핵실험, 북한 핵무기, 북한 미사일, 북한 ICBM, 북한 핵폐기물, 북한 핵개발, 북한 우라늄 농축, 북한 플루토늄, 조선 핵, 북한 제재

### English
North Korean nuclear, DPRK nuclear test, North Korea missile launch, DPRK ICBM, North Korea nuclear weapons, Pyongyang nuclear program, North Korea uranium enrichment, DPRK sanctions, North Korea plutonium, Korean peninsula nuclear

### Japanese (日本語)
北朝鮮 核実験, 北朝鮮 ミサイル, 北朝鮮 核開発, 北朝鮮 ウラン濃縮, 朝鮮半島 核問題

### Chinese (中文)
朝鲜核试验, 朝鲜核武器, 朝鲜导弹, 朝鲜半岛核问题, 朝鲜铀浓缩, 朝鲜制裁

## Search Methods

### Method 1: WebSearch (Built-in)
Claude Code의 빌트인 WebSearch 도구로 일반 웹검색을 수행한다.

### Method 2: Cheliped Browser (Site-specific)
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
- 검색 결과의 links와 texts를 수집하여 분석한다

## Commit Convention
- 보고서 커밋: `report: daily NK nuclear update (YYYY-MM-DD)`
- 구조/설정 변경: `chore: 설명`

## Rules
- 출처 URL 없는 정보는 보고서에 포함하지 않는다
- 보고서 언어는 한국어가 기본이며, 원문 인용은 원어 보존한다
- 동일 뉴스가 여러 매체에서 보도된 경우 대표 1개만 포함하되 출처 목록에 모두 기재한다
- 보고서가 비어있더라도 파일은 생성한다 (특이사항 없음 보고서)
