---
name: nk-nuclear-report
description: "북한 핵활동 일일 보고서를 자동 생성하는 파이프라인 오케스트레이터. 4단계(수집→태깅→분석→보고서)로 구성되며, 각 단계의 산출물을 sources/ 폴더에 파일로 저장하여 전 과정을 추적할 수 있다. '북한 핵 보고서', 'NK nuclear report', '일일 보고서 생성', 'daily report' 요청 시 사용."
---

# NK Nuclear Report — 파이프라인 오케스트레이터

이 스킬은 4개 에이전트를 순차적으로 연결하여 일일 보고서를 생성한다.
각 에이전트는 자신의 역할(`.claude/agents/`)과 참조 스킬(`references/`)을 읽고 작업한다.

## 실행 흐름

### Phase 0: 준비
1. 대상 날짜 결정
2. `mkdir -p sources/YYYY-MM-DD/items`
3. `mkdir -p reports/YYYY/MM`
4. 이전 7일의 `sources/*/index.json` 파일 목록 조회
5. 이전 7일의 `reports/YYYY/MM/*.md` 파일 목록 조회

### Phase 1: 수집 (Collect)
**에이전트:** `.claude/agents/nk-collector.md`를 읽고 역할을 따른다
**참조 스킬:** `references/search-strategy.md`를 읽고 검색 키워드·방법·스키마를 따른다
**산출물:** `sources/YYYY-MM-DD/search-results.json`

### Phase 2: 태깅 (Tag)
**에이전트:** `.claude/agents/nk-tagger.md`를 읽고 역할을 따른다
**참조 스킬:** `references/tagging-rules.md`를 읽고 태그 정의·판단 기준·스키마를 따른다
**산출물:** `sources/YYYY-MM-DD/index.json` + `sources/YYYY-MM-DD/items/src-XXX.json`

### Phase 3: 분석 (Analyze)
**에이전트:** `.claude/agents/nk-analyst.md`를 읽고 역할을 따른다
**참조 스킬:** `references/analysis-guide.md`를 읽고 분석 원칙·출력 형식을 따른다
**산출물:** `sources/YYYY-MM-DD/analysis.md` + `sources/YYYY-MM-DD/report-basis.md`

### Phase 4: 보고서 (Report)
**에이전트:** `.claude/agents/nk-reporter.md`를 읽고 역할을 따른다
**참조 스킬:** `references/report-format.md`를 읽고 보고서 구조·Wiki 규칙을 따른다
**산출물:** `reports/YYYY/MM/YYYY-MM-DD.md` + Wiki 페이지

### Phase 5: 커밋 & 발행
```bash
git add sources/ reports/
git commit -m "report: daily NK nuclear update (YYYY-MM-DD)"
git push
```

Wiki 발행:
- wiki.git clone → 보고서 복사 (frontmatter 제거) → Home.md, _Sidebar.md 업데이트 → push

## 에러 핸들링

| Phase | 에러 | 전략 |
|-------|------|------|
| 1 | 검색 전체 실패 | 빈 search-results.json, 다음 Phase 계속 |
| 2 | 이전 index.json 없음 | 전체 `new`로 태깅 |
| 3 | 이전 보고서 없음 | 신규 소스만으로 분석 |
| 4 | 포함 항목 0건 | "특이사항 없음" 보고서 |
| 5 | Wiki clone 실패 | 메인 리포 커밋 유지, Wiki 스킵 |
