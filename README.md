# inactive-customer-report

나인하이어 고객성공팀 전용 비활성 고객사 정기 리스트업 자동화 스킬.

Enterprise 플랜 고객사 중 Amplitude 이벤트 기반 비활성 조건(신규 지원자 < 5 AND 합불처리 = 0, 최근 90일)을 충족하는 고객사를 추출해 Notion 정기 점검 DB에 N월 M주차 페이지 + 인라인 DB로 자동 등록한다. 지난 달 컨택 진행 여부도 자동 판별하고, 완료/실패/이상 상황을 Slack `#cscx_claude`에 알림으로 발송한다.

핵심: Amplitude 차트 API의 **500-시리즈 하드 리밋**을 우회하는 **CSV 익스포트 엔드포인트**(`POST /d/data/{app_id}/csv`)를 사용해 enterprise 전수 조회를 수행한다.

## 사전 준비

### MCP 서버 연동

| MCP 서버 | 용도 | 필수 여부 |
|----------|------|-----------|
| Notion | 메인 DB / 주차 페이지 / 인라인 DB CRUD | 필수 |
| Claude in Chrome | Amplitude 브라우저 세션을 통한 CSV 익스포트 호출 | 필수 |

### Amplitude 사전 조건

- Chrome MCP에 연결된 브라우저로 Amplitude에 로그인되어 있어야 한다.
- 로그인 계정은 **Ninehire (Prod)** 워크스페이스(`nh-jk`, 조직 `JobKorea_Ninehire`, App ID `742296`)에 접근 권한이 있어야 한다.
- 다른 도메인 계정으로 로그인된 경우 `no-access/ninehire` 페이지로 리다이렉트되며 스킬이 실패 처리하고 Slack에 알림을 보낸다.

### API 토큰 설정

`~/.claude/.tokens.env`에 아래 토큰 추가:

| 토큰 키 | 발급 방법 |
|---------|-----------|
| `NOTION_API_TOKEN` | Notion 내부 통합 API 토큰 |
| `SLACK_WEBHOOK_CSCX` | Slack Incoming Webhook URL (`#cscx_claude` 채널) |

> 이 스킬은 별도의 Amplitude API 키를 사용하지 않고, Chrome 세션의 쿠키 인증으로 내부 엔드포인트를 호출한다.

### 권한 설정 (선택)

`.claude/settings.local.json` 권한 예시:

```json
{
  "permissions": {
    "allow": [
      "Bash(curl -* -H * https://hooks.slack.com/*)",
      "Bash(curl -* -H * https://api.github.com/*)"
    ]
  }
}
```

## Notion 환경

스킬에서 하드코딩된 Notion 리소스 ID:

| 항목 | ID |
|---|---|
| 메인 DB (`🌒 비활성 고객사 리스트`) | `3347d8322b048067925ceffb381e6d9d` |
| Data Source | `collection://3347d832-2b04-80bf-bd28-000b55e4f205` |
| 페이지 템플릿 (`새 페이지`) | `3357d8322b0480c9a683c2c1b807d56d` |

다른 워크스페이스에서 사용하려면 `SKILL.md` 상단의 환경 설정 섹션을 수정해야 한다.

## 실행 방법

트리거 워딩 예시:

- `비활성 고객사 찾아줘`
- `이탈 위험 고객 뽑아줘`
- `정기 점검 리스트 만들어줘`
- `엠플리튜드 조건으로 고객사 추출해줘`
- `enterprise 비활성 고객사 추출`
- `최근 90일 기준 enterprise 고객사 중 신규 지원자 5명 미만, 합/불 처리 1회 미만 고객사 뽑아줘`

스케줄 자동 실행: `~/.claude/scheduled-tasks/inactive-customer-extraction-monthly/SKILL.md`에서 cron으로 매월 첫째 월요일 오전 11시 트리거.

## 출력물

1. **Notion 주차 페이지** (`YYYY년 M월 N주차`, 상태=`추출`, 기간=조회 90일 범위)
2. 페이지 본문 상단 **추출 요약 표** (전체 추출 / AND 충족 / 신규 컨택 대상 / 지난 달 컨택 진행 완료)
3. 페이지 본문 **인라인 DB** (`엠플리튜드 추출 고객사 목록`) — 회사명·신규 지원자 수·합불처리 수·지난 달 컨택·조건·상태
4. **Amplitude 영구 차트 URL** (요약 + Slack 메시지에 링크)
5. **Slack 알림** (`#cscx_claude`)

## 알려진 한계

- **0이벤트 enterprise 고객사 누락**: Amplitude는 이벤트 기반 분석 도구로, 90일간 단 하나의 이벤트(신규지원자/합불처리)도 발생시키지 않은 회사는 결과에 포함되지 않는다. 이러한 회사를 식별하려면 Notion 워크스페이스 마스터 DB와 별도 대조가 필요하다.
- **차트 UI는 500개로 잘려보임**: 영구 차트 링크는 참고용이며, 실제 65개사 같은 전수 데이터는 CSV 익스포트 결과에서 확인 가능하다.

## 버전 히스토리

- **v3.1 (2026-05-19)**: 지난 달 컨택 판단 룰 강화 — `리스트 존재` 단순 매칭에서 `리스트 존재 + 상태 IN {메일 발송 완료, 회신 완료}`로 변경.
- **v3 (2026-05-19)**: Amplitude CSV 익스포트 엔드포인트 도입, 차트 API의 500-시리즈 한도 우회. 차트 API 방식(v2)은 deprecated.
- **v2**: 차트 API(`/d/data/{app}/dataset`) 기반 (500-시리즈 한도로 누락 발생).

상세 변경 이력은 `skills/inactive-customer-report/SKILL.md`의 부록 참고.
