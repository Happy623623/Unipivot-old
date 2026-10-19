# .ics 내려받기 — 플래너의 마감·수업 일정을 기본 캘린더 앱에 넣는다

> 📝 기준: API 명세 v0.4 7장 · PRD v0.9 F-34 · RFC 5545 · 2026-10-19
>
> 적용할 곳: **탈퇴 PR까지 합친 main** 위 새 브랜치(앞 PR들과 겹치는 파일이 없어 순서가 바뀌어도 적용된다). **마이그레이션 없음**

F-34(P1)는 PRD 처음부터 있었는데 `GET /calendar/ics`가 코드에 없었다(라우터에 "F-34(P1)다"라는 주석
하나뿐). Google 캘린더를 쓰지 않는 학생 — iPhone 기본 캘린더, 삼성 캘린더, Outlook — 은 마감일을 손으로
옮겨 적어야 했다. 이제 플래너에서 단추 하나로 **오늘부터 1년 안의 준비한 공고 마감·LMS 과제 마감·확인한
수업 일정**을 .ics 파일 하나로 받는다. 기록은 남기지 않는다(PRD).

| 값 | 결과 |
| --- | --- |
| 테스트 | **1415 passed**(1399 → +16) · DB 없이 1074 passed, 341 skipped |
| 브라우저 | **실제 모드**(진짜 API·DB): 플래너 → ".ics 내려받기" → `unipivot-20261019.ics` → 일정 3개(공고·과제·시험), 지난 과제·끝낸 과제는 빠짐, `calendar_events` 0행 · 목 모드도 파일이 내려온다(API 호출 0) |
| 파일 검증 | 받은 파일을 `icalendar` 라이브러리(레포 밖)로 읽어 일정·시각·UID가 그대로 나오는지 봤다 |
| 되돌림 시험 | 돌연변이 **14개 전부 잡힘** |
| 시나리오 | 18/18 · 0.011040 USD · LLM 8회(앞과 같다 — 이 경로는 시나리오에 없다) |
| 마이그레이션 | 없음 |
| 새 파일 | 5개(api 2·web 3) · 고친 파일 4개(api 1·web 3) · 문서 spec 3곳·PRD 1곳 |

## 정한 것

1. **진입점은 플래너 한 곳, 한 번에 1년치다.** PRD 화면표가 F-34를 플래너 줄에 뒀다. 공고 상세마다
   단추를 두면 파일을 공고 수만큼 받아야 하는데, 캘린더 앱에 넣는 건 한 번이면 된다. 플래너 아래
   Google 연결 줄 밑에 ".ics 파일로 내려받기" 줄을 하나 더 뒀다. 화면은 `GET /planner/tasks`를 오늘부터
   365일로 다시 받아 대상을 모은다 — 마감 마커(준비를 마친 공고), 끝내지 않은 LMS 과제·계획서 할 일.
   오늘보다 앞선 날과 끝낸 할 일은 보내지 않는다.
