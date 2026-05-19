---
name: inactive-customer-report
description: |
  나인하이어 고객성공팀 전용 비활성 고객사 정기 리스트업 자동화 스킬.
  Enterprise 플랜 고객사 중 Amplitude 조건 기반 비활성 고객사를 추출하고,
  Notion 정기 점검 DB에 N월 M주차 페이지 + 인라인 DB를 자동 생성한다.
  이전 달 컨택 여부를 자동으로 체크하고, 완료/실패/이상 상황을 Slack으로 알림.

  트리거: "비활성 고객사 찾아줘", "이탈 위험 고객 뽑아줘", "정기 점검 리스트 만들어줘",
  "엠플리튜드 조건으로 고객사 추출해줘", "비활성 고객사 노션에 정리해줘",
  "enterprise 비활성 고객사 추출", "enterprise 고객사 점검해줘",
  "최근 90일 기준 enterprise 고객사 중 신규 지원자 5명 미만, 합/불 처리 1회 미만 고객사 뽑아줘",
  신규지원자/합불처리 등의 조건을 언급하며 고객사 추출·저장을 요청하는 경우 반드시 이 스킬을 사용할 것.
---

# 비활성 고객사 리스트업 스킬 (v3.2 — 지난 달 컨택 의미 정정)

## v3.2 변경 사항 (2026-05-19)

- **`지난 달 컨택` 컬럼의 의미를 컬럼명 그대로 정의**:
  - `해당` = 지난 달에 컨택을 함 (이력 있음) ← 이전 달 리스트 존재 + 상태 IN {메일 발송 완료, 회신 완료}
  - `미해당` = 지난 달에 컨택 안 함 (이력 없음) ← 그 외 모든 경우 (이전 달 리스트에 없음 / 있어도 상태=추출)
- v3.1에서 라벨링이 반대로 적용된 버그를 정정. v3.1의 `해당`/`미해당`은 거꾸로 사용되었음.

## v3.1 변경 사항 (DEPRECATED — v3.2에서 의미 반대로 정정)

- 지난 달 컨택 판단 룰을 단순 존재 여부에서 `이전 달 리스트 존재 + 상태 IN {메일 발송 완료, 회신 완료}` 조합으로 강화.
- 단, `해당`/`미해당` 라벨 의미를 거꾸로 매핑했음 → v3.2에서 정정.

## v3 변경 사항 (2026-05-19)

- Amplitude **차트 API의 500-시리즈 하드 리밋**을 우회하기 위해 **CSV 익스포트 엔드포인트**를 사용한다 (`POST /d/data/{app_id}/csv`).
- 차트 API 방식(`/d/data/{app_id}/dataset`)으로는 상위 500개 그룹만 반환되어 신규지원자 수가 적은(=비활성에 가까운) 회사가 누락됐다. CSV는 685행 가량 반환되며 enterprise 회사 전수 조회가 가능하다.
- 단, **90일간 이벤트가 0건인 enterprise 고객사는 여전히 Amplitude 결과에 포함되지 않는다**. 이는 Amplitude의 이벤트 기반 분석 특성상 불가피한 한계이므로, 요약 섹션에 별도 명시한다.

## 환경 설정

- **Amplitude Project ID**: `742296` (Ninehire Prod, 워크스페이스 slug `nh-jk`, 조직 `JobKorea_Ninehire`)
- **Amplitude 차트 베이스 URL**: `https://app.amplitude.com/analytics/nh-jk/chart/`
- **Notion 상위 DB URL**: `https://www.notion.so/3347d8322b048067925ceffb381e6d9d`
- **Notion 템플릿 페이지 ID**: `3357d8322b0480c9a683c2c1b807d56d`
- **Slack Webhook**: `~/.claude/.tokens.env`의 `SLACK_WEBHOOK_CSCX` 환경 변수에서 로드. 하드코딩 금지.
- **Slack 채널**: `#cscx_claude`

### 사전 조건

- **Amplitude 브라우저 세션 로그인 필요**. Chrome MCP를 통해 Ninehire 워크스페이스(`nh-jk`)에 접근 권한이 있는 계정으로 로그인되어 있어야 한다.
- 다른 계정(예: worxphere.ai 도메인)으로 로그인된 경우 `no-access/ninehire` 페이지가 뜨며, 작업 중단 후 Slack에 실패 알림을 보내고 사용자에게 재로그인을 요청한다.
- 세션 확인 방법: `https://app.amplitude.com/analytics/nh-jk/home` 접근 후 페이지 정상 로드 여부 + 로그인 이메일 확인.

