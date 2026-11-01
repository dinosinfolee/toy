# 두루보드 자료 제공 규격 (v1, 초안)

두루보드와 연동하려는 서버가 지켜야 할 약속입니다.

## 0. AI 코딩 도우미에게 (작업 지시)

이 문서를 받은 AI 코딩 도우미는 함께 받은 시스템에 '두루보드 자료 제공 기능'을 아래 순서로 추가하십시오. 기존 기능과 화면은 변경하지 마십시오.

**먼저 확인할 것**

- 소스 코드를 보고 사용 기술(언어·프레임워크·자료 저장 방식)을 파악하여 그 방식에 맞게 구현하십시오.
- 교사가 '전송할 자료'를 알려 주었으면 그대로 따르고, 알려 주지 않았으면 소스 코드에서 알맞은 자료를 찾아 제안한 뒤 구현 전에 확인을 받으십시오.
- 행을 구분하는 값(예: 학번)을 찾아 식별 열(`role: "id"`)로 지정하고, 한 행이 그 값 하나(예: 학생 한 명)에 대응하도록 구성하십시오.
- 이름·연락처 등 개인정보는 교사가 요청한 경우에만 포함하십시오.

**구현할 내용**

1. 요청 받기: 일반 서버는 1장(`GET /api/v1/data`, `Authorization: Bearer` 인증, CORS), Google Apps Script 웹 앱은 3장(Apps Script 연결 방식)을 따릅니다.
2. 인증코드: 관리자가 발급·폐기할 수 있게 하고, 원문은 저장하지 않고 해시로만 보관합니다. 인증코드마다 응답할 표(범위)를 정할 수 있게 합니다.
3. 응답: 2장 형식(`meta`·`columns`·`rows`)을 따릅니다. 요청 한 번에 표 하나, 값은 숫자·문자·true/false·null만 사용합니다. 제출 여부는 1/0, 점수는 숫자, 미제출·미응시는 null로 표현합니다.
4. 제한과 오류: 응답 20MB·50,000행 이하, 요청이 너무 많으면 429. 오류 응답에 개인정보나 내부 정보를 넣지 않습니다.
5. 연결 안내: 인증코드를 발급하면 4장의 '두루보드 연결 정보'를 한 번 표시합니다.

**완료 후 교사에게 알려 줄 것**

- 변경·추가한 파일 목록
- 인증코드 발급 방법과, 발급 시 표시되는 '두루보드 연결 정보' 예시
- 시험 방법(일반 서버는 curl 명령)과 실제 응답 예시

## 1. 요청

```
GET https://{도메인}/api/v1/data
Authorization: Bearer {인증코드}
Accept: application/json
```

- HTTPS만 허용합니다.
- 조회 전용(GET)입니다.
- 인증코드는 URL에 넣지 않고 `Authorization` 헤더로만 받습니다.
- 인증코드가 없거나 틀리면 401, 권한이 없으면 403, 경로가 없으면 404, 요청이 너무 많으면 429로 응답합니다.
- CORS: 두루보드는 브라우저에서 직접 요청하므로 다음 머리글로 응답합니다. 쿠키를 쓰지 않고 Bearer 인증만 사용하므로 모든 출처를 허용해도 됩니다.
  - `Access-Control-Allow-Origin: *`
  - `Access-Control-Allow-Headers: Authorization, Accept`
  - `Access-Control-Allow-Methods: GET, OPTIONS` (OPTIONS 사전 요청에는 204로 응답)
- 응답 머리글: `Content-Type: application/json; charset=utf-8`
- **요청 한 번에 표 하나**를 응답합니다. 여러 표를 보여 주려면 서버가 다음 중 하나를 택합니다.
  - 행 단위가 같으면(예: 학생 한 명당 한 행) 열을 붙여 **하나의 표로 병합**합니다.
  - 행 단위가 다르면 **표마다 다른 인증코드**를 발급하고, 인증코드에 따라 해당 표를 응답합니다.

## 2. 응답

```json
{
  "meta": {
    "id": "progress",
    "title": "단원별 진도 현황",
    "source": "시험 서버",
    "updatedAt": "2026-10-18T09:00:00+09:00"
  },
  "columns": [
    { "key": "id", "label": "번호", "type": "string", "role": "id" }
  ],
  "rows": [
    { "id": "S01", "1단원": "완료", "2단원": null },
    { "id": "S02", "1단원": "완료", "2단원": "완료" }
  ]
}
```

### 수록 정보

이 규격은 자료의 형식만 정하며, 특정 정보(이름·학번 등)를 요구하지 않습니다. 응답에 포함할 정보는 자료 제공자(교사)가 결정합니다. 위 예시의 값은 모두 가상의 값입니다.