2. **기록하지 않는다.** 명세 7장의 이 절은 스스로 모순이었다 — 위에는 "기록하지 않는다", 아래에는
   "동시에 `provider = ics`로 기록"이 같이 있었다. PRD F-34 줄("내려받기는 캘린더 등록 기록으로 남기지
   않는다")을 따라 옛 문장을 지웠다. 파일은 학생 손에 있고 우리는 그게 캘린더에 들어갔는지 알 수
   없어서 "넣음"이라고 적을 근거가 없다. `provider` 값 `ics`는 ERD에 그대로 둔다(앱으로 갈 때 기기
   캘린더용, PRD 캘린더 구조 메모).
3. **넣을 수 없는 대상은 건너뛴다. 하나도 없으면 404.** `POST /calendar/events`는 하나를 넣는 거라
   422·404·409로 바로 끝내지만, 내려받기는 여럿을 한 번에 받는다. 화면이 보낸 뒤 그새 마감이 지난 공고
   하나 때문에 파일 전체가 안 나오면 안 된다. 같은 조회 함수(`_opportunity_target` 등)를 그대로 쓰고
   그 셋(404·409·422)만 건너뛴다 — DB가 없는 것 같은 오류(503)는 그대로 올린다. 다 빠지면
   `404 NOT_FOUND`("내려받을 일정이 없어요.").
4. **대상에 계획서 일정(`syllabus_item_ids`)을 더했다.** 명세의 옛 모양은 공고·LMS 과제뿐이었는데 PRD는
   "마감일·**시험 일정**"이라 했고, 시험 일정은 계획서에서만 온다. 10/17 보강에서 Google 등록에
   `syllabus_item_id`가 들어간 것과 같은 자리다.
5. **일정 모양은 Google에 넣을 때와 똑같다.** 제목(`[마감] …`, `[과제] 과목 · 제목`, `[시험] 과목 ·
   제목`)과 시각(마감으로 끝나는 1시간, 계획서 일정은 시작부터 1시간) 모두 같은 코드에서 나온다. 한
   학생이 Google과 .ics를 섞어 써도 둘이 다르게 보이지 않는다.
6. **시각은 UTC(`Z`)로 적는다.** `DTSTART;TZID=Asia/Seoul`로 쓰면 VTIMEZONE 블록을 같이 넣어야 하고
   빠뜨리면 앱마다 다르게 읽는다. `Z`는 어느 앱이나 그 기기 시간대로 바꿔 보여 준다.
7. **`UID`는 `<종류>-<id>@unipivot`이다.** 같은 파일을 다음 달에 또 받아 넣어도 캘린더 앱이 같은 UID를
   하나로 합친다(복사본이 안 생긴다). Google 등록의 일정 번호를 행 id에서 만드는 것과 같은 생각이다.
8. **표준 라이브러리로 직접 만든다.** PRD 기술표에 "icalendar 라이브러리 등"으로 있었는데, VEVENT 몇
   개에 필요한 건 줄 접기(75바이트, 한글 한가운데서 안 자르기)와 글자 피하기(`\` `;` `,` 줄바꿈)뿐이라
   `app/calendar/ics.py` 85줄이다. 의존을 더하면 `uv.lock`과 네 PC의 `uv sync`가 따라오는데 그럴 만한
   양이 아니다. 대신 **받은 파일을 `icalendar` 라이브러리로 읽어** 제대로 읽히는지 샌드박스에서 확인했다
   (레포에는 넣지 않았다). PRD 기술표 줄을 고쳤다.
9. **한 번에 200개까지.** 대상 하나가 조회 하나라서 상한을 뒀다. uuid 200개면 주소가 7.5KB인데 uvicorn이
   그대로 받는 걸 실제 서버로 확인했다(201개는 422).
10. **파일 이름은 `unipivot-YYYYMMDD.ics`(KST 오늘)다.** 서버는 `Content-Disposition`으로 주고, 화면은
    같은 이름을 스스로 만든다(브라우저가 CORS 응답 헤더를 못 읽어서 `expose_headers`를 여는 대신).
11. **화면은 `window`를 만지지 않는다.** 파일로 떨어뜨리는 DOM 조작(`URL.createObjectURL` + `<a download>`)은
    `src/lib/saveFile.ts`에, JSON이 아닌 응답을 받는 `apiFetchText`는 `http.ts`에 뒀다(`apiFetch`와
    같은 머리 — 토큰·401·오류 처리 — 를 `apiResponse`로 나눠 둘이 같이 쓴다). `client.ts`의
    `downloadPlannerIcs()`가 받아서 떨어뜨리고, 받을 게 없으면 파일을 만들지 않고 `false`다.
12. **목 모드는 축소판 파일을 준다.** 같은 대상 고르기(`lib/ics.ts`)로 고른 뒤 18:00 KST로 끝나는
    1시간짜리를 만든다(줄 접기 없음). 상태 미리보기용이다.

## 레포에 넣는 법

````text
[0] 인수인계 순서대로면 탈퇴 PR까지 main에 들어가 있다(겹치는 파일이 없어 먼저 넣어도 적용은 된다).
    git checkout main && git pull
    git checkout -b feat/ics-download

[1] "코드" 절의 새 파일 5개를 그대로 만들어 줘.
    api/app/calendar/ics.py
    api/tests/test_calendar_ics.py
    web/src/lib/ics.ts
    web/src/lib/saveFile.ts
    web/src/mocks/ics.ts

[2] "패치" 절의 내용을 그대로 ics.patch로 저장해서 적용해 줘(고친 파일 4개).
    git apply ics.patch
    (적용한 뒤 ics.patch는 지워 줘)

[3] "문서에 반영하기" 절대로 docs/API_명세_v0.4.md 3곳과 docs/PRD_v0.9.md 1곳을 고쳐 줘.

[4] 이 문서도 docs/ics내려받기_구현.md로 같이 넣어 줘.

[5] 검사: api> uv run ruff check . && uv run ruff format --check . && uv run pytest
    web> npm run typecheck && npm run build
    DB 없이 `1074 passed, 341 skipped` 근처(네 PC는 보통 passed 하나 적고 skipped 하나 많다),
    CI에서는 `1415 passed`여야 한다(탈퇴 PR 다음이면).

[6] 통과하면 "feat(F-34): 플래너에서 마감·수업 일정을 .ics 파일로 내려받는다"로 커밋해서 PR을 올려 줘.
    DB 변경이 없으니 db push는 필요 없다. 합친 뒤 따로 할 일은 없다.
````

## 합친 뒤 확인

````text
web> npm run dev  (API도 켜져 있어야 한다)

1. 플래너 맨 아래 ".ics 파일로 내려받기" 줄의 ".ics 내려받기"를 누르면 unipivot-날짜.ics가 떨어지고
   "내려받았어요" 안내가 뜬다. 준비한 공고·LMS 과제·확인한 계획서 일정이 하나도 없으면
   "내려받을 일정이 없어요"다(그때는 공고 하나를 준비하고 다시).
2. 파일을 열어 보면 BEGIN:VCALENDAR로 시작하고 일정마다 SUMMARY가 [마감]·[과제]·[시험]으로 붙어 있다.
3. 휴대폰에서 한 번: 파일을 iPhone(메일·AirDrop)이나 삼성 폰으로 보내 열면 "일정 추가"가 뜨고,
   넣은 뒤 그 날짜에 마감 1시간 전 일정이 보인다. 같은 파일을 한 번 더 넣어도 복사본이 안 생긴다.
   (Windows면 Outlook이나 기본 캘린더 앱에서 열어도 된다)
````

## 구조

```
api/app/calendar/ics.py           IcsEvent · render_calendar(): VCALENDAR 글 만들기(줄 접기·글자 피하기·UTC)
api/app/routers/calendar.py       GET /calendar/ics: id 목록 → 같은 조회 함수로 대상 → 건너뛰기 → 파일 응답
web/src/lib/ics.ts                icsQuery(): 플래너 응답에서 대상 고르기(마커·끝내지 않은 과제·계획서 할 일)
web/src/lib/saveFile.ts           saveTextFile(): 글을 파일로 떨어뜨린다(DOM은 여기서만)
web/src/api/http.ts               apiResponse()로 머리를 나누고 apiFetchText() 추가
web/src/api/client.ts             downloadPlannerIcs(): 1년치 플래너 → 쿼리 → 받기 → 저장
web/src/mocks/ics.ts              목 모드 축소판 파일
web/src/screens/Planner.tsx       ".ics 파일로 내려받기" 줄 + 단추 + 안내
```

## 확인한 것

| 확인 | 결과 |
| --- | --- |
| 브라우저(실제 모드) | 준비한 공고(마감 +10일)·LMS 과제(+4일)·지난 과제(−1일)·끝낸 과제(+5일)·확인한 계획서 일정(+6일)을 넣은 사용자로 플래너 → ".ics 내려받기" → 보낸 요청이 `opportunity_ids=…&lms_assignment_ids=…&syllabus_item_ids=…` **셋 하나씩**(지난 것·끝낸 것 빠짐) → `unipivot-20261019.ics` → `icalendar`로 읽으면 `[과제] 데이터베이스 · 실습 2주차 결과 제출`(마감 1시간 전부터) · `[시험] 데이터베이스 · 중간고사` · `[마감] E2E ics 장학금` → `calendar_events` 0행 → 안내 문구. 페이지 오류 0 |
| 브라우저(목 모드) | 플래너 → 단추 → `unipivot-20261005.ics`(목 기준일) 내려옴, API 호출 0, 페이지 오류 0 |
| 실제 서버로 | uuid 200개(주소 7.5KB) → 404(없는 id라) · 201개 → 422 `fields.ids` · `abc` → 422 `fields.lms_assignment_ids` · 비어 있음 → 422 `fields.ids` · 로그인 없이 → 401 |
| 단위(파일) | `;` `,` `\` 줄바꿈 피하기 · 120바이트 한글 줄을 접어도 글자가 안 깨짐(각 줄 75바이트 이하) · KST 시각이 `…Z`로 · 시간대 없는 시각은 거절 · 시작 시각 순 · CRLF |
| 단위(API) | 세 종류가 Google과 같은 제목·시각으로 한 파일에, 기록 없음 · 없는 것·지난 것·날짜 없는 것은 빠지고 나머지로 · 다 빠지면 404 · 같은 id 두 번은 하나 · uuid 아님·비어 있음·201개는 422 · 200개는 404 · 파일 이름은 KST 날짜 · 503은 건너뛰지 않음 · 로그인 없으면 401 |
| 테스트 | 1415 passed(DB 포함) · DB 없이 1074 passed, 341 skipped · ruff check·format 통과 |
| web | `npm run typecheck`·`npm run build` 통과 |
| 시나리오 | 18/18 · 0.011040 USD · LLM 8회 — 앞과 같다 |

SQL은 새로 없다 — 대상 조회(`prepared_opportunity`·`assignment_deadline`·`syllabus_deadline`)는
Google 등록이 쓰던 것 그대로이고 `test_calendar_db.py`가 이미 본다(남의 것·수강 뺀 과목·확인 전 일정).

## 되돌림 시험

14가지를 일부러 고장 내고 시험이 잡는지 봤다. 전부 잡혔다.

| # | 고장 낸 것 | 잡은 시험 |
| --- | --- | --- |
| M1 | 줄을 접을 때 한글 한가운데서 자른다 | 단위 — 접은 줄을 이어 붙이면 원문 |
| M2 | `;`를 피하지 않는다 | 단위 — 피하기 |
| M3 | UTC로 바꾸지 않고 `Z`만 붙인다 | 단위 — KST 18:00이 `090000Z` + API 시각 |
| M4 | 시작 시각 순으로 늘어놓지 않는다 | 단위 — 순서 |
| M5 | 줄 끝을 LF로 쓴다 | 단위 — CRLF(네 시험) |
| M6 | 넣을 수 없는 대상을 건너뛰지 않는다(전체 실패) | API — 건너뛰기(세 시험) |
| M7 | 모든 오류를 건너뛴다(DB 오류도) | API — 503은 그대로 |
| M8 | 하나도 없어도 빈 파일을 준다 | API — 404 |
| M9 | 같은 id를 두 번 넣는다 | API — 하나 |
| M10 | 상한이 199개다 | API — 200개는 404 |
| M11 | 일정이 마감 시각에 시작한다 | API — `DTSTART` |
| M12 | 파일 이름 날짜를 UTC로 쓴다 | API — KST 16일 |
| M13 | `Content-Type`이 `text/plain`이다 | API — 헤더 |
| M14 | `UID`에 대상 종류가 없다 | API — UID(두 시험) |

web 쪽(끝낸 할 일·지난 날을 빼는 것)은 단위 시험이 없어 실제 모드 브라우저 확인이 본다 — 다섯 대상 중
셋만 보냈다.

## 검토에서 고친 것

| 본 것 | 고친 것 |
| --- | --- |
| 명세 7장 `GET /calendar/ics` 절이 "기록하지 않는다"와 "`provider = ics`로 기록"을 같이 말하고 있었다 | PRD F-34를 따라 기록하지 않는 쪽으로 절을 다시 썼다(정한 것 2) |
| `utc_stamp` 시험이 UTC 시각만 줘서, 시간대 변환을 빼도(M3) 단위 시험은 못 잡고 API 시험만 잡았다 | KST 18:00 → `090000Z`로 바꿨다 |
| 건너뛰기를 "모든 `ApiError`"로 하면 DB가 없을 때(503)도 빈 파일·404가 나와 원인이 묻힌다 | 404·409·422만 건너뛰고 나머지는 올린다 + 시험(M7) |
| 파일 이름 날짜를 `now`에서 바로 만들면 UTC 자정 전후에 어제 날짜가 된다 | `now.astimezone(KST)` + 시험(M12) |
| 목 모드 축소판이 마커 제목(`… 마감`)에 `[마감]`을 또 붙여 "[마감] 생활지원 장학금 마감"이 됐다 | 뒤의 " 마감"을 떼고 붙인다 |
| 라우터의 "`.ics` 내려받기는 F-34(P1)다" 주석과 명세의 "(`ics`는 F-34)"가 아직 없는 기능처럼 읽힌다 | ".ics는 등록이 아니라 내려받기다(`GET /calendar/ics`)"로 고쳤다 |

## 남은 것

- **휴대폰에서 실제로 넣어 보는 것**은 네 몫이다("합친 뒤 확인" 3번). 샌드박스는 파일을 읽어 봤을 뿐
  캘린더 앱에 넣지는 못한다.
- 공고 상세나 과목 화면의 **개별 .ics**는 두지 않았다. 필요하면 같은 API에 id 하나만 보내면 된다
  (화면 단추 하나).
- 플래너가 못 보는 것은 파일에도 없다 — 준비를 **마치지 않은** 공고, 날짜가 없는 계획서 일정, 마감
  없는 상시 공고. 1년 밖의 일정도(`ICS_DAYS`).
- 구독 주소(webcal://, 바뀌면 자동 반영)는 다른 설계다 — 사용자별 비밀 주소가 필요하다. 지금은 파일
  한 번이고, 다시 받아 넣으면 UID로 합쳐진다.

## 문서에 반영하기

````text
docs/API_명세_v0.4.md — 3곳, docs/PRD_v0.9.md — 1곳

[1] docs/API_명세_v0.4.md 머리말. 이 줄을 찾아서

> v0.4 보강(10/13, 시·군·구 선택): 프로필 region_sigungu 값

그 줄 **앞에** 이 두 줄을 붙여 줘(바로 앞 줄은 `>` 하나다. 앞 PR들의 10/19 줄 다음에 오게 된다).

> v0.4 보강(10/19, .ics 내려받기): `GET /calendar/ics`를 만들었다(F-34). 대상에 `syllabus_item_ids`를 더했고, 넣을 수 없는 대상은 건너뛰며, 기록은 남기지 않는다(PRD와 맞춤 — "provider = ics로 기록" 문장을 지웠다)
>

[2] docs/API_명세_v0.4.md 7장 `POST /calendar/events`. 이 글자를 찾아서

`provider`는 `google`만 받는다(`ics`는 F-34)

이렇게 바꿔 줘.

`provider`는 `google`만 받는다(.ics는 등록이 아니라 내려받기다 — `GET /calendar/ics`)

[3] docs/API_명세_v0.4.md 7장. 이 절 전체(제목부터 "## 8. 강의계획서 · 강의자료" 앞까지)를

### `GET /calendar/ics` (P1)

- 내려받기는 `calendar_events`에 기록하지 않는다

`?opportunity_ids=uuid,uuid&lms_assignment_ids=uuid` → `Content-Type: text/calendar`, 파일 다운로드. 동시에 `calendar_events`에 `provider = ics`로 기록.

이 절로 바꿔 줘.

### `GET /calendar/ics` (P1)

`?opportunity_ids=uuid,uuid&lms_assignment_ids=uuid&syllabus_item_ids=uuid` → `200`, `Content-Type: text/calendar; charset=utf-8`, `Content-Disposition: attachment; filename="unipivot-YYYYMMDD.ics"`(KST 오늘). Google 캘린더를 쓰지 않는 학생이 iPhone·삼성·Outlook 캘린더에 넣는 파일이다(F-34).

- 대상은 `POST /calendar/events`와 같다 — 준비를 마친 공고의 마감(`opportunity_ids`), LMS 과제 마감(`lms_assignment_ids`), 확인한 계획서 일정(`syllabus_item_ids`). 각 파라미터는 쉼표로 나눈 uuid 목록이고 셋을 합쳐 하나 이상, 200개까지다. uuid가 아닌 값이 있으면 `422 VALIDATION_FAILED`(`details.fields`에 그 파라미터 이름), 하나도 없거나 200개를 넘으면 `422`(`details.fields.ids`)
- 일정 하나는 Google에 넣을 때와 같다: 제목이 같고(`[마감] …`, `[과제] 과목 · 제목`, `[시험] 과목 · 제목`), 마감 시각으로 끝나는 1시간(계획서 일정은 시작 시각부터 1시간)이다. 시각은 UTC(`Z`)로 적는다
- 넣을 수 없는 대상은 **건너뛴다** — 준비하지 않은 공고, 마감·날짜가 없는 것, 이미 지난 것, 남의 것이거나 없는 것. 하나도 남지 않으면 `404 NOT_FOUND`("내려받을 일정이 없어요."). 같은 id를 두 번 주면 한 번만 넣는다
- `UID`는 `<종류>-<id>@unipivot`(`opportunity`·`lms_assignment`·`syllabus_item`)이다. 같은 파일을 두 번 넣어도 캘린더 앱이 하나로 합친다
- 내려받기는 `calendar_events`에 **기록하지 않는다**(PRD F-34). 공고 상세·과목 화면의 "캘린더에 넣음"은 Google 등록만 뜻한다
- 화면은 플래너에서 오늘부터 1년 안의 대상(마감 마커, 끝내지 않은 LMS 과제·계획서 할 일)을 모아 한 번에 받는다

[4] docs/PRD_v0.9.md 9장 기술 스택 표. 이 줄을 찾아서

| ICS 처리 | Python icalendar 라이브러리 등 | .ics 내려받기 파일 생성 |

이 줄로 바꿔 줘.

| ICS 처리 | 표준 라이브러리로 직접 생성(RFC 5545, `app/calendar/ics.py`) | .ics 내려받기 파일 생성. 일정 몇 개뿐이라 줄 접기·글자 피하기만 있으면 된다 |
````

## 코드

새로 만드는 파일 5개다.

#### `api/app/calendar/ics.py`

```python
""".ics 파일 만들기 (F-34, RFC 5545).

Google 캘린더를 쓰지 않는 학생이 마감일·수업 일정을 iPhone·삼성·Outlook 캘린더에 넣는 파일이다.
표준 라이브러리만으로 만든다 — VEVENT 몇 개에 필요한 건 줄 접기(75바이트)와 글자 피하기뿐이고,
시각은 모두 UTC(`Z`)로 적어 시간대 블록(VTIMEZONE) 없이도 어느 앱에서나 같은 시각이 된다.
"""

from collections.abc import Sequence
from dataclasses import dataclass
from datetime import UTC, datetime

PRODID = "-//UNIPIVOT//UNIPIVOT Calendar//KO"
UID_DOMAIN = "unipivot"
LINE_LIMIT = 75  # RFC 5545 3.1: 한 줄은 75바이트까지, 넘치면 다음 줄을 공백 하나로 시작한다


@dataclass(frozen=True)
class IcsEvent:
    """파일에 들어갈 일정 하나. `starts_at`·`ends_at`은 시간대가 있는 시각이어야 한다."""

    uid: str  # 대상 id로 만든다. 같은 파일을 두 번 넣어도 캘린더 앱이 하나로 합친다
    title: str
    starts_at: datetime
    ends_at: datetime


def event_uid(kind: str, target_id: str) -> str:
    """`opportunity-<uuid>@unipivot`처럼 대상 종류와 id로 만든 UID."""
    return f"{kind}-{target_id}@{UID_DOMAIN}"


def escape_text(value: str) -> str:
    """TEXT 값의 특수 글자(역슬래시·`;`·`,`·줄바꿈)를 RFC 5545 3.3.11대로 피한다."""
    return (
        value.replace("\\", r"\\")
        .replace(";", r"\;")
        .replace(",", r"\,")
        .replace("\r\n", "\n")
        .replace("\n", r"\n")
    )


def fold_line(line: str) -> str:
    """75바이트가 넘는 줄을 접는다. 한글(3바이트) 한가운데서는 자르지 않는다."""
    data = line.encode("utf-8")
    parts: list[bytes] = []
    while len(data) > LINE_LIMIT:
        cut = LINE_LIMIT
        while cut > 1 and (data[cut] & 0xC0) == 0x80:  # UTF-8 이어지는 바이트면 앞으로 물린다
            cut -= 1
        parts.append(data[:cut])
        data = b" " + data[cut:]
    parts.append(data)
    return b"\r\n".join(parts).decode("utf-8")


def utc_stamp(value: datetime) -> str:
    """`20261024T090000Z` 꼴. 시간대 없는 시각은 받지 않는다."""
    if value.tzinfo is None:
        raise ValueError("시간대가 없는 시각이에요.")
    return value.astimezone(UTC).strftime("%Y%m%dT%H%M%SZ")


def render_calendar(events: Sequence[IcsEvent], *, now: datetime) -> str:
    """VCALENDAR 문서. 줄 끝은 CRLF이고 일정은 시작 시각·제목 순이다."""
    lines = [
        "BEGIN:VCALENDAR",
        "VERSION:2.0",
        f"PRODID:{PRODID}",
        "CALSCALE:GREGORIAN",
        "METHOD:PUBLISH",
    ]
    stamp = utc_stamp(now)
    for event in sorted(events, key=lambda one: (one.starts_at, one.title)):
        lines += [
            "BEGIN:VEVENT",
            f"UID:{event.uid}",
            f"DTSTAMP:{stamp}",
            f"DTSTART:{utc_stamp(event.starts_at)}",
            f"DTEND:{utc_stamp(event.ends_at)}",
            f"SUMMARY:{escape_text(event.title)}",
            "END:VEVENT",
        ]
    lines.append("END:VCALENDAR")
    return "\r\n".join(fold_line(line) for line in lines) + "\r\n"
```

#### `api/tests/test_calendar_ics.py`

```python
""".ics 내려받기(F-34): 파일 만들기와 `GET /calendar/ics`. 대상 조회는 test_calendar_api.py의 가짜 저장소를 쓴다."""

from collections.abc import Iterator
from datetime import UTC, datetime, timedelta

import pytest
from fastapi.testclient import TestClient

from app.calendar.ics import IcsEvent, escape_text, event_uid, fold_line, render_calendar, utc_stamp
from app.clock import current_time
from app.config import Settings, get_settings
from app.eligibility import KST
from app.errors import ApiError
from app.main import create_app
from app.repositories.calendar import AssignmentDeadline, SyllabusDeadline, get_calendar_repo
from tests.conftest import make_token
from tests.test_calendar_api import (
    ASSIGNMENT_ID,
    DEADLINE,
    NOW,
    OPPORTUNITY_ID,
    SYLLABUS_ITEM_ID,
    FakeCalendarRepository,
)

ICS = "/api/v1/calendar/ics"
UNKNOWN_ID = "00000000-0000-4000-8000-0000000000ff"


def unfold(body: str) -> list[str]:
    """접은 줄을 다시 편다(RFC 5545 3.1). 확인할 때만 쓴다."""
    lines: list[str] = []
    for line in body.split("\r\n"):
        if line.startswith(" ") and lines:
            lines[-1] += line[1:]
        else:
            lines.append(line)
    return lines


def events_in(body: str) -> list[dict[str, str]]:
    found: list[dict[str, str]] = []
    for line in unfold(body):
        if line == "BEGIN:VEVENT":
            found.append({})
        elif found and ":" in line and line not in ("END:VEVENT", "END:VCALENDAR"):
            key, value = line.split(":", 1)
            found[-1].setdefault(key, value)
    return found


@pytest.fixture
def calendar(settings: Settings) -> FakeCalendarRepository:
    return FakeCalendarRepository(settings.token_encryption_key)


@pytest.fixture
def client(settings: Settings, calendar: FakeCalendarRepository) -> Iterator[TestClient]:
    app = create_app(settings)
    app.dependency_overrides[get_settings] = lambda: settings
    app.dependency_overrides[get_calendar_repo] = lambda: calendar
    app.dependency_overrides[current_time] = lambda: NOW
    with TestClient(app) as test_client:
        yield test_client


@pytest.fixture
def auth() -> dict[str, str]:
    return {"Authorization": f"Bearer {make_token()}"}


# ---- 파일 만들기 ----


def test_text_is_escaped_and_long_lines_fold_between_characters() -> None:
    assert escape_text("a;b,c\\d\r\ne\nf") == r"a\;b\,c\\d\ne\nf"
    folded = fold_line("SUMMARY:" + "가" * 40)  # 8 + 120바이트
    pieces = folded.split("\r\n")
    assert len(pieces) == 2 and pieces[1].startswith(" ")
    assert all(len(piece.encode()) <= 75 for piece in pieces)
    assert "".join([pieces[0], pieces[1][1:]]) == "SUMMARY:" + "가" * 40  # 글자가 깨지지 않는다


def test_stamps_are_utc_and_naive_times_are_refused() -> None:
    assert utc_stamp(datetime(2026, 10, 24, 18, 0, tzinfo=KST)) == "20261024T090000Z"
    with pytest.raises(ValueError):
        utc_stamp(datetime(2026, 10, 24, 18, 0))


def test_render_orders_by_start_and_ends_lines_with_crlf() -> None:
    later = IcsEvent(event_uid("opportunity", "b"), "둘째", DEADLINE, DEADLINE + timedelta(hours=1))
    first = IcsEvent(event_uid("opportunity", "a"), "첫째", DEADLINE - timedelta(days=1), DEADLINE)
    body = render_calendar([later, first], now=NOW)
    assert body.endswith("END:VCALENDAR\r\n") and "\n" not in body.replace("\r\n", "")
    assert unfold(body)[:3] == [
        "BEGIN:VCALENDAR",
        "VERSION:2.0",
        "PRODID:-//UNIPIVOT//UNIPIVOT Calendar//KO",
    ]
    found = events_in(body)
    assert [one["SUMMARY"] for one in found] == ["첫째", "둘째"]
    assert found[0]["UID"] == "opportunity-a@unipivot" and found[0]["DTSTAMP"] == "20261015T030000Z"


# ---- GET /calendar/ics ----


def test_ics_requires_login(client: TestClient) -> None:
    assert client.get(ICS, params={"opportunity_ids": OPPORTUNITY_ID}).status_code == 401


def test_the_file_holds_all_three_kinds_and_records_nothing(
    client: TestClient, auth: dict[str, str], calendar: FakeCalendarRepository
) -> None:
    """공고 마감·LMS 과제·계획서 일정이 Google에 넣을 때와 같은 시각·제목으로 들어간다."""
    res = client.get(
        ICS,
        params={
            "opportunity_ids": OPPORTUNITY_ID,
            "lms_assignment_ids": ASSIGNMENT_ID,
            "syllabus_item_ids": SYLLABUS_ITEM_ID,
        },
        headers=auth,
    )
    assert res.status_code == 200
    assert res.headers["content-type"] == "text/calendar; charset=utf-8"
    assert res.headers["content-disposition"] == 'attachment; filename="unipivot-20261015.ics"'
    found = {one["UID"]: one for one in events_in(res.text)}
    assert set(found) == {
        f"opportunity-{OPPORTUNITY_ID}@unipivot",
        f"lms_assignment-{ASSIGNMENT_ID}@unipivot",
        f"syllabus_item-{SYLLABUS_ITEM_ID}@unipivot",
    }
    deadline = found[f"opportunity-{OPPORTUNITY_ID}@unipivot"]
    assert deadline["SUMMARY"] == "[마감] 한양브레인 장학금"
    assert (deadline["DTSTART"], deadline["DTEND"]) == ("20261024T080000Z", "20261024T090000Z")
    assignment = found[f"lms_assignment-{ASSIGNMENT_ID}@unipivot"]
    assert assignment["SUMMARY"] == "[과제] 데이터베이스 · 실습 2주차 결과 제출"
    assert (assignment["DTSTART"], assignment["DTEND"]) == ("20261024T080000Z", "20261024T090000Z")
    exam = found[f"syllabus_item-{SYLLABUS_ITEM_ID}@unipivot"]
    assert exam["SUMMARY"] == "[시험] 데이터베이스 · 중간고사"
    assert (exam["DTSTART"], exam["DTEND"]) == ("20261024T090000Z", "20261024T100000Z")
    assert calendar.added == [] and calendar.events == {}  # 내려받기는 기록하지 않는다


def test_targets_that_cannot_go_in_are_skipped(
    client: TestClient, auth: dict[str, str], calendar: FakeCalendarRepository
) -> None:
    """없는 것·지난 것·마감 없는 것은 빠지고 나머지로 파일을 만든다."""
    calendar.assignment = AssignmentDeadline(
        ASSIGNMENT_ID, "지난 과제", "데이터베이스", NOW - timedelta(hours=1)
    )
    calendar.syllabus = SyllabusDeadline(
        SYLLABUS_ITEM_ID, "날짜 없는 일정", "데이터베이스", "exam", None
    )
    res = client.get(
        ICS,
        params={
            "opportunity_ids": f"{OPPORTUNITY_ID},{UNKNOWN_ID}",
            "lms_assignment_ids": ASSIGNMENT_ID,
            "syllabus_item_ids": SYLLABUS_ITEM_ID,
        },
        headers=auth,
    )
    assert res.status_code == 200
    assert [one["SUMMARY"] for one in events_in(res.text)] == ["[마감] 한양브레인 장학금"]


def test_nothing_left_is_404(
    client: TestClient, auth: dict[str, str], calendar: FakeCalendarRepository
) -> None:
    calendar.opportunity = None  # 준비하지 않은 공고
    res = client.get(ICS, params={"opportunity_ids": OPPORTUNITY_ID}, headers=auth)
    assert res.status_code == 404
    assert res.json()["error"] == {
        "code": "NOT_FOUND",
        "message": "내려받을 일정이 없어요.",
        "details": {},
    }


def test_the_same_id_twice_is_one_event(client: TestClient, auth: dict[str, str]) -> None:
    res = client.get(
        ICS, params={"opportunity_ids": f"{OPPORTUNITY_ID}, {OPPORTUNITY_ID},"}, headers=auth
    )
    assert res.status_code == 200 and len(events_in(res.text)) == 1


def test_the_filename_uses_the_kst_date(
    settings: Settings, calendar: FakeCalendarRepository, auth: dict[str, str]
) -> None:
    """UTC로는 아직 15일이어도 KST로 16일이면 16일 파일이다."""
    app = create_app(settings)
    app.dependency_overrides[get_settings] = lambda: settings
    app.dependency_overrides[get_calendar_repo] = lambda: calendar
    app.dependency_overrides[current_time] = lambda: datetime(2026, 10, 15, 16, 0, tzinfo=UTC)
    with TestClient(app) as late_client:
        res = late_client.get(ICS, params={"opportunity_ids": OPPORTUNITY_ID}, headers=auth)
    assert res.status_code == 200
    assert res.headers["content-disposition"] == 'attachment; filename="unipivot-20261016.ics"'


def test_other_errors_are_not_skipped(
    client: TestClient, auth: dict[str, str], calendar: FakeCalendarRepository
) -> None:
    """건너뛰는 건 넣을 수 없는 대상(404·409·422)뿐이다. DB가 없는 것 같은 오류는 그대로 올라간다."""

    async def unavailable(user_id: str, opportunity_id: str) -> None:
        raise ApiError(503, "DB_UNAVAILABLE", "잠시 후 다시 시도해 주세요.")

    calendar.prepared_opportunity = unavailable  # type: ignore[method-assign]
    res = client.get(ICS, params={"opportunity_ids": OPPORTUNITY_ID}, headers=auth)
    assert res.status_code == 503 and res.json()["error"]["code"] == "DB_UNAVAILABLE"


@pytest.mark.parametrize(
    ("params", "field"),
    [
        ({"opportunity_ids": "abc"}, "opportunity_ids"),
        ({"lms_assignment_ids": f"{ASSIGNMENT_ID},x"}, "lms_assignment_ids"),
        ({"syllabus_item_ids": "1"}, "syllabus_item_ids"),
        ({}, "ids"),
        ({"opportunity_ids": ","}, "ids"),
    ],
)
def test_bad_or_missing_ids_are_422(
    client: TestClient, auth: dict[str, str], params: dict[str, str], field: str
) -> None:
    res = client.get(ICS, params=params, headers=auth)
    assert res.status_code == 422
    assert res.json()["error"]["code"] == "VALIDATION_FAILED"
    assert list(res.json()["error"]["details"]["fields"]) == [field]


def test_too_many_ids_are_422(client: TestClient, auth: dict[str, str]) -> None:
    many = ",".join(f"00000000-0000-4000-8000-{index:012d}" for index in range(201))
    res = client.get(ICS, params={"opportunity_ids": many}, headers=auth)
    assert res.status_code == 422
    assert res.json()["error"]["details"]["fields"] == {"ids": "한 번에 200개까지 받을 수 있어요."}
    just_enough = ",".join(f"00000000-0000-4000-8000-{index:012d}" for index in range(200))
    assert client.get(ICS, params={"opportunity_ids": just_enough}, headers=auth).status_code == 404
```

#### `web/src/lib/ics.ts`

```ts
import type { PlannerResponse } from "@/types/api";

// .ics 내려받기(F-34)의 대상 고르기. 서버(`GET /calendar/ics`)가 받는 쿼리 문자열을 만든다

// 오늘부터 이만큼 앞까지 받는다. 플래너 조회 상한(400일) 안이다
export const ICS_DAYS = 365;

// "YYYY-MM-DD"에 날을 더한다(KST 날짜 문자열끼리라 시간대가 끼어들지 않는다)
export function daysLater(date: string, days: number): string {
  const base = Date.UTC(Number(date.slice(0, 4)), Number(date.slice(5, 7)) - 1, Number(date.slice(8, 10)));
  return new Date(base + days * 86_400_000).toISOString().slice(0, 10);
}

// 플래너에서 캘린더에 넣을 수 있는 것: 준비를 마친 공고의 마감(마커), 아직 끝내지 않은 LMS 과제와
// 계획서 일정. 오늘보다 앞선 날은 뺀다(서버도 지난 것은 건너뛴다). 받을 게 없으면 null
export function icsQuery(planner: PlannerResponse, today: string): string | null {
  const open = planner.items.filter((task) => !task.is_done && task.due_date >= today);
  const lists: [string, string[]][] = [
    ["opportunity_ids", planner.markers.filter((m) => m.date >= today).map((m) => m.opportunity_id)],
    ["lms_assignment_ids", open.filter((t) => t.link.type === "lms_assignment").map((t) => t.link.id ?? "")],
    ["syllabus_item_ids", open.filter((t) => t.link.type === "syllabus_item").map((t) => t.link.id ?? "")],
  ];
  const query = new URLSearchParams();
  for (const [name, ids] of lists) {
    const given = Array.from(new Set(ids.filter(Boolean)));
    if (given.length) query.set(name, given.join(","));
  }
  const text = query.toString();
  return text ? text : null;
}
```

#### `web/src/lib/saveFile.ts`

```ts
// 글을 파일로 내려받게 한다(.ics 등). 화면 코드 대신 여기서만 문서(DOM)를 만진다 (web/CLAUDE.md)
export function saveTextFile(filename: string, text: string, type: string): void {
  const url = URL.createObjectURL(new Blob([text], { type }));
  const anchor = document.createElement("a");
  anchor.href = url;
  anchor.download = filename;
  anchor.click();
  setTimeout(() => URL.revokeObjectURL(url), 1_000); // 내려받기가 시작된 뒤에 주소를 거둔다
}
```

#### `web/src/mocks/ics.ts`

```ts
import { icsQuery } from "@/lib/ics";
import type { PlannerResponse } from "@/types/api";

// 목 모드의 .ics 미리보기. 서버가 주는 파일의 축소판이다(일정은 그날 18:00 KST로 끝나는 1시간, 줄 접기 없음)
export function mockIcs(planner: PlannerResponse, today: string): string {
  const query = new URLSearchParams(icsQuery(planner, today) ?? "");
  const chosen = (name: string) => new Set((query.get(name) ?? "").split(",").filter(Boolean));
  const opportunities = chosen("opportunity_ids");
  const assignments = chosen("lms_assignment_ids");
  const syllabusItems = chosen("syllabus_item_ids");
  const events = [
    ...planner.markers
      .filter((m) => opportunities.has(m.opportunity_id))
      .map((m) => ({ uid: `opportunity-${m.opportunity_id}@unipivot`, title: `[마감] ${m.title.replace(/ 마감$/, "")}`, date: m.date })),
    ...planner.items
      .filter((t) => t.link.id !== null && assignments.has(t.link.id))
      .map((t) => ({ uid: `lms_assignment-${t.link.id}@unipivot`, title: `[과제] ${t.link.title} · ${t.title}`, date: t.due_date })),
    ...planner.items
      .filter((t) => t.link.id !== null && syllabusItems.has(t.link.id))
      .map((t) => ({ uid: `syllabus_item-${t.link.id}@unipivot`, title: `[일정] ${t.link.title} · ${t.title}`, date: t.due_date })),
  ];
  const stamp = (date: string, hour: string) => `${date.replace(/-/g, "")}T${hour}0000Z`; // 18:00 KST = 09:00Z
  const lines = ["BEGIN:VCALENDAR", "VERSION:2.0", "PRODID:-//UNIPIVOT//UNIPIVOT Calendar//KO", "CALSCALE:GREGORIAN", "METHOD:PUBLISH"];
  for (const event of events) {
    lines.push("BEGIN:VEVENT", `UID:${event.uid}`, `DTSTAMP:${stamp(today, "00")}`, `DTSTART:${stamp(event.date, "08")}`, `DTEND:${stamp(event.date, "09")}`, `SUMMARY:${event.title}`, "END:VEVENT");
  }
  lines.push("END:VCALENDAR");
  return lines.join("\r\n") + "\r\n";
}
```

## 패치

고친 파일 4개다. 그대로 `ics.patch`로 저장해서 `git apply ics.patch`.

```diff
diff --git a/api/app/routers/calendar.py b/api/app/routers/calendar.py
index b90f340..b3a261c 100644
--- a/api/app/routers/calendar.py
+++ b/api/app/routers/calendar.py
@@ -1,10 +1,12 @@
-"""마감일 캘린더 등록 (F-32·F-51, API 명세 7장).
+"""마감일 캘린더 등록 (F-32·F-51, API 명세 7장)과 .ics 내려받기 (F-34).
 
 대상은 공고 마감이나 LMS 과제 마감 하나다. 일정은 마감 시각으로 끝나는 1시간짜리이고, 일정 번호는
-우리 행 id에서 만들어 보내므로 다시 눌러도 캘린더에 하나만 들어간다.
+우리 행 id에서 만들어 보내므로 다시 눌러도 캘린더에 하나만 들어간다. .ics 내려받기는 같은 대상을
+여럿 받아 파일 하나로 주고, 기록은 남기지 않는다.
 """
 
 import logging
+from collections.abc import Awaitable, Callable
 from dataclasses import dataclass
 from datetime import datetime, timedelta
 from uuid import UUID
@@ -13,6 +15,7 @@ from fastapi import APIRouter, Response
 
 from app.auth import CurrentUserDep
 from app.calendar import EVENT_MINUTES, assignment_title, event_id_for, event_title
+from app.calendar.ics import IcsEvent, event_uid, render_calendar
 from app.calendar.service import CalendarServiceDep
 from app.clock import NowDep
 from app.eligibility import KST
@@ -25,6 +28,8 @@ logger = logging.getLogger(__name__)
 router = APIRouter(prefix="/calendar", tags=["캘린더"])
 
 NOT_FOUND = "캘린더 일정을 찾을 수 없어요."
+NOTHING_TO_DOWNLOAD = "내려받을 일정이 없어요."
+ICS_LIMIT = 200  # 한 번에 받는 대상 수 상한. 주소 길이와 조회 횟수를 묶어 둔다
 
 
 @router.post("/events", status_code=201, response_model=CalendarEventView)
@@ -41,7 +46,7 @@ async def add_event(
     일정은 마감 시각으로 끝나는 1시간짜리다. 연결이 없거나 만료면 409 CALENDAR_NOT_CONNECTED,
     Google이 거절하면 502 GOOGLE_API_ERROR다.
     """
-    if body.provider != "google":  # .ics 내려받기는 F-34(P1)다
+    if body.provider != "google":  # .ics는 등록이 아니라 내려받기다(GET /calendar/ics)
         raise ApiError(
             422,
             "VALIDATION_FAILED",
@@ -188,3 +193,79 @@ def _invalid(reason: str, *, field: str = "opportunity_id") -> ApiError:
     return ApiError(
         422, "VALIDATION_FAILED", "입력값을 확인해 주세요.", {"fields": {field: reason}}
     )
+
+
+@router.get("/ics")
+async def download_ics(
+    user: CurrentUserDep,
+    repo: CalendarRepoDep,
+    now: NowDep,
+    opportunity_ids: str = "",
+    lms_assignment_ids: str = "",
+    syllabus_item_ids: str = "",
+) -> Response:
+    """고른 마감·일정을 .ics 파일 하나로 준다(F-34). Google 캘린더를 쓰지 않는 학생용이다.
+
+    대상과 일정 모양은 `POST /calendar/events`와 같다. 넣을 수 없는 대상(준비하지 않은 공고,
+    마감·날짜가 없는 것, 이미 지난 것, 남의 것·없는 것)은 건너뛰고, 하나도 남지 않으면 404다.
+    `calendar_events`에는 기록하지 않는다(PRD F-34).
+    """
+    wanted = _ics_targets(opportunity_ids, lms_assignment_ids, syllabus_item_ids)
+    events: list[IcsEvent] = []
+    for kind, resolve, target_id in wanted:
+        try:
+            target = await resolve(repo, user.id, target_id, now)
+        except ApiError as exc:
+            if exc.status not in (404, 409, 422):
+                raise
+            continue  # 넣을 수 없는 대상은 건너뛴다
+        events.append(
+            IcsEvent(
+                uid=event_uid(kind, target_id),
+                title=target.title,
+                starts_at=target.ends_at - timedelta(minutes=EVENT_MINUTES),
+                ends_at=target.ends_at,
+            )
+        )
+    if not events:
+        raise ApiError(404, "NOT_FOUND", NOTHING_TO_DOWNLOAD)
+    filename = f"unipivot-{now.astimezone(KST):%Y%m%d}.ics"
+    return Response(
+        render_calendar(events, now=now),
+        media_type="text/calendar",
+        headers={"Content-Disposition": f'attachment; filename="{filename}"'},
+    )
+
+
+_Resolve = Callable[[CalendarRepository, str, str, datetime], Awaitable[_Target]]
+_ICS_KINDS: list[tuple[str, str, _Resolve]] = [
+    ("opportunity_ids", "opportunity", _opportunity_target),
+    ("lms_assignment_ids", "lms_assignment", _assignment_target),
+    ("syllabus_item_ids", "syllabus_item", _syllabus_target),
+]
+
+
+def _ics_targets(*given: str) -> list[tuple[str, _Resolve, str]]:
+    """쉼표로 나눈 uuid 목록 셋을 (종류, 조회 함수, id)로 편다. 같은 id는 한 번만 둔다.
+
+    uuid가 아닌 값이 있으면 그 파라미터 이름으로 422, 합쳐서 하나도 없거나 상한을 넘어도 422다.
+    """
+    targets: list[tuple[str, _Resolve, str]] = []
+    seen: set[str] = set()
+    for (param, kind, resolve), raw in zip(_ICS_KINDS, given, strict=True):
+        for value in raw.split(","):
+            value = value.strip()
+            if not value:
+                continue
+            try:
+                target_id = str(UUID(value))
+            except ValueError:
+                raise _invalid("uuid가 아닌 값이 있어요.", field=param) from None
+            if target_id not in seen:
+                seen.add(target_id)
+                targets.append((kind, resolve, target_id))
+    if not targets:
+        raise _invalid("공고·LMS 과제·계획서 일정 id를 하나 이상 주세요.", field="ids")
+    if len(targets) > ICS_LIMIT:
+        raise _invalid(f"한 번에 {ICS_LIMIT}개까지 받을 수 있어요.", field="ids")
+    return targets
diff --git a/web/src/api/client.ts b/web/src/api/client.ts
index a203d83..8332726 100644
--- a/web/src/api/client.ts
+++ b/web/src/api/client.ts
@@ -1,8 +1,11 @@
-import { ApiError, apiFetch, USE_MOCKS, waitForRun } from "@/api/http";
+import { ApiError, apiFetch, apiFetchText, USE_MOCKS, waitForRun } from "@/api/http";
 import * as mock from "@/api/mockClient";
 import { MockApiError, respond } from "@/api/mockClient";
 import { getKstDateKey } from "@/lib/format";
+import { daysLater, ICS_DAYS, icsQuery } from "@/lib/ics";
+import { saveTextFile } from "@/lib/saveFile";
 import { mockNow, mockToday } from "@/mocks";
+import { mockIcs } from "@/mocks/ics";
 import {
   getStoredPlanner,
   addStoredManualTask,
@@ -120,6 +123,19 @@ export function getPlanner(from: string, to: string): Promise<PlannerResponse> {
   return apiFetch<PlannerResponse>(`/planner/tasks?${query}`);
 }
 
+// GET /calendar/ics (F-34). 오늘부터 1년 안의 준비한 공고 마감·LMS 과제·계획서 일정을 .ics 파일 하나로
+// 내려받는다. Google 캘린더를 쓰지 않는 학생이 iPhone·삼성·Outlook 캘린더에 넣는 파일이다.
+// 서버에 기록은 남지 않는다. 받을 일정이 없으면 파일을 만들지 않고 false다
+export async function downloadPlannerIcs(): Promise<boolean> {
+  const today = todayKst();
+  const planner = await getPlanner(today, daysLater(today, ICS_DAYS));
+  const query = icsQuery(planner, today);
+  if (query === null) return false;
+  const text = USE_MOCKS ? await respond(mockIcs(planner, today)) : await apiFetchText(`/calendar/ics?${query}`);
+  saveTextFile(`unipivot-${today.replace(/-/g, "")}.ics`, text, "text/calendar");
+  return true;
+}
+
 // PATCH /planner/tasks/{id}. 완료 체크는 어느 출처든 할 수 있다
 export function setPlannerTaskDone(id: string, isDone: boolean): Promise<PlannerTask> {
   if (USE_MOCKS) return respond(setStoredPlannerTaskDone(id, isDone));
diff --git a/web/src/api/http.ts b/web/src/api/http.ts
index 945723e..f4d231c 100644
--- a/web/src/api/http.ts
+++ b/web/src/api/http.ts
@@ -33,7 +33,8 @@ function networkMessage(): string {
     : "서버에 연결하지 못했어요. 잠시 후 다시 시도해 주세요.";
 }
 
-export async function apiFetch<T>(path: string, init: RequestInit = {}): Promise<T> {
+// 응답을 받기까지. 로그인 전·세션 만료(401)면 로그인 화면으로 보내고 끝나지 않는 Promise를 둔다
+async function apiResponse(path: string, init: RequestInit = {}): Promise<Response> {
   const base = process.env.NEXT_PUBLIC_API_BASE_URL;
   if (!base) {
     throw new ApiError("CONFIG_MISSING", "API 주소(NEXT_PUBLIC_API_BASE_URL)가 설정되지 않았어요.", 0);
@@ -59,7 +60,7 @@ export async function apiFetch<T>(path: string, init: RequestInit = {}): Promise
   // 페이지가 바뀌므로 화면에 오류를 넘기지 않고 기다리게 둔다
   if (response.status === 401 && typeof window !== "undefined") {
     window.location.assign(token ? "/login?error=session" : "/login");
-    return new Promise<T>(() => {});
+    return new Promise<Response>(() => {});
   }
   if (!response.ok) {
     const body = await response.json().catch(() => null);
@@ -70,9 +71,19 @@ export async function apiFetch<T>(path: string, init: RequestInit = {}): Promise
       body?.error?.details,
     );
   }
+  return response;
+}
+
+export async function apiFetch<T>(path: string, init: RequestInit = {}): Promise<T> {
+  const response = await apiResponse(path, init);
   return (response.status === 204 ? undefined : await response.json()) as T;
 }
 
+// JSON이 아닌 응답(.ics 파일 같은 글)을 그대로 받는다
+export async function apiFetchText(path: string): Promise<string> {
+  return (await apiResponse(path)).text();
+}
+
 // 화면에 띄울 오류 문구. 서버(13장)와 프론트 코드가 준 문구는 그대로 쓰고,
 // 서버 내부 오류(5xx)·알 수 없는 오류·목 모드 오류는 화면이 준 문구를 쓴다
 export function errorMessage(error: unknown, fallback: string): string {
diff --git a/web/src/screens/Planner.tsx b/web/src/screens/Planner.tsx
index 47225da..f818822 100644
--- a/web/src/screens/Planner.tsx
+++ b/web/src/screens/Planner.tsx
@@ -2,6 +2,7 @@ import { useEffect, useState } from "react";
 import {
   addManualPlannerTask,
   deleteManualPlannerTask,
+  downloadPlannerIcs,
   errorMessage,
   getMe,
   getPlanner,
@@ -52,6 +53,7 @@ export default function Planner({ navigate }: { navigate: (route: Route) => void
   const [selectedDate, setSelectedDate] = useState(todayKst);
   const [plannerOverride, setPlannerOverride] =
     useState<PlannerResponse | null>(null);
+  const [downloading, setDownloading] = useState(false);
   const [toast, showToast] = useToast();
   const range = monthRange(view.year, view.month);
   const request = useAsyncData(
@@ -104,6 +106,23 @@ export default function Planner({ navigate }: { navigate: (route: Route) => void
       "할 일을 체크하지 못했어요. 잠시 후 다시 눌러 주세요.",
     ).catch(() => {});
 
+  // Google 캘린더를 쓰지 않는 학생용(F-34). 오늘부터 1년 안의 마감·일정을 .ics 파일 하나로 받는다
+  const downloadIcs = async () => {
+    setDownloading(true);
+    try {
+      const saved = await downloadPlannerIcs();
+      showToast(
+        saved
+          ? ".ics 파일을 내려받았어요. 캘린더 앱에서 열면 일정이 들어가요."
+          : "내려받을 일정이 없어요. 준비한 공고나 LMS 과제가 생기면 받을 수 있어요.",
+      );
+    } catch (error) {
+      showToast(errorMessage(error, "파일을 내려받지 못했어요. 잠시 후 다시 눌러 주세요."));
+    } finally {
+      setDownloading(false);
+    }
+  };
+
   return (
     <>
       <Header
@@ -168,6 +187,18 @@ export default function Planner({ navigate }: { navigate: (route: Route) => void
           </Button>
         )}
       </section>
+      {/* Google 캘린더를 쓰지 않는 학생은 파일로 받아 기본 캘린더 앱에 넣는다(F-34) */}
+      <section className="flex items-center justify-between gap-4 rounded-[14px] border border-[#e5e7eb] bg-white p-[18px]">
+        <div>
+          <p className="text-[14px] font-semibold">.ics 파일로 내려받기</p>
+          <p className="mt-1 text-[12px] text-[#667085]">
+            Google 캘린더를 쓰지 않으면 다가오는 마감일과 수업 일정을 파일로 받아 iPhone·삼성·Outlook 캘린더에 넣을 수 있어요.
+          </p>
+        </div>
+        <Button secondary onClick={downloadIcs} disabled={downloading}>
+          {downloading ? "파일 만드는 중…" : ".ics 내려받기"}
+        </Button>
+      </section>
       <Toast toast={toast} />
     </>
   );
```