---

## 기본 조건 (사용자가 별도 지정하지 않을 경우)

- 조회 기간: **최근 90일 (총합 기준, interval=0)**
- 대상 플랜: **enterprise** (`grp:Plan`)
- 아래 **AND 조건** 두 가지 모두 충족 시 비활성 고객사로 분류:
  - 신규 지원자 90일 총합 **5명 미만** (`Add Applicant` 총합 < 5)
  - 합불처리 90일 총합 **0건** (`Pass Applicant` + `Fail Applicant` 총합 < 1)
- 신규 지원자 수 < 5이더라도 합불처리 수가 1 이상이면 대상에서 제외

Amplitude 기준 필드:
- 회사명: `grp:Name`
- 플랜: `grp:Plan`
- 워크스페이스 ID: 현재 Company 그룹 taxonomy에 노출된 ID 프로퍼티 없음 → 공란 처리

사용자가 새 조건을 제시하면 해당 조건으로 대체한다.

---

## 실행 순서

### 1단계: Amplitude CSV 익스포트로 enterprise 전수 데이터 수집

**핵심: CSV 익스포트 엔드포인트 사용**

차트 API(`/d/data/{app_id}/dataset`)는 결과를 상위 500개 시리즈로 제한하지만, CSV 익스포트는 그 제한을 우회한다. Add/Pass/Fail 각 이벤트에 대해 동일한 정의로 CSV를 받는다.

**엔드포인트**: `POST https://app.amplitude.com/d/data/742296/csv`

**Body** (`application/x-www-form-urlencoded`):
- `definition` (JSON 문자열): 차트 정의
- `downloadId`: 임의 8자 문자열
- `name`: CSV 파일명 (식별용)
- `chart_id`: 임의 문자열 (예: `"ic"`)
- `headers` (JSON 배열): 헤더 행, 예 `[["회사명","Plan","Total"]]`

**Definition 템플릿** (eventType만 교체):

```javascript
const definition = {
  vis: "line", app: "742296", colorAssignments: {}, recycledColors: [], shouldShowTwoYearRange: false,
  params: {
    segments: [{name: "All Users", conditions: []}],
    interval: 0, nthTimeLookbackWindow: 365, additionalPeriods: [],
    metric: "totals", countGroup: "Company", excludeDays: [],
    periodOverPeriod: false,
    groupBy: [
      {type: "group", value: "grp:Name", group_type: "Company"},
      {type: "group", value: "grp:Plan", group_type: "Company"}
    ],
    events: [{event_type: EVENT_TYPE, filters: [], group_by: []}],  // "Add Applicant" / "Pass Applicant" / "Fail Applicant"
    range: "Last 90 Days", timezone: "Asia/Seoul"
  },
  type: "eventsSegmentation", version: 41
};
```

**실행 코드 (Chrome MCP javascript_tool로 실행)**:

```javascript
async function csvExport(eventType) {
  const definition = {
    vis: "line", app: "742296", colorAssignments: {}, recycledColors: [], shouldShowTwoYearRange: false,
    params: {
      segments: [{name: "All Users", conditions: []}],
      interval: 0, nthTimeLookbackWindow: 365, additionalPeriods: [],
      metric: "totals", countGroup: "Company", excludeDays: [],
      periodOverPeriod: false,
      groupBy: [
        {type: "group", value: "grp:Name", group_type: "Company"},
        {type: "group", value: "grp:Plan", group_type: "Company"}
      ],
      events: [{event_type: eventType, filters: [], group_by: []}],
      range: "Last 90 Days", timezone: "Asia/Seoul"
    },
    type: "eventsSegmentation", version: 41
  };
  const params = new URLSearchParams();
  params.set("definition", JSON.stringify(definition));
  params.set("downloadId", "ic_" + Math.random().toString(36).substring(2, 10));
  params.set("name", `inactive_${eventType}`);
  params.set("chart_id", "ic");
  params.set("headers", JSON.stringify([["회사명", "Plan", "Total"]]));
  const r = await fetch("/d/data/742296/csv", {
    method: "POST",
    credentials: "include",
    body: params
  });
  if (!r.ok) throw new Error(`CSV export failed for ${eventType}: ${r.status}`);
  return await r.text();
}

// 세 이벤트 병렬 호출
const [addCsv, passCsv, failCsv] = await Promise.all([
  csvExport("Add Applicant"),
  csvExport("Pass Applicant"),
  csvExport("Fail Applicant")
]);
```

