---
name: nk-nuclear-report
description: "북한 핵활동 일일 보고서를 자동 생성하는 오케스트레이터 스킬. 다국어 웹검색(한/영/일/중), 이전 7일 보고서 대비 중복제거, Markdown 보고서 작성, git 자동커밋을 수행한다. '북한 핵 보고서', 'NK nuclear report', '일일 보고서 생성', 'daily report' 요청 시 사용."
---

# NK Nuclear Report — 일일 보고서 생성 오케스트레이터

## 실행 흐름

### Phase 1: 준비
1. 대상 날짜 결정 (입력값 또는 오늘 날짜)
2. `reports/YYYY/MM/` 디렉토리 생성 (mkdir -p)
3. 이전 7일 보고서 파일 목록 조회 (Glob)
4. 이전 보고서 내용 읽기 (Read) → 이미 보고된 사건/출처 URL 목록 추출

### Phase 2A: WebSearch (빌트인)
최소 다음 키워드 그룹별 1회 이상 WebSearch 수행:

**필수 검색 (4개 언어):**
1. `북핵 최신 뉴스` / `북한 핵실험` / `북한 미사일 발사`
2. `North Korean nuclear news today` / `DPRK missile launch` / `North Korea sanctions`
3. `北朝鮮 核実験 最新` / `北朝鮮 ミサイル`
4. `朝鲜核试验 最新` / `朝鲜导弹`

### Phase 2B: Cheliped Browser (사이트별 검색)
`config/search-sites.json` 파일을 읽고 Cheliped CLI로 추가 검색 수행:

**검색 엔진 (search 커맨드):**
```bash
node $CHELIPED_CLI '[{"cmd":"search","args":["북핵","google"]},{"cmd":"extract","args":["all"]},{"cmd":"close"}]'
node $CHELIPED_CLI '[{"cmd":"search","args":["북한 핵실험","naver"]},{"cmd":"extract","args":["all"]},{"cmd":"close"}]'
node $CHELIPED_CLI '[{"cmd":"search","args":["朝鲜核试验","baidu"]},{"cmd":"extract","args":["all"]},{"cmd":"close"}]'
node $CHELIPED_CLI '[{"cmd":"search","args":["北朝鮮 核実験","yahoo_japan"]},{"cmd":"extract","args":["all"]},{"cmd":"close"}]'
```

**커스텀 사이트 (goto + extract):**
- `config/search-sites.json`의 `custom_sites`에서 `enabled: true`인 사이트 순회
- `search_url`의 `{query}`를 URL-인코딩된 키워드로 치환
- goto → wait(2000) → extract(all) → close 순서로 실행

**검색 전략:**
- WebSearch와 Cheliped 결과를 병합하여 더 넓은 범위의 소스 확보
- 각 언어별 최소 2개 키워드 검색
- 검색 결과에서 최근 24~48시간 내 뉴스에 집중
- 전문 사이트(38 North, NK News) 결과를 우선 활용

### Phase 3: 중복 제거
1. Phase 1에서 추출한 기존 보고 사건 목록과 비교
2. 제목 유사도 + 출처 URL 일치로 중복 판단
3. 동일 사건의 후속 보도는 "업데이트"로 표시하여 포함
4. 완전 중복은 제외

### Phase 4: 보고서 생성
`reports/YYYY/MM/YYYY-MM-DD.md` 형식으로 작성:

```markdown
---
title: "YYYY-MM-DD 북한 핵활동 일일 보고서"
date: YYYY-MM-DD
sources_count: N
new_items: N
updated_items: N
---

# YYYY-MM-DD 북한 핵활동 일일 보고서

## 요약
(3-5문장 핵심 요약. 특이사항 없으면 "금일 북한 핵활동 관련 특이사항 없음" 기재)

## 주요 뉴스
### 1. [뉴스 제목]
- **출처:** [매체명](URL)
- **일시:** YYYY-MM-DD
- **내용:** 상세 내용 2-3문장 요약
- **상태:** 신규 / 업데이트

(뉴스별 반복)

## 분석 및 평가
(수집된 뉴스 기반 종합 분석. 핵 프로그램 진전, 군사적 함의, 외교적 맥락 등)

## 국제 반응
(각국 정부/기관/전문가 반응. 출처 명시)

## 동향 요약
| 분류 | 상태 | 비고 |
|------|------|------|
| 핵실험 | 상태 | 설명 |
| 미사일 | 상태 | 설명 |
| 우라늄 농축 | 상태 | 설명 |
| 외교/제재 | 상태 | 설명 |

## 출처 목록
1. [제목](URL) - 매체명, 날짜
```

### Phase 5: 커밋
```bash
git add reports/
git commit -m "report: daily NK nuclear update (YYYY-MM-DD)"
```

## 에러 핸들링

| 에러 | 전략 |
|------|------|
| 검색 결과 전무 | "특이사항 없음" 보고서 생성 후 정상 커밋 |
| 일부 언어 검색 실패 | 성공한 언어 결과로 진행, 보고서에 누락 언어 표기 |
| 이전 보고서 없음 | 중복 제거 스킵, 전체 결과 사용 |
| 디렉토리 미존재 | mkdir -p로 자동 생성 |

## 테스트 시나리오

### 정상 흐름
1. workflow_dispatch로 수동 트리거
2. 보고서 생성 확인: `reports/2026/04/2026-04-01.md` 존재
3. frontmatter 형식 검증
4. 출처 URL 최소 1개 포함 확인
5. git commit 메시지 형식 확인

### 에러 흐름 — 검색 결과 없음
1. 모든 WebSearch가 결과 0건 반환
2. "특이사항 없음" 보고서가 생성되는지 확인
3. 보고서 구조(frontmatter, 섹션)는 동일하게 유지되는지 확인

### 중복 제거 흐름
1. 2일 연속 실행
2. 첫날 보고서의 주요 뉴스가 둘째날에 반복되지 않는지 확인
3. 후속 보도만 "업데이트" 태그로 포함되는지 확인
