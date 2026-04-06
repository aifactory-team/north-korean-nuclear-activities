# 태깅 규칙

## 태그 정의

| 태그 | 의미 | 조건 |
|------|------|------|
| `new` | 신규 | 이전 소스에 동일 URL·유사 제목 없음 |
| `reported` | 보고됨 | 이전에 이미 보고된 동일 내용 |
| `update` | 후속보도 | 기존 사건의 후속이지만 유의미한 새 정보 포함 |

## 중복 판단 기준

1. **URL 일치** → `reported` (가장 확실한 기준)
2. **제목 유사도** — 핵심 키워드(인명, 지명, 사건명) 80%+ 일치 → `reported`
3. **후속 보도 판단** — 동일 사건 + 새로운 수치/성명/결과 → `update`

## 비교 범위

- 이전 **7일**의 `sources/*/index.json`만 읽는다 (개별 items 파일은 읽지 않음)
- 7일 이전 소스는 무시한다

## 출력 스키마

### index.json (경량 인덱스)
소스당 필수 필드만 포함. 7일치 읽어도 ~14KB.
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

### items/src-XXX.json (개별 소스 상세)
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

## 원칙
- `reported` 태그 소스도 index + items에 기록한다 (추적 가능성 보존)
- `tag_reason`은 개별 items 파일에 구체적으로 작성한다
- `related_report`/`related_item`은 `reported`/`update` 태그에만 기록한다