**CSV 파싱**:

CSV 구조 (Amplitude 응답):
1. 행 1: 헤더 (`회사명, Plan, Total`)
2. 행 2: `Totals`
3. 행 3: 이벤트 이름 (e.g. `Add Applicant`)
4. 행 4: 그룹 라벨 (`Name; Plan, (none)`)
5. 행 5+: **데이터 행** — `"회사명; plan", "값"` 형식

```javascript
function parseEntMap(csv) {
  const lines = csv.split("\r\n").filter(l => l.length > 0);
  const map = {};
  function parseLine(line) {
    const out = []; let cur = "", inQ = false;
    for (const ch of line) {
      if (ch === '"') inQ = !inQ;
      else if (ch === ',' && !inQ) { out.push(cur); cur = ""; }
      else cur += ch;
    }
    out.push(cur);
    return out;
  }
  for (const line of lines.slice(4)) {  // 5번째 행부터 데이터
    const cols = parseLine(line);
    if (cols.length < 2) continue;
    const label = cols[0].replace(/^"|"$/g, "");
    const v = parseInt(cols[1].replace(/"/g, ""));
    if (isNaN(v)) continue;
    const [name, plan] = label.split("; ");
    if (plan === "enterprise") map[name] = (map[name] || 0) + v;
  }
  return map;
}

const addMap = parseEntMap(addCsv);
const passMap = parseEntMap(passCsv);
const failMap = parseEntMap(failCsv);
```

**집합 추출**:

```javascript
const blocked = /test|TEST|테스트|demo|DEMO|나인하이어/i;

// 집합 A: Add Applicant < 5 (테스트 계정 제외)
const setA = Object.entries(addMap)
  .filter(([name, v]) => v < 5 && !blocked.test(name))
  .map(([name, v]) => ({name, add: v}));

// 최종 = A ∩ B (합불처리 = 0)
const enriched = setA.map(item => {
  const pass = passMap[item.name] || 0;
  const fail = failMap[item.name] || 0;
  return {...item, pass, fail, judgeTotal: pass + fail, meetsAND: (pass + fail) < 1};
});

const final = enriched.filter(e => e.meetsAND);
```

**집합 통계** (요약에 사용):
- `count_setA` = `setA.length` (신규지원자 < 5)
- `count_final` = `final.length` (AND 조건 충족)

### 2단계: Amplitude 차트 저장 (요약 링크용)

차트 API(`/d/data/.../dataset`)를 한 번 호출하여 차트를 그린 후 **저장(💾) 버튼**으로 영구 URL 확보.

```javascript
// 새 차트 페이지로 이동 후 차트 저장
// URL 패턴: https://app.amplitude.com/analytics/nh-jk/chart/{chart_id}
// 차트 이름: "[정기점검] 비활성 enterprise 고객사 추출 - YYYY년 M월 N주차"
```

저장된 차트 URL을 `amplitude_chart_url`로 보관.

> 차트 자체는 1단계 CSV 결과와 별개이며, Notion 요약과 Slack 메시지의 참고 링크용으로만 사용된다.
> 차트 표시에서는 500-시리즈 한도로 잘려보일 수 있다는 점을 명심한다.

### 3단계: 이전 달 Notion DB 조회 (지난 달 컨택 확인)

이번 달 고객사 중 지난 달에 실제로 컨택이 진행된 회사를 가려내기 위해 조회한다.

1. 메인 DB(`collection://3347d832-2b04-80bf-bd28-000b55e4f205`)를 쿼리하여 이전 달 주차 페이지 검색
   - 예: 현재가 2026년 5월이면 이름에 "2026년 4월" 포함 페이지 검색
2. 해당 페이지가 존재하면, 그 페이지 하위의 인라인 DB를 `notion-query-data-sources`로 조회하여 **회사명 + 상태** 조회
   - 예: `SELECT "회사명", "상태" FROM "collection://..."`
3. 이번 달 비활성 목록(집합 A 전체)과 교차하여 `{회사명: "해당"/"미해당"}` 맵 생성:
   - **`해당` 조건 (지난 달 컨택 이력 있음)**: 이전 달 리스트에 회사가 있고 + 그 행의 상태가 `메일 발송 완료` 또는 `회신 완료`
   - **`미해당` 조건 (지난 달 컨택 이력 없음)** — 이하 모든 경우:
     - 이전 달 리스트에 없는 신규 회사
     - 이전 달 리스트에 있지만 상태가 `추출`인 회사 (추출만 됐고 컨택 미진행)
     - 이전 달 페이지 자체가 존재하지 않는 경우
