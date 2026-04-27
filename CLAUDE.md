# North Korean Nuclear Activities Monitor

## Project Purpose
북한 핵활동에 대한 일일 자동 모니터링 및 보고서 생성 시스템.
4단계 파이프라인(수집→태깅→분석→보고서)으로 추적 가능한 일일 보고서를 생성한다.

## Pipeline Architecture

```
Phase 1        Phase 2        Phase 3         Phase 4
수집(Collect) → 태깅(Tag)   → 분석(Analyze) → 보고서(Report)
    │              │              │               │
    ▼              ▼              ▼               ▼
search-        index.json     analysis.md     YYYY-MM-DD.md
results.json   items/         report-basis.md + Wiki
```

## Directory Structure

```
sources/YYYY-MM-DD/
├── search-results.json   # Phase 1 (write-only)
├── index.json             # Phase 2 (경량 인덱스 ~2KB)
├── items/src-XXX.json     # Phase 2 (개별 소스 상세)
├── analysis.md            # Phase 3 (연관관계)
└── report-basis.md        # Phase 3 (작성 근거)

reports/YYYY/MM/
└── YYYY-MM-DD.md          # Phase 4 (최종 보고서)
```

## Agents & Skills

에이전트(역할)와 스킬(상세 규칙)이 분리되어 있다. 로컬과 GHA에서 동일한 파일을 사용한다.

| 에이전트 | 참조 스킬 | Phase |
|----------|----------|-------|
| `.claude/agents/nk-collector.md` | `references/search-strategy.md` | 1 |
| `.claude/agents/nk-tagger.md` | `references/tagging-rules.md` | 2 |
| `.claude/agents/nk-analyst.md` | `references/analysis-guide.md` | 3 |
| `.claude/agents/nk-reporter.md` | `references/report-format.md` | 4 |

오케스트레이터: `.claude/skills/nk-nuclear-report/skill.md`

## Commit Convention
- 보고서 커밋: `report: daily NK nuclear update (YYYY-MM-DD)`
- 구조/설정 변경: `chore: 설명`
- `git add sources/ reports/` — sources 디렉토리도 함께 커밋한다

## Rules
- 출처 URL 없는 정보는 보고서에 포함하지 않는다
- 보고서 언어는 한국어가 기본이며, 원문 인용은 원어 보존한다
- 동일 뉴스가 여러 매체에서 보도된 경우 대표 1개만 포함하되 출처 목록에 모두 기재한다
- 보고서가 비어있더라도 파일은 생성한다 (특이사항 없음 보고서)
- 파이프라인 중간 산출물은 항상 생성한다