| 항목 | 필수 | 설명 |
|---|---|---|
| `rows` | 필수 | 행 객체의 배열. 키는 모든 행에서 모아 사용합니다. |
| `columns` | 선택 | 열별 표시 이름(`label`), 자료형(`type`: `number` 수치 / `string` 문자 / `boolean` 논리 / `mixed` 혼합), 식별 열 여부(`role`: `"id"`) |
| `meta` | 선택 | `id`(표 구분 이름), `title`(제목), `source`(출처), `updatedAt`(기준 시각) |
| `layout` | 선택 | 서버가 제공하는 기본 레이아웃 `{ "charts": [ … ], "groups": [ … ] }`. 두루보드의 [레이아웃 내보내기]로 만든 내용을 그대로 넣습니다. 사용자가 '서버에 저장된 레이아웃이 있으면 불러오기'를 선택한 경우에만 적용됩니다. |

- `layout.charts`의 각 항목 형식: `{ "kind": "chart" | "stat", "title": "학생별 진도율", "mode": "col" | "row", "cols": ["1단원", …], "label": "id", "read": "filled" | "blank" | "num", "how": "each" | "sum" | "avg" | "max" | "min", "type": "bar" | "hbar" | "line" | "hist" | "table" | "list", "pct": true | false }`
  - `kind`: 그래프(chart) 또는 통계(stat). 통계는 선택한 열의 모든 칸을 하나의 값으로 집계하며 `mode`·`label`·`type`을 사용하지 않습니다.
  - `mode`: 집계 기준 (행 기준 / 열 기준), `label`: 행 기준일 때 행 이름 열(막대마다 표시되는 이름)
  - `read`: 값 해석 (입력 여부 / 미입력 여부 / 숫자 값)
  - `pct`: 비율(%) 표시. 입력 여부·미입력 여부는 `값 ÷ 가능한 최댓값(칸 수)`, 숫자 값은 `값 ÷ 가장 큰 항목의 값`으로 환산합니다.
  - `type`의 `list`(명단)는 행 기준 그래프에서 해당 항목을 목록으로 표시합니다. `hl`(선택): `none` | `min` | `max` — 최저값·최고값 강조.
  - `type`의 `hist`(히스토그램)는 행 기준 그래프에서 값의 분포를 구간별 항목 수로 표시합니다.
  - `filter`(선택): 대상 행 조건. `{ "op": "and", "items": [ … ] }` 형식이며 `items`의 조건을 모두 만족하는 행만 사용합니다. 조건 형식:
    - 값 선택: `{ "col": "반", "kind": "vals", "vals": ["2-1", "2-2"], "blank": false, "not": false }` — 값 중 하나에 해당(`blank`가 참이면 빈칸 포함), `not`이 참이면 제외
    - 범위: `{ "col": "rate", "kind": "range", "lo": 0, "hi": 70, "not": false }` — 범위 안(경계 포함), `not`이 참이면 범위 밖
  - `range`(선택): 산출 범위. `{ "mode": "top" | "bottom" | "le" | "ge", "n": 10, "unit": "pct" | "cnt" }`. 상위·하위는 `n`%(pct) 또는 `n`개(cnt), le(≤)·ge(≥)는 기준값입니다. 행 기준 그래프는 항목별 값, 통계·열별 그래프는 행별 값을 기준으로 합니다.
  - `group`(선택): 개체가 속한 그룹의 `id`. `layout.groups`의 각 항목은 `{ "id": "g1", "name": "정보과학", "parent": null }` 형식이며, `parent`에 다른 최상위 그룹의 `id`를 넣으면 하위 그룹이 됩니다(최대 2단계, 그룹 최대 30개).
  - `how`: 계산 방식 (열별 개별 표시 / 합계 / 평균 / 최댓값 / 최솟값). `each`는 행 기준 그래프에서만 사용합니다.
- `layout.ai`(선택): 분석 파트너 설정. `{ "provider": "openai" | "anthropic" | "gemini" | "upstage", "model": "모델명", "columns": { "열 이름": "send" | "alias" | "drop" } }`. `columns`는 열별 전송 설정(전송·가명·제외)입니다. **인증 키는 넣지 않으며, 넣더라도 두루보드는 읽지 않습니다.**

서버 레이아웃 예시 (그룹 포함):