4. `count_prev_contact` = 맵에서 `"해당"`인 회사 수 (= 지난 달 컨택 진행 완료 건수, Notion 요약에 사용)
5. `count_new_contact_target` = count_setA - count_prev_contact (= 이번 달 신규 컨택 대상 건수, 즉 지난 달 컨택 미진행)

> ⚠️ 지난 달 컨택 값은 반드시 실제 이전 달 리스트의 `회사명 + 상태` 조회 결과를 기반으로 입력한다. 이전 달 페이지가 없거나 행이 없는 경우 자동으로 `미해당` 처리 (공란 금지).
>
> 의미: `해당` = 지난 달에 컨택 함 (이력 있음) / `미해당` = 지난 달에 컨택 안 함 (이력 없음).

### 4단계: Notion 상위 페이지 생성

**타이틀 형식**: `YYYY년 M월 N주차`

주차 계산:
- 해당 월 1일을 포함하는 주를 1주차로 함
- 예: 2026-04-01(수) → 4월 1주차, 2026-03-31(화) → 3월 5주차

**생성 전 중복 확인**: 동일 타이틀 페이지가 이미 존재하면 덮어쓰지 않고 사용자에게 알린다. (스케줄 자동 실행 시에는 동일 페이지 존재 시 업데이트 모드로 전환 — 기존 페이지 ID 재사용)

`notion-create-pages`로 메인 DB에 페이지를 생성한다.

**페이지 프로퍼티:**
- `이름`: `YYYY년 M월 N주차`
- `기간`: 조회 시작일 ~ 종료일 (date range 형식)
  - API 입력 시: `date:기간:start`, `date:기간:end`, `date:기간:is_datetime`
  - 예: start: `2026-02-18`, end: `2026-05-18`
- `상태`: `추출`

### 5단계: 템플릿 적용 및 인라인 DB 컬럼 확인

**템플릿 적용:**

생성된 페이지에 `notion-update-page` (command=`apply_template`)를 사용하여 템플릿 페이지(`3357d832-2b04-80c9-a683-c2c1b807d56d`)의 내용을 적용한다.

> 중요: 템플릿 원본 페이지에 직접 값을 입력하면 안 된다. 반드시 새로 생성된 주차 페이지에 적용한다.

**인라인 DB 컬럼 확인:**

템플릿이 적용된 페이지를 fetch하여 본문에 생성된 `<database>` 인라인 DB의 collection URL(data-source-url)을 확보한다. 그 URL을 `notion-fetch`로 다시 조회하여 **실제 컬럼명 및 타입**을 확인한다.

확인해야 할 주요 컬럼 (현재 템플릿 기준):

| 컬럼명 | 타입 | 입력값 |
|---|---|---|
| 회사명 | title | Amplitude 추출 회사명 |
| 신규 지원자 수 | number | Add Applicant 90일 총합 (숫자) |
| 합불처리 수 | number | Pass + Fail Applicant 90일 총합 (숫자) |
| 지난 달 컨택 | select | `해당` 또는 `미해당` |
| 조건 | select | `충족` (AND 만족) 또는 `미충족` |
| 상태 | status | `추출` |
| 웍스ID | rich_text | 공란 (taxonomy 미노출) |
| 고객사 담당자 | rich_text | 공란 |
| 이메일 | email | 공란 |
| 비고 | rich_text | 공란 |
| 날짜 | created_time | 자동 |

**상태 SELECT 허용 옵션**: `추출`, `메일 발송 완료`, `회신 완료`
**지난 달 컨택 SELECT 허용 옵션**: `해당`, `미해당`
**조건 SELECT 허용 옵션**: `충족`, `미충족`

### 6단계: Notion 페이지 요약 작성

행 삽입 전, `notion-update-page` (command=`update_content`)로 주차 페이지 본문 상단에 추출 요약을 작성한다. 인라인 DB `<database>` 블록 **앞에** 삽입한다.

**요약 내용 (Notion Markdown):**

