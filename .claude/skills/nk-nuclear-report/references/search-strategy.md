# 검색 전략

## 검색 키워드

### Korean (한국어)
북핵, 북한 핵활동, 북한 핵실험, 북한 핵무기, 북한 미사일, 북한 ICBM, 북한 핵폐기물, 북한 핵개발, 북한 우라늄 농축, 북한 플루토늄, 조선 핵, 북한 제재

### English
North Korean nuclear, DPRK nuclear test, North Korea missile launch, DPRK ICBM, North Korea nuclear weapons, Pyongyang nuclear program, North Korea uranium enrichment, DPRK sanctions, North Korea plutonium, Korean peninsula nuclear

### Japanese (日本語)
北朝鮮 核実験, 北朝鮮 ミサイル, 北朝鮮 核開発, 北朝鮮 ウラン濃縮, 朝鮮半島 核問題

### Chinese (中文)
朝鲜核试验, 朝鲜核武器, 朝鲜导弹, 朝鲜半岛核问题, 朝鲜铀浓缩, 朝鲜制裁

## 필수 검색 횟수
- 한국어: 최소 3회
- 영어: 최소 3회
- 일본어: 최소 1회
- 중국어: 최소 1회
- 합계: 최소 8회 이상

## 검색 방법

### Method 1: WebSearch (빌트인)
Claude Code의 WebSearch 도구를 직접 호출한다.

### Method 2: Cheliped Browser (사이트별)
`config/search-sites.json`에서 `enabled: true`인 사이트만 검색한다.

**검색 엔진 (search 커맨드):**
```bash
node $CHELIPED_CLI '[{"cmd":"search","args":["검색어","엔진명"]},{"cmd":"extract","args":["all"]},{"cmd":"close"}]'
```

**커스텀 사이트 (goto + extract):**
```bash
node $CHELIPED_CLI '[{"cmd":"goto","args":["URL"]},{"cmd":"wait","args":["2000"]},{"cmd":"extract","args":["all"]},{"cmd":"close"}]'
```

- 각 검색 후 반드시 `close` 커맨드로 세션을 종료한다

## 출력 스키마: search-results.json

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

이 파일은 Phase 1에서 쓰고 이후 Phase에서 재읽기 않는다 (write-only, 디버깅/추적 전용).
