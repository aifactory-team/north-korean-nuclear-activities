# North Korean Nuclear Activities Monitor

북한 핵활동에 대한 자동화된 일일 모니터링 및 보고서 생성 시스템.

GitHub Actions에서 Claude Code 에이전트가 매일 실행되어 다국어 웹검색을 수행하고, 중복을 제거한 일일 보고서를 자동 생성합니다.

## 구조

```
reports/
├── 2026/
│   ├── 04/
│   │   ├── 2026-04-01.md
│   │   ├── 2026-04-02.md
│   │   └── ...
│   ├── 05/
│   │   └── ...
```

## 동작 방식

1. **일일 자동 실행** — GitHub Actions가 매일 KST 09:00 (UTC 00:00)에 실행
2. **다국어 웹검색** — 한국어, 영어, 일본어, 중국어 키워드로 뉴스 검색
3. **중복 제거** — 이전 7일 보고서와 비교하여 이미 보고된 내용 제외
4. **보고서 생성** — 정형화된 Markdown 보고서 자동 작성
5. **자동 커밋** — 생성된 보고서를 git push

## 검색 키워드

| 언어 | 주요 키워드 |
|------|------------|
| 한국어 | 북핵, 북한 핵실험, 북한 미사일, 북한 핵무기, 북한 ICBM |
| English | North Korean nuclear, DPRK nuclear test, North Korea missile |
| 日本語 | 北朝鮮 核実験, 北朝鮮 ミサイル, 北朝鮮 核開発 |
| 中文 | 朝鲜核试验, 朝鲜核武器, 朝鲜导弹 |

## 보고서 형식

각 보고서에는 다음 섹션이 포함됩니다:

- **요약** — 3-5문장 핵심 요약
- **주요 뉴스** — 출처 URL 포함 상세 뉴스
- **분석 및 평가** — 종합 분석
- **국제 반응** — 각국 반응
- **동향 요약** — 핵실험/미사일/외교 상태 테이블
- **출처 목록** — 전체 참고 자료

## 설정

### 필수 시크릿

| Secret | 설명 |
|--------|------|
| `CLAUDE_CODE_OAUTH_TOKEN` | Claude Code OAuth 토큰. [Claude Code GitHub Actions 설정 가이드](https://code.claude.com/docs/en/github-actions) 참조 |

### 설정 방법

1. GitHub 리포지토리 Settings → Secrets and variables → Actions
2. `CLAUDE_CODE_OAUTH_TOKEN` 시크릿 추가
3. 값: Claude Code OAuth 토큰

### 수동 실행

Actions 탭 → "Daily NK Nuclear Activities Report" → "Run workflow" 클릭

특정 날짜 보고서 생성: `target_date`에 `YYYY-MM-DD` 형식 입력

## 기술 스택

- **CI/CD:** GitHub Actions
- **AI Agent:** [Claude Code](https://claude.com/claude-code) via [claude-code-action](https://github.com/anthropics/claude-code-action)
- **검색:** Claude Code 빌트인 WebSearch / WebFetch
- **모델:** Claude Opus 4.6

## 라이선스

MIT License - [LICENSE](LICENSE) 참조