```markdown
## 📊 추출 요약

**[집합 A] 신규지원자 < 5 (enterprise)**

| 항목 | 수치 |
| --- | --- |
| 전체 추출 고객사 (신규지원자 < 5) | {count_setA}개사 |
| AND 조건 해당 (합불처리 = 0) | {count_final}개사 |
| 지난 달 컨택 이력 있음 (지난 달 컨택=해당) | {count_prev_contact}개사 |
| 지난 달 컨택 이력 없음 (지난 달 컨택=미해당) | {count_new_contact_target}개사 |

- 조회 기간: 최근 90일 ({start_date} ~ {end_date}, 총합 기준)
- 집합 A 조건: 신규지원자 90일 총합 < 5 (enterprise)
- AND 조건: 집합 A 중 합불처리(합격+불합격) 90일 총합 = 0 → 조건에 `충족` 표시
- 지난 달 컨택 룰: `이전 달 리스트 존재 + 상태 IN {메일 발송 완료, 회신 완료}` → `해당`, 그 외 → `미해당`
- Amplitude 차트: [차트 보기]({amplitude_chart_url})

> ℹ️ **데이터 한계 안내**: Amplitude는 이벤트 기반 분석 도구로, 90일간 단 하나의 이벤트(신규지원자/합불처리)도 발생시키지 않은 enterprise 고객사는 본 결과에 포함되지 않는다. 이러한 '0이벤트' 고객사를 식별하려면 Notion 워크스페이스 마스터 DB와 별도 대조가 필요하다.
```

### 7단계: 고객사 행 삽입

**집합 A 전체**(테스트 제외)를 `notion-create-pages`로 인라인 DB에 삽입한다. AND 조건 충족 여부는 `조건` 컬럼에 표시.

각 행의 값:
- `회사명`: 회사명 (title)
- `신규 지원자 수`: Add Applicant 90일 총합 (숫자)
- `합불처리 수`: Pass + Fail Applicant 90일 총합 (숫자)
- `지난 달 컨택`: `해당` 또는 `미해당` (3단계 결과)
- `조건`: AND 충족 시 `충족`, 아니면 `미충족`
- `상태`: `추출`
- `웍스ID`, `고객사 담당자`, `이메일`, `비고`: 공란

**정렬**: 회사명 알파벳/한글 순 또는 신규지원자 수 오름차순 권장.

> 고객사 수가 50개를 초과하면 50개 단위로 나눠서 삽입한다 (Notion API 안정성, `notion-create-pages`는 1회 호출 100개 제한이지만 안전 마진으로 50개).

### 8단계: Slack 알림 발송

완료 후 Bash `curl` 또는 Chrome MCP로 Webhook 호출. 토큰은 `~/.claude/.tokens.env`의 `SLACK_WEBHOOK_CSCX` 사용 권장 (없으면 폴백).

**성공 시:**

```bash
SLACK_URL=$(sed -n 's/^SLACK_WEBHOOK_CSCX=//p' ~/.claude/.tokens.env)
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"text":"✅ *비활성 고객사 리스트업 완료*\n• 조회 기간: RANGE_TEXT\n• 추출 고객사: N개사 (신규지원자<5 enterprise) / AND 조건 충족: M개사\n• 이번 달 신규 컨택 대상: K개사 / 지난 달 컨택 진행 완료: L개사\n• 노션 페이지: <NOTION_PAGE_URL|TITLE>\n• Amplitude 차트: <AMPLITUDE_CHART_URL|차트 보기>\n• 조건: 신규지원자 90일 총합 < 5 AND 합불처리(합격+불합격) 90일 총합 < 1 (enterprise)\nℹ️ 0이벤트 enterprise는 Amplitude 특성상 누락 — 필요 시 Notion 워크스페이스 DB와 대조 필요"}' \
  "$SLACK_URL"
```

**실패 / 이상 상황:**

| 상황 | Slack 메시지 |
|---|---|
| Amplitude 세션 만료/권한 없음 | `❌ *리스트업 실패* — Amplitude 세션 만료 또는 권한 없음 (no-access). 재로그인 후 재실행 필요.` |
| CSV 익스포트 실패 (status != 200) | `❌ *리스트업 실패* — CSV export 오류: [status, eventType]` |
| Notion 페이지 생성 오류 | `❌ *리스트업 실패* — Notion 생성 오류: [에러 내용]` |
| 추출 고객사 0개 | `⚠️ *이상 감지* — 비활성 고객사가 0개로 추출됨. 조건 또는 데이터를 확인하세요.` |
| 추출 고객사가 이전 달 대비 2배 초과 | `⚠️ *이상 감지* — 이번 달 추출 N개사 / 이전 달 M개사. 조건을 확인하세요.` |

이상 상황(0개, 2배 초과)은 경고만 발송하고 작업은 계속 진행한다.
API 오류로 작업 자체가 불가능한 경우에만 실패 메시지 발송 후 중단한다.