```json
"layout": {
  "groups": [
    { "id": "g-sum", "name": "요약", "parent": null },
    { "id": "g-info", "name": "정보과학", "parent": null },
    { "id": "g-info-a", "name": "A반", "parent": "g-info" }
  ],
  "charts": [
    { "id": "srv0", "group": "g-sum", "kind": "stat", "title": "전체 진도율", "cols": ["1단원", "2단원", "3단원"], "read": "filled", "how": "avg" },
    { "id": "srv1", "group": "g-info", "kind": "chart", "title": "단원별 완료 인원", "mode": "col", "cols": ["1단원", "2단원", "3단원"], "read": "filled", "how": "sum", "type": "bar" },
    { "id": "srv2", "group": "g-info-a", "kind": "chart", "title": "학생별 진도율", "mode": "row", "label": "id", "cols": ["1단원", "2단원", "3단원"], "read": "filled", "how": "avg", "type": "hbar" }
  ],
  "ai": { "provider": "upstage", "model": "solar-pro3", "columns": { "id": "alias" } }
}
```


## 3. Apps Script 연결 방식 (Google Apps Script 웹 앱)

Google Apps Script 웹 앱은 요청 머리글(`Authorization`)을 읽을 수 없고, 상태 코드와 사전 요청(OPTIONS)을 직접 정할 수 없습니다. 이 경우 아래 방식을 사용합니다. 응답 자료의 형식(2장)은 같습니다.

### 요청

```
POST https://script.google.com/macros/s/{배포 ID}/exec
Content-Type: text/plain;charset=utf-8

{"code":"{인증코드}","api":"duruboard/v1"}
```

- 교사는 두루보드의 주소 칸에 **웹 앱 주소 전체**(`…/exec`)를 입력합니다. 주소 뒤에 다른 경로를 붙이지 않습니다.
- 인증코드는 주소(쿼리)가 아니라 **요청 본문**으로 보냅니다.
- `Content-Type: text/plain`이므로 사전 요청(OPTIONS)이 발생하지 않습니다.
- Apps Script는 응답을 `script.googleusercontent.com`을 거쳐 전달합니다. 두루보드는 이 주소로의 전달만 허용합니다.

### 응답

- 정상: 2장과 같은 `{ "meta": …, "columns": …, "rows": … }`
- 오류: 상태 코드 대신 본문에 표시합니다.

```json
{ "error": { "status": 401, "message": "인증코드가 유효하지 않습니다." } }
```

| status | 의미 |
|---|---|
| 401 | 인증코드가 없거나 틀림 |
| 403 | 권한 없음 |
| 429 | 요청이 너무 많음 |
| 500 | 서버 오류 |

### 구현 예시 (Apps Script)

```javascript
function doPost(e) {
  let body = {};
  try { body = JSON.parse(e.postData.contents); } catch (err) {}
  if (!isValidCode(body.code)) return json({ error: { status: 401, message: '인증코드가 유효하지 않습니다.' } });
  return json({ meta: { id: 'progress', title: '활동지 제출 현황' }, columns: [ /* … */ ], rows: getRows() });
}
function doGet() { return json({ error: { status: 405, message: '두루보드에서 POST로 요청하십시오.' } }); }
function json(obj) {
  return ContentService.createTextOutput(JSON.stringify(obj)).setMimeType(ContentService.MimeType.JSON);
}
```

- 인증코드는 스크립트 속성(`PropertiesService`)에 해시(`Utilities.computeDigest`)로 저장합니다. 발급 시 연결 안내는 4장을 따릅니다.
- 배포: [배포] → [새 배포] → 유형 '웹 앱', 실행 계정 '나', 액세스 권한 '모든 사용자'. 인증코드가 없으면 자료를 응답하지 않으므로 자료는 보호됩니다.
- 학교(Google Workspace) 계정에서 '모든 사용자' 배포가 막혀 있으면 두루보드에서 연결할 수 없습니다. 이 경우 관리자에게 문의하십시오.

## 4. 인증코드 발급과 연결 안내 (공통)

자료 제공 측은 인증코드를 발급할 때, 교사가 두루보드에 그대로 입력할 수 있도록 연결 정보를 **한 번만** 표시합니다. 인증코드 원문은 저장하지 않고 해시로만 보관하므로 다시 보여 줄 수 없습니다.

```
===== 두루보드 연결 정보 =====
도메인 또는 웹 앱 주소: data.school.kr
인증코드: dur_8Kq3…(발급 값)
표: 활동지 제출 현황
==============================
```

- 일반 서버: 관리 화면이나 명령 출력에 표시합니다. 주소 칸에는 도메인을 표시합니다(`/api/v1/data`는 두루보드가 붙입니다).
- Apps Script: 발급 함수(예: `issueCode()`)를 편집기에서 실행하면 실행 로그(`Logger.log`)에 표시합니다. 웹 앱 주소는 `ScriptApp.getService().getUrl()`로 얻되, 배포 전이거나 `…/dev` 주소이면 "[배포] → [배포 관리]의 웹 앱 URL(…/exec)을 입력하십시오"라고 안내합니다.