---

## 완료 체크리스트

- [ ] Amplitude 브라우저 세션 확인 (Ninehire 워크스페이스 권한 있는 계정으로 로그인됨)
- [ ] CSV 익스포트 3회 호출 (Add / Pass / Fail Applicant) — 각 행 500개 이상 반환되는지 확인
- [ ] enterprise 필터링 + 테스트 계정 제외
- [ ] 집합 A (Add < 5) 및 AND 조건(합불처리 = 0) 분리 계산
- [ ] Amplitude 차트 저장 → `amplitude_chart_url`
- [ ] 이전 달 Notion 페이지 확인 → 지난 달 컨택 맵 생성 (해당/미해당)
- [ ] Notion 상위 페이지 생성 (N월 M주차, 기간 date range, 상태=추출)
- [ ] 템플릿 적용 (`3357d832-2b04-80c9-a683-c2c1b807d56d`)
- [ ] 인라인 DB fetch → 실제 컬럼명/타입 확인
- [ ] Notion 페이지 요약 작성 (집합 A/AND/지난 달 컨택 + Amplitude 차트 링크 + 0이벤트 한계 안내)
- [ ] 집합 A 전체 행 삽입 (조건: 충족/미충족 표시, 50개 단위 분할)
- [ ] Slack #cscx_claude 알림 발송 (성공/실패/이상 상황 분기)

---

## 주의사항

- **CSV 익스포트는 Amplitude UI에 명시된 기능이 아닌 내부 엔드포인트**다. 향후 Amplitude 측 변경 가능성이 있으므로, 응답이 비정상(파싱 실패, 0행)인 경우 차트 API 폴백 + Slack 경고 발송으로 graceful fail.
- 테스트/내부 계정 필터링: 이름에 `test`, `TEST`, `테스트`, `demo`, `DEMO`, `나인하이어` 포함 시 제외.
- 페이지 생성 전 동일 타이틀(주차) 페이지 존재 여부를 확인한다. 자동 스케줄 실행이면 기존 페이지를 업데이트, 수동 실행이면 사용자에게 알리고 중단 여부 확인.
- 템플릿 원본 페이지(`3357d8322b0480c9a683c2c1b807d56d`)에 직접 데이터를 입력하지 않는다.
- 인라인 DB 컬럼명은 반드시 fetch 결과 기준으로 사용한다. UI 표기 이름을 그대로 가정하지 않는다.
- 지난 달 컨택 값은 반드시 실제 이전 달 리스트 조회 결과를 기반으로 입력한다. 이전 달 목록에 있으면 `해당`, 없거나 페이지 자체가 없으면 `미해당` (공란 금지).
- **0이벤트 enterprise 고객사 한계**: Amplitude 응답에 포함되지 않으므로, Notion 요약과 Slack 메시지에 항상 명시한다. 정확한 전수 조사가 필요한 경우 Notion 워크스페이스 마스터 DB와 대조하는 별도 작업이 필요하다.

---

## 부록: 폐기된 접근 방식 (히스토리)

### v2 — 차트 API 방식 (deprecated 2026-05-19)

`POST /d/data/{app_id}/dataset` 으로 차트 쿼리 결과를 받는 방식. **상위 500개 시리즈만 반환**되어 비활성에 가까운 (=이벤트 적은) 회사가 누락되는 결정적 한계가 있었다. v2의 chart-API 코드 블록은 더 이상 사용하지 않는다.

검증 결과 비교 (2026년 5월 4주차 기준):

| 방식 | enterprise 회사 수 | Add Applicant < 5 | AND 조건 충족 |
|---|---|---|---|
| v2 차트 API | 312 | 4 | 1 |
| **v3 CSV 익스포트** | **364** | **65** | **21** |
| 2026년 4월 1주차 (기준) | — | 62 | 17 |

### 시도했으나 작동하지 않은 방식

- `chunkGroupByLimit` / `groupByLimit` 파라미터 변경 → 항상 500으로 cap
- `orderBy: asc` / sort 방향 변경 → "Invalid definition" 오류
- `dataTable` type 지정 → "unknown definition type" 오류
- 회사명 prefix `contains` 필터 chunking → 가능하지만 enumeration 누락 위험 및 쿼리 횟수 폭증
- Behavioral Cohort API → 엔드포인트 발견 못함
- `/d/data/{app}/groups` 등 group enumeration → 모두 404
- `gp:Company` / `grp:id` 등 워크스페이스 ID 프로퍼티 → taxonomy에 미노출
