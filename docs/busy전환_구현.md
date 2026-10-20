# 진행 중 버튼 busy 전환 구현 (web, `fix/web-busy-buttons`)

> 📝 기준: web/CLAUDE.md "하지 않는 것"(busy 규칙) · docs/더보기초점_구현.md "남은 것" 둘째 줄
> 적용할 곳: #75까지 합친 `main`(742f066) 위에 **더 보기 마지막 쪽 초점 PR(docs/더보기마지막초점_구현.md)을 합친 뒤**. 둘 다 `screens/Notifications.tsx`·`web/CLAUDE.md`를 고쳐서, 그 PR을 얹은 트리에서 패치를 만들었다.

누른 일이 진행 중일 때 `disabled`로 막던 버튼·스위치·체크박스를 `busy`(또는 `aria-disabled`)로 바꿔, 키보드로 누른 뒤 초점이 `<body>`로 빠지지 않게 한다. 처음에는 같은 자리에 남아 다시 누르는 곳만 바꾸려 했지만, 독립 검토가 "성공하면 화면이 바뀌는 버튼도 실패하면 그 자리에 남는다"고 짚어 **진행 중에 `disabled`로 막던 곳을 모두** 바꿨다. 남은 `disabled`는 "지금 할 수 없는 일"(입력이 덜 됨, 원문 주소 없음 등)뿐이다. 자리를 비운 동안 고른 것은 1장에 적었다.

| 값 | 결과 |
| --- | --- |
| 브랜치 | `fix/web-busy-buttons` |
| 파일 | 고친 파일 24개(전부 web), 새 파일 없음 |
| api | 안 바뀜(CI 1598 passed 그대로) |
| web | `npm run typecheck`·`npm run build`(실제·목 모드) 통과 |
| 브라우저 | 실제 모드(.ics, 알림함 모두 읽음, 앞 PR의 더 보기 점검 전부)·목 모드(플래너 체크박스, 소득 동의 스위치, 헤더 알림 패널 모두 읽음·안내) — 5장 |
| 되돌림 시험 | 7개 중 6개를 브라우저 점검이 잡음(못 잡은 1개는 6장) |
| 마이그레이션 | 없음 |

## 1. 정한 것

| 곳 | 바꾼 것 | 판단 |
| --- | --- | --- |
| 플래너 `.ics 내려받기`(1순위) | `disabled={downloading}` → `busy` | 같은 버튼이 "파일 만드는 중…"으로 바뀌었다가 돌아온다. 다시 받으려면 그대로 Enter |
| `DeadlineCalendarRow` 캘린더에 넣기·빼기 | `busy` | 같은 버튼의 글이 넣기↔빼기로 바뀐다 |
| `MaterialSummaryPanel` 번역하기 | 고른 구간이 없으면 `disabled`, 번역 중이면 `busy` | 둘을 같이 주지 않도록 `disabled={!translating && selected.length === 0}` |
| `PlannerDayPanel` 할 일 체크박스 | `disabled` → `aria-disabled`·`aria-busy` + `onChange`에서 저장 중이면 무시. 잠금은 **할 일마다**(저장 중인 id의 `Set`) | 체크박스는 `aria-disabled`가 토글을 막지 못한다. 제어 체크박스라 무시하면 `checked`가 그대로다. 전에는 화면에 잠금이 하나라 A 저장 중에 B를 누르면 A의 잠금이 풀리고, A가 끝나면 B의 잠금이 풀렸다(검토) |
| `PlannerDayPanel` 일정 저장 | `busy` | 실패하면 창이 남아 다시 누른다 |
| `PreparationProgress` 완료 | `busy` | |
| `IncomeConsentSetting` 스위치·동의 철회 | 스위치는 `aria-disabled`·`aria-busy` + 누름 무시, 철회 버튼은 `busy` | 스위치는 직접 만든 `role="switch"` 버튼이다. 모양은 `aria-disabled:cursor-wait aria-disabled:opacity-45`(Button과 같다) |
| 공용 `Toggle` | `busy` 더함(`disabled`는 그대로) | `BrowserPushSetting`이 바꾸는 중이면 `busy`, 브라우저가 막았으면(`denied`) 진짜 `disabled` |
| `SyllabusConfirmedList` 캘린더에 넣기 | `aria-disabled`(넣는 중엔 모든 줄) + 누름 무시, 누른 줄은 `aria-busy`. 기다리는 줄은 기본 커서, 넣는 줄만 대기 커서 | 다른 줄의 버튼으로 Tab해 가도 초점이 남는다. 성공하면 그 줄은 "캘린더에 넣었어요" 글로 바뀌어 초점이 빠진다(8장) |
| 알림함·헤더 패널 "모두 읽음" | `disabled`(읽을 것 없음) → `aria-disabled` + 누름 무시 | 누르는 순간 안 읽은 알림이 0이 되어 disabled가 되고 초점이 빠졌다. "할 수 없는 일"이지만 누른 버튼이 바로 그렇게 되는 경우라 같은 규칙으로 묶었다(web/CLAUDE.md에 한 줄) |
| 준비 일정 만들기·그래도 준비하기(`OpportunityPreparation`), 알림 켜기(`PushEnableBanner`), 캘린더 연결하기(`ConnectedServices`), 공고 등록(`PosterExtractForm`), 계획서 확정(`SyllabusReviewPanel`), 탈퇴, 온보딩 세 단계, 소득 동의하고 입력하기(`ProfileInputModal`) | `busy`. 조건이 함께 있는 곳은 `disabled={!진행중 && 조건} busy={진행중}`(둘을 같이 주지 않는다) | 성공하면 화면·창이 바뀌지만, 실패하면 그 자리에 남아 다시 누른다(검토) |
| 프로필 저장 둘(`ProfileForm`·`ProfileInputModal`, `type="submit"`) | `busy` + 폼 `onSubmit` 첫 줄에서 저장 중이면 돌아간다 | `busy`는 누름만 막는다. 입력란에서 Enter(암묵적 제출)는 `onSubmit`으로 바로 온다. 전에는 `disabled` 기본 버튼이라 브라우저가 암묵적 제출을 막아 줬다 |
| 로그인 "Google로 시작하기"(직접 만든 버튼) | `aria-disabled`·`aria-busy` + 누름 무시 | Google로 가지 못하면 이 화면에 남는다 |
| 헤더 알림 패널 "모두 읽음" 결과 | 늘 두는 `sr-only` `role="status"`에 "알림을 모두 읽음으로 바꿨어요."/실패 문구 | 초점이 버튼에 남으니 결과를 읽어 줄 자리가 필요하다. 알림함 화면은 원래 토스트(`role="status"`)로 알린다(검토) |
| 그대로 둔 것 | `OpportunityDocuments` 서류 체크(준비 전), `PosterExtractForm` 입력란(수정 전), 원문 보기(주소 없음), `LmsConnectModal` 다음(토큰 없음), `SyllabusReviewPanel`·`SyllabusReviewTable`의 입력(저장 중 다른 입력을 막는 것) | 지금 할 수 없는 일이다. 누른 버튼이 아니라 초점과 상관없다 |

## 2. 레포에 넣는 법

````text
[0] 더 보기 마지막 쪽 초점 PR(docs/더보기마지막초점_구현.md)이 main에 들어가 있어야 한다(screens/Notifications.tsx와 web/CLAUDE.md를 둘 다 고친다).
    git checkout main && git pull
    git checkout -b fix/web-busy-buttons

[1] 새 파일은 없다.

[2] "패치" 절의 내용을 그대로 busy-buttons.patch로 저장해서 적용해 줘(고친 파일 24개, 전부 web).
    git apply busy-buttons.patch
    (적용한 뒤 busy-buttons.patch는 지워 줘)

[3] 명세·PRD·ERD는 바뀌지 않는다("문서에 반영하기" 없음. web/CLAUDE.md 규칙 한 줄은 패치에 들어 있다).

[4] 이 문서도 docs/busy전환_구현.md로 같이 넣어 줘.

[5] 검사: web> npm run typecheck && npm run build
    api는 바뀌지 않는다(CI에서 1598 passed 그대로).

[6] 통과하면 "fix(web): 진행 중인 버튼을 disabled 대신 busy로 막아 초점을 지킨다"로 커밋해서 PR을 올려 줘.
````

## 3. 합친 뒤 확인

키보드로만 본다.

````text
1. 플래너 맨 아래 ".ics 내려받기"에 Tab으로 초점 → Enter. "파일 만드는 중…"인 동안과 끝난 뒤에도 초점 테두리가 그 버튼에 남아야 한다.
2. 플래너의 할 일 체크박스에 초점 → Space. 저장한 뒤에도 초점이 그 체크박스에 남고, Space를 빨리 두 번 눌러도 한 번만 바뀐다.
3. 알림함(또는 헤더 알림 패널)에서 "모두 읽음"에 초점 → Enter. 회색으로 바뀌어도 초점이 남고, 다시 Enter를 눌러도 아무 일도 없어야 한다.
4. 설정 > 계정의 소득·수급 동의 스위치에 초점 → Enter(켜기). 바꾸는 동안과 뒤에도 초점이 스위치에 남는다.
````

## 4. 구조

```text
Button (#74)  : busy → aria-disabled·aria-busy, 누름 취소, 초점 유지
Toggle (이번) : busy → aria-disabled·aria-busy, 누름 무시. disabled는 "켤 수 없음"(브라우저가 막음)
직접 만든 것  : <input type=checkbox aria-disabled> + onChange 무시 / role=switch 버튼 + onClick 무시 / 글 버튼 + onClick 무시
```

## 5. 확인한 것

| 확인 | 결과 |
| --- | --- |
| `npm run typecheck` | 통과 |
| `npm run build` (실제·목 모드) | 통과 |
| 플래너 `.ics 내려받기`(실제, 할 일 목록 응답을 1.5초 늦춤) | 누른 뒤 0.5초: 초점 그 버튼, 글 "파일 만드는 중…", `aria-busy=true`. 끝난 뒤: 초점 그 버튼 |
| 알림함 모두 읽음(실제, 안 읽은 알림 60개) | 누른 뒤 초점 그 버튼, `aria-disabled=true`, 읽음 요청 1번. 다시 Enter → 요청 그대로 1번, 초점 그대로 |
| 플래너 체크박스(목) | Space 직후 초점 그 체크박스·`aria-disabled=true`, 저장 중 두 번째 Space는 무시(한 번만 바뀜), 끝난 뒤 초점 그대로 |
| 소득 동의 스위치(목, 철회해 끈 뒤 켜기) | 누른 직후 초점 스위치·`aria-busy=true`, 끝난 뒤 초점 그대로·켜짐 |
| 헤더 알림 패널 모두 읽음(목) | 누른 뒤 초점 그 버튼, `aria-disabled=true`, 상태 줄 "알림을 모두 읽음으로 바꿨어요." |
| 앞 PR(더 보기 마지막 쪽 초점) 점검 | 이 트리에서 다시 돌려 전부 같음 |
| 페이지 오류 | 0 |
| 브라우저로 못 본 것 | 번역하기, 캘린더에 넣기·빼기, 완료, 일정 저장, 계획서 캘린더에 넣기, `BrowserPushSetting`(샌드박스에 푸시 설정이 없다), 1장 둘째 묶음(준비하기·연결·등록·확정·탈퇴·온보딩·로그인), 프로필 저장의 Enter 막기. 같은 `busy`/`aria-disabled` 방식이라 typecheck·build로만 봤다 |

## 6. 되돌림 시험

실제·목 모드로 다시 빌드해 브라우저 점검으로 잡았다.

| # | 고장 낸 곳 | 결과 |
| --- | --- | --- |
| B1 | `.ics 내려받기`를 다시 `disabled`로 | 잡음 — 만드는 동안·끝난 뒤 초점이 `<body>` |
| B2 | 알림함 "모두 읽음"을 다시 `disabled`로 | 잡음 — 누른 뒤 초점이 빠짐 |
| B3 | 체크박스 저장 중 누름 무시 빼기 | **못 잡음** — 두 번째 Space도 같은 값으로 한 번 더 저장해 화면 결과가 같다. 요청 수로만 보이는데 목 모드라 셀 수 없다. 코드로만 지킨다 |
| B4 | 체크박스를 다시 `disabled`로 | 잡음 — 저장 중 초점이 빠짐 |
| B5 | 소득 동의 스위치를 다시 `disabled`로 | 잡음 — 바꾸는 동안 초점이 빠짐 |
| B6 | 헤더 패널 "모두 읽음"을 다시 `disabled`로 | 잡음 — 누른 뒤 초점이 빠짐 |
| B7 | 패널 "모두 읽음" 결과 안내 빼기 | 잡음 — 상태 줄이 비어 있음 |

## 7. 검토에서 고친 것

새 에이전트(이 작업을 못 본)가 읽기만 해서 본 결과다. 높음은 없었고, 바꾼 파일 안에서 누름 무시가 빠진 곳은 없었다.

| 지적 | 처리 |
| --- | --- |
| 중간 — 체크박스 잠금이 화면에 하나라 다른 할 일끼리 서로 잠금을 푼다 | 저장 중인 id의 `Set`으로 바꾸고 끝날 때 자기 id만 뺀다 |
| 중간 — 헤더 패널 "모두 읽음"은 결과를 화면 읽기 프로그램에 알리지 않는다 | 늘 두는 `sr-only` 상태 줄에 결과를 넣는다. 되돌림 시험 B7 |
| 중간 — 진행 중에 `disabled`로 막는 버튼이 더 남아 새 규칙과 어긋난다(실패하면 그 자리에 남는다) | 1장 둘째 묶음을 모두 바꿨다. submit 둘은 `onSubmit`도 막았다 |
| 낮음 — 체크박스에 저장 중 모양·`aria-busy`가 없다 | `aria-busy`와 `aria-disabled:cursor-wait aria-disabled:opacity-45`를 더했다 |
| 낮음 — 계획서 캘린더: 기다리는 줄도 대기 커서 | 기다리는 줄은 기본 커서, 넣는 줄만 `aria-busy:cursor-wait` |
| 낮음 — `/notifications`에서 헤더 패널과 화면이 따로 목록을 들고 있어 한쪽 "모두 읽음"이 다른 쪽에 안 보인다 | 그대로 둔다(원래 있던 구조, 8장) |
| 낮음 — 패널에서 목록을 받기 전·받기 실패 땐 "모두 읽음"이 흐린데 옆 숫자는 N개 | 그대로 둔다(원래 동작). 누름 무시는 맞다 |
| 낮음 — 번역 중에 구간을 모두 해제하면 끝날 때 `disabled`가 되어 초점이 빠진다 | 그대로 둔다(드물다) |
| 낮음 — 소득 동의 철회 창이 성공·실패와 상관없이 닫혀 초점이 빠진다, 계획서 캘린더 성공 뒤 초점, 플래너 "삭제" 연타 | 그대로 둔다(원래 동작, 8장) |
| 낮음 — 켜는 중 권한을 거부하면 `disabled`가 되어 초점이 빠진다 | 그대로 둔다. 정말 할 수 없게 된 상태다 |

## 8. 남은 것

- `Modal`에 초점 가두기·닫을 때 원래 버튼으로 돌아가기가 없다(소득 동의 철회 창 등). 다른 창과 같이 고치는 작은 PR이 맞다.
- `/notifications` 화면에서는 헤더 패널과 화면이 `useNotifications`를 따로 들고 있어 한쪽의 "모두 읽음"이 다른 쪽 목록에 바로 안 보인다.
- 플래너 할 일 메뉴의 "삭제"는 연타를 막지 않는다.
- 계획서 "캘린더에 넣기"는 성공하면 버튼이 글로 바뀌어 초점이 빠진다. 바뀐 글(`tabIndex={-1}`)로 옮기는 작은 고침이 남았다.

## 9. 문서에 반영하기

````text
없다. 명세·PRD·ERD는 바뀌지 않는다(web/CLAUDE.md 규칙 한 줄은 패치에 들어 있다).
````

## 10. 코드와 패치

### 코드 (새 파일)

새 파일은 없다.

### 패치

```diff
diff --git a/web/CLAUDE.md b/web/CLAUDE.md
index cd60d64..3e93a55 100644
--- a/web/CLAUDE.md
+++ b/web/CLAUDE.md
@@ -70,4 +70,4 @@ Figma Make 시안을 옮긴 코드다. 아래 화면·데이터 규칙을 지키
 - 라우터·상태관리 라이브러리 추가(아이콘은 lucide-react만), 채팅 UI
 - 실명이나 토큰 원문 표시
 - 화면 컴포넌트 안에서 `fetch` 직접 호출(데이터는 `client.ts`로만)
-- 진행 중인 버튼을 `disabled`로 막기(키보드 초점이 빠진다). `Button`의 `busy`(aria-disabled·aria-busy, 초점 유지)를 쓰고, `disabled`는 지금 할 수 없는 일(입력이 덜 됨)에만 쓴다. 둘을 같이 주지 않는다
+- 진행 중인 버튼을 `disabled`로 막기(키보드 초점이 빠진다). `Button`·`Toggle`의 `busy`(aria-disabled·aria-busy, 초점 유지)를 쓰고, `disabled`는 지금 할 수 없는 일(입력이 덜 됨)에만 쓴다. 둘을 같이 주지 않는다. 직접 만든 버튼·체크박스·스위치도 `aria-disabled`로 알리고 누름(체크박스는 `onChange`)을 무시한다. 누르면 할 일이 없어지는 버튼("모두 읽음")도 같다
diff --git a/web/src/app/(auth)/login/page.tsx b/web/src/app/(auth)/login/page.tsx
index 12cebc0..0a21afd 100644
--- a/web/src/app/(auth)/login/page.tsx
+++ b/web/src/app/(auth)/login/page.tsx
@@ -65,9 +65,13 @@ function LoginCard() {
       </h1>
       <button
         type="button"
-        onClick={signIn}
-        disabled={pending}
-        className="mt-8 flex h-[48px] w-full items-center justify-center gap-3 rounded-[10px] border border-[#e5e7eb] bg-white text-[15px] font-semibold text-[#181a20] transition hover:bg-[#f7f8fa] disabled:cursor-not-allowed disabled:opacity-50"
+        // Google로 가는 동안 disabled로 막으면 초점이 빠진다(가지 못하면 이 화면에 남는다). 누름만 무시한다
+        onClick={() => {
+          if (!pending) void signIn();
+        }}
+        aria-disabled={pending || undefined}
+        aria-busy={pending || undefined}
+        className="mt-8 flex h-[48px] w-full items-center justify-center gap-3 rounded-[10px] border border-[#e5e7eb] bg-white text-[15px] font-semibold text-[#181a20] transition hover:bg-[#f7f8fa] aria-disabled:cursor-wait aria-disabled:opacity-50"
       >
         <GoogleMark />
         {pending ? "Google로 이동 중" : "Google로 시작하기"}
diff --git a/web/src/components/BrowserPushSetting.tsx b/web/src/components/BrowserPushSetting.tsx
index d282f97..f898e18 100644
--- a/web/src/components/BrowserPushSetting.tsx
+++ b/web/src/components/BrowserPushSetting.tsx
@@ -23,7 +23,8 @@ export default function BrowserPushSetting({ pushEnabled }: { pushEnabled: boole
         </div>
         <Toggle
           checked={state.subscribed}
-          disabled={busy || denied}
+          disabled={denied}
+          busy={busy && !denied}
           label="이 브라우저에서 알림 받기"
           onChange={(checked) => void (checked ? enable() : disable())}
         />
diff --git a/web/src/components/ConnectedServices.tsx b/web/src/components/ConnectedServices.tsx
index 612353e..e2dbae0 100644
--- a/web/src/components/ConnectedServices.tsx
+++ b/web/src/components/ConnectedServices.tsx
@@ -86,7 +86,7 @@ export default function ConnectedServices({ me, lms: initialLms, calendarResult,
           {me.calendar_connected ? (
             <Button secondary onClick={() => setCalendarOpen(true)}>연결 해제</Button>
           ) : (
-            <Button secondary disabled={connecting} onClick={connectCalendar}>
+            <Button secondary busy={connecting} onClick={connectCalendar}>
               {connecting ? "Google로 이동 중" : "연결하기"}
             </Button>
           )}
diff --git a/web/src/components/DeadlineCalendarRow.tsx b/web/src/components/DeadlineCalendarRow.tsx
index ff2f738..45af63b 100644
--- a/web/src/components/DeadlineCalendarRow.tsx
+++ b/web/src/components/DeadlineCalendarRow.tsx
@@ -29,7 +29,7 @@ export default function DeadlineCalendarRow({ registered, busy, onToggle }: Dead
             : "마감일을 Google Calendar에 넣어 두면 캘린더 알림으로도 받아요."}
         </p>
       </div>
-      <Button secondary onClick={onToggle} disabled={busy}>
+      <Button secondary onClick={onToggle} busy={busy}>
         {registered ? "캘린더에서 빼기" : "캘린더에 넣기"}
       </Button>
     </section>
diff --git a/web/src/components/IncomeConsentSetting.tsx b/web/src/components/IncomeConsentSetting.tsx
index 8e1690f..fef75b6 100644
--- a/web/src/components/IncomeConsentSetting.tsx
+++ b/web/src/components/IncomeConsentSetting.tsx
@@ -41,9 +41,15 @@ export default function IncomeConsentSetting({ consented, onChanged }: IncomeCon
           role="switch"
           aria-checked={consented}
           aria-label="소득·수급 정보 수집·이용 동의"
-          disabled={saving}
-          onClick={() => (consented ? setConfirmOpen(true) : change(true))}
-          className={`relative h-6 w-11 shrink-0 rounded-full ${consented ? "bg-[#4f6ef7]" : "bg-[#e5e7eb]"}`}
+          // 바꾸는 동안 disabled로 막으면 초점이 빠진다. aria-disabled·aria-busy로 알리고 누름은 무시한다
+          aria-disabled={saving || undefined}
+          aria-busy={saving || undefined}
+          onClick={() => {
+            if (saving) return;
+            if (consented) setConfirmOpen(true);
+            else void change(true);
+          }}
+          className={`relative h-6 w-11 shrink-0 rounded-full aria-disabled:cursor-wait aria-disabled:opacity-45 ${consented ? "bg-[#4f6ef7]" : "bg-[#e5e7eb]"}`}
         >
           <span className={`absolute top-1 size-4 rounded-full bg-white transition ${consented ? "left-6" : "left-1"}`} />
         </button>
@@ -55,7 +61,7 @@ export default function IncomeConsentSetting({ consented, onChanged }: IncomeCon
         </p>
         <div className="mt-6 flex justify-end gap-2">
           <Button secondary onClick={() => setConfirmOpen(false)}>취소</Button>
-          <Button disabled={saving} onClick={() => change(false)}>동의 철회</Button>
+          <Button busy={saving} onClick={() => change(false)}>동의 철회</Button>
         </div>
       </Modal>
     </>
diff --git a/web/src/components/MaterialSummaryPanel.tsx b/web/src/components/MaterialSummaryPanel.tsx
index 2edc1c4..adbaf77 100644
--- a/web/src/components/MaterialSummaryPanel.tsx
+++ b/web/src/components/MaterialSummaryPanel.tsx
@@ -130,7 +130,8 @@ export default function MaterialSummaryPanel({
       <div className="mt-4 flex items-center justify-between">
         {translating && <Badge>번역 중</Badge>}
         <div className="ml-auto">
-          <Button onClick={translate} disabled={selected.length === 0 || translating}>
+          {/* 고른 구간이 없으면 할 수 없는 일(disabled), 번역 중이면 진행 중(busy). 둘을 같이 주지 않는다 */}
+          <Button onClick={translate} disabled={!translating && selected.length === 0} busy={translating}>
             선택한 구간 번역하기
           </Button>
         </div>
diff --git a/web/src/components/NotificationPanel.tsx b/web/src/components/NotificationPanel.tsx
index 82235ce..5a1834e 100644
--- a/web/src/components/NotificationPanel.tsx
+++ b/web/src/components/NotificationPanel.tsx
@@ -20,18 +20,24 @@ export default function NotificationPanel({ open, onClose }: NotificationPanelPr
   const { unreadNotifications, setUnreadNotifications, navigate } = useAppShell();
   const { items, unread, loading, error, reload, read, readAll } = useNotifications({ enabled: open });
   const [readFailed, setReadFailed] = useState(false);
+  const [readNotice, setReadNotice] = useState(""); // "모두 읽음" 결과를 화면 읽기 프로그램에 알린다(알림함 화면은 토스트로 알린다)
 
   useEffect(() => {
     if (unread !== null) setUnreadNotifications(unread);
   }, [unread, setUnreadNotifications]);
 
   useEffect(() => {
-    if (open) setReadFailed(false);
+    if (open) {
+      setReadFailed(false);
+      setReadNotice("");
+    }
   }, [open]);
 
   // 패널은 알림을 누르면 바로 다른 화면으로 가서, 읽음 처리 실패는 "모두 읽음"에서만 알린다
   const readEverything = async () => {
-    setReadFailed(!(await readAll()));
+    const saved = await readAll();
+    setReadFailed(!saved);
+    setReadNotice(saved ? "알림을 모두 읽음으로 바꿨어요." : "읽음 처리를 저장하지 못했어요. 다시 눌러 주세요.");
   };
 
   const openNotification = (notification: AppNotification) => {
@@ -66,13 +72,17 @@ export default function NotificationPanel({ open, onClose }: NotificationPanelPr
             <p className="mt-1 text-[12px] text-[#667085]">
               {readFailed ? "읽음 처리를 저장하지 못했어요. 다시 눌러 주세요." : `읽지 않은 알림 ${unreadNotifications}개`}
             </p>
+            <p role="status" className="sr-only">{readNotice}</p>
           </div>
           <div className="flex items-center gap-3">
             <button
               type="button"
-              onClick={readEverything}
-              disabled={unread === null || unread === 0}
-              className="text-[12px] font-semibold text-[#4f6ef7] disabled:cursor-default disabled:text-[#a0a4ab]"
+              // 누르면 읽지 않은 알림이 0이 되어 disabled가 되고 초점이 빠졌다. 읽을 것이 없으면 aria-disabled로 알리고 누름은 무시한다
+              onClick={() => {
+                if (unread !== null && unread > 0) void readEverything();
+              }}
+              aria-disabled={unread === null || unread === 0 || undefined}
+              className="text-[12px] font-semibold text-[#4f6ef7] aria-disabled:cursor-default aria-disabled:text-[#a0a4ab]"
             >
               모두 읽음
             </button>
diff --git a/web/src/components/OpportunityPreparation.tsx b/web/src/components/OpportunityPreparation.tsx
index ac20c5e..a42db39 100644
--- a/web/src/components/OpportunityPreparation.tsx
+++ b/web/src/components/OpportunityPreparation.tsx
@@ -180,14 +180,14 @@ export default function OpportunityPreparation({
                 className="mt-1 block h-[42px] rounded-[10px] border border-[#e5e7eb] px-3 text-[14px] text-[#181a20]"
               />
             </label>
-            <Button onClick={() => startPreparation()} disabled={!targetDate || running}>
+            <Button onClick={() => startPreparation()} disabled={!running && !targetDate} busy={running}>
               준비 일정 만들기
             </Button>
           </div>
         );
       }
       return (
-        <Button onClick={() => startPreparation()} disabled={running}>
+        <Button onClick={() => startPreparation()} busy={running}>
           {running ? "만드는 중" : "준비 일정 만들기"}
         </Button>
       );
@@ -208,7 +208,7 @@ export default function OpportunityPreparation({
         <span title="대체 공고 추천은 준비 중이에요">
           <Button disabled>대체 공고 보기</Button>
         </span>
-        <Button secondary onClick={() => setConfirmOpen(true)} disabled={running}>
+        <Button secondary onClick={() => setConfirmOpen(true)} busy={running}>
           그래도 준비하기
         </Button>
       </div>
diff --git a/web/src/components/PlannerDayPanel.tsx b/web/src/components/PlannerDayPanel.tsx
index e758f69..ab48ebb 100644
--- a/web/src/components/PlannerDayPanel.tsx
+++ b/web/src/components/PlannerDayPanel.tsx
@@ -41,7 +41,7 @@ export default function PlannerDayPanel({
   const [dueDate, setDueDate] = useState(date);
   const [menuOpen, setMenuOpen] = useState<string | null>(null);
   const [saving, setSaving] = useState(false);
-  const [toggling, setToggling] = useState<string | null>(null);
+  const [toggling, setToggling] = useState<ReadonlySet<string>>(new Set()); // 저장 중인 할 일(할 일마다 따로 잠근다)
 
   const openForm = (task?: PlannerTask) => {
     setEditing(task ?? null);
@@ -88,16 +88,25 @@ export default function PlannerDayPanel({
                     <input
                       type="checkbox"
                       checked={task.is_done}
-                      disabled={toggling === task.id}
+                      // 저장하는 동안 disabled로 막으면 체크마다 초점이 빠진다. aria-disabled로 알리고 누름은 무시한다
+                      // (제어 체크박스라 무시하면 checked가 그대로 남는다)
+                      aria-disabled={toggling.has(task.id) || undefined}
+                      aria-busy={toggling.has(task.id) || undefined}
                       onChange={async () => {
-                        setToggling(task.id); // 연타하면 응답 순서가 엇갈린다
+                        if (toggling.has(task.id)) return;
+                        // 연타하면 응답 순서가 엇갈린다. 다른 할 일의 저장이 끝나도 이 할 일의 잠금은 풀리지 않게 자기 것만 뺀다
+                        setToggling((current) => new Set(current).add(task.id));
                         try {
                           await onToggle(task);
                         } finally {
-                          setToggling(null);
+                          setToggling((current) => {
+                            const next = new Set(current);
+                            next.delete(task.id);
+                            return next;
+                          });
                         }
                       }}
-                      className="mt-1 accent-[#4f6ef7]"
+                      className="mt-1 accent-[#4f6ef7] aria-disabled:cursor-wait aria-disabled:opacity-45"
                     />
                     <span>
                       <b className={`block text-[14px] ${task.is_done ? "text-[#a0a4ab] line-through" : ""}`}>{task.title}</b>
@@ -163,7 +172,7 @@ export default function PlannerDayPanel({
         </label>
         <div className="mt-6 flex justify-end gap-2">
           <Button secondary onClick={() => setModalOpen(false)}>취소</Button>
-          <Button onClick={save} disabled={saving}>{saving ? "저장 중" : "저장"}</Button>
+          <Button onClick={save} busy={saving}>{saving ? "저장 중" : "저장"}</Button>
         </div>
       </Modal>
     </>
diff --git a/web/src/components/PosterExtractForm.tsx b/web/src/components/PosterExtractForm.tsx
index 5f13865..42fafbd 100644
--- a/web/src/components/PosterExtractForm.tsx
+++ b/web/src/components/PosterExtractForm.tsx
@@ -203,7 +203,7 @@ export default function PosterExtractForm({
         <Button secondary onClick={() => setEditing((value) => !value)}>
           {editing ? "수정 완료" : "정보 수정"}
         </Button>
-        <Button onClick={onRegister} disabled={registering}>
+        <Button onClick={onRegister} busy={registering}>
           {registering ? "등록 중" : "공고 등록하고 자격 확인"}
         </Button>
       </div>
diff --git a/web/src/components/PreparationProgress.tsx b/web/src/components/PreparationProgress.tsx
index dac1184..af9d022 100644
--- a/web/src/components/PreparationProgress.tsx
+++ b/web/src/components/PreparationProgress.tsx
@@ -101,7 +101,7 @@ export default function PreparationProgress({
             <p className="mt-2 text-[12px] text-[#c53030]">{result.calendar_error}</p>
           )}
           <div className="mt-4 flex justify-end">
-            <Button onClick={onFinish} disabled={finishing}>
+            <Button onClick={onFinish} busy={finishing}>
               {finishing ? "완료 중" : "완료"}
             </Button>
           </div>
diff --git a/web/src/components/ProfileForm.tsx b/web/src/components/ProfileForm.tsx
index e28c9ef..4715613 100644
--- a/web/src/components/ProfileForm.tsx
+++ b/web/src/components/ProfileForm.tsx
@@ -55,6 +55,7 @@ export default function ProfileForm({
     <form
       onSubmit={(event) => {
         event.preventDefault();
+        if (saving) return; // 저장 버튼은 busy라 누름은 막지만 입력란의 Enter(암묵적 제출)는 여기로 온다
         const nextErrors = validateProfile(values, editableFields);
         setErrors(nextErrors);
         if (Object.keys(nextErrors).length > 0) return;
@@ -96,7 +97,7 @@ export default function ProfileForm({
       {message && <p role="alert" className="mt-4 text-[13px] text-[#c53030]">{message}</p>}
       <div className="mt-6 flex justify-end gap-2">
         {onCancel && <Button secondary onClick={onCancel}>취소</Button>}
-        <Button type="submit" disabled={saving}>
+        <Button type="submit" busy={saving}>
           {saving ? "저장 중" : submitLabel}
         </Button>
       </div>
diff --git a/web/src/components/ProfileInputModal.tsx b/web/src/components/ProfileInputModal.tsx
index 77f1535..114b16a 100644
--- a/web/src/components/ProfileInputModal.tsx
+++ b/web/src/components/ProfileInputModal.tsx
@@ -132,7 +132,8 @@ export default function ProfileInputModal({ open, opportunityId, missingFields,
 
   const submit = async (event: FormEvent) => {
     event.preventDefault();
-    if (!loaded || !values) return;
+    // 저장 버튼은 busy라 누름은 막지만 입력란의 Enter(암묵적 제출)는 여기로 온다
+    if (saving || !loaded || !values) return;
     const nextErrors = validateProfile(values, editable);
     setErrors(nextErrors);
     if (Object.keys(nextErrors).length > 0) return;
@@ -178,7 +179,7 @@ export default function ProfileInputModal({ open, opportunityId, missingFields,
                 <>
                   <p>소득·수급 정보는 선택 동의를 하면 입력할 수 있어요. 자격 판정에만 쓰고, 탈퇴하면 바로 지워요.</p>
                   <div className="mt-2">
-                    <Button secondary onClick={agreeIncome} disabled={agreeing}>
+                    <Button secondary onClick={agreeIncome} busy={agreeing}>
                       {agreeing ? "동의하는 중" : "동의하고 입력하기"}
                     </Button>
                   </div>
@@ -206,7 +207,7 @@ export default function ProfileInputModal({ open, opportunityId, missingFields,
           {error && <p role="alert" className="mt-4 text-[13px] text-[#c53030]">{error}</p>}
           <div className="mt-6 flex justify-end gap-2">
             <Button secondary onClick={onClose}>취소</Button>
-            <Button type="submit" disabled={saving || editable.length === 0}>
+            <Button type="submit" disabled={!saving && editable.length === 0} busy={saving}>
               {saving ? "다시 판정 중" : "저장하고 다시 판정"}
             </Button>
           </div>
diff --git a/web/src/components/PushEnableBanner.tsx b/web/src/components/PushEnableBanner.tsx
index 21d31f6..e5b3663 100644
--- a/web/src/components/PushEnableBanner.tsx
+++ b/web/src/components/PushEnableBanner.tsx
@@ -46,7 +46,7 @@ export default function PushEnableBanner({ onDone }: { onDone: (message: string)
       </div>
       <div className="flex shrink-0 items-center gap-2">
         <Button secondary onClick={hide}>나중에</Button>
-        <Button disabled={busy} onClick={() => void turnOn()}>알림 켜기</Button>
+        <Button busy={busy} onClick={() => void turnOn()}>알림 켜기</Button>
       </div>
     </section>
   );
diff --git a/web/src/components/SyllabusConfirmedList.tsx b/web/src/components/SyllabusConfirmedList.tsx
index 21583d0..99b88e0 100644
--- a/web/src/components/SyllabusConfirmedList.tsx
+++ b/web/src/components/SyllabusConfirmedList.tsx
@@ -64,9 +64,13 @@ export default function SyllabusConfirmedList({
                 (calendarConnected ? (
                   <button
                     type="button"
-                    onClick={() => onAddToCalendar(item)}
-                    disabled={busyItem !== null}
-                    className="text-[12px] font-semibold text-[#4f6ef7] disabled:text-[#a0a4ab]"
+                    // 넣는 동안 disabled로 막으면 초점이 빠진다. aria-disabled로 알리고 누름은 무시한다
+                    onClick={() => {
+                      if (busyItem === null) onAddToCalendar(item);
+                    }}
+                    aria-disabled={busyItem !== null || undefined}
+                    aria-busy={busyItem === item.id || undefined}
+                    className="text-[12px] font-semibold text-[#4f6ef7] aria-disabled:cursor-default aria-disabled:text-[#a0a4ab] aria-busy:cursor-wait"
                   >
                     {busyItem === item.id ? "넣는 중…" : "캘린더에 넣기"}
                   </button>
diff --git a/web/src/components/SyllabusReviewPanel.tsx b/web/src/components/SyllabusReviewPanel.tsx
index 7c1053e..95857ad 100644
--- a/web/src/components/SyllabusReviewPanel.tsx
+++ b/web/src/components/SyllabusReviewPanel.tsx
@@ -72,7 +72,7 @@ export default function SyllabusReviewPanel({
           >
             그만두기
           </button>
-          <Button onClick={onSave} disabled={chosen === 0 || saving}>
+          <Button onClick={onSave} disabled={!saving && chosen === 0} busy={saving}>
             {saving ? "넣는 중…" : label}
           </Button>
         </div>
diff --git a/web/src/components/Toggle.tsx b/web/src/components/Toggle.tsx
index 20f8c62..eaa6957 100644
--- a/web/src/components/Toggle.tsx
+++ b/web/src/components/Toggle.tsx
@@ -1,12 +1,13 @@
 interface ToggleProps {
   checked: boolean;
   label: string; // 화면에 글이 따로 있어 스위치에는 읽기용 이름만 준다
-  disabled?: boolean;
+  disabled?: boolean; // 지금은 켤 수 없음(브라우저가 알림을 막았다 등). 초점을 받지 않는다
+  busy?: boolean; // 바꾸는 중. 초점은 남기고 누름은 무시한다(Button의 busy와 같다). disabled와 같이 주지 않는다
   onChange: (checked: boolean) => void;
 }
 
 // 설정의 켜기·끄기 스위치. 알림 설정과 브라우저 알림 줄이 함께 쓴다
-export default function Toggle({ checked, label, disabled = false, onChange }: ToggleProps) {
+export default function Toggle({ checked, label, disabled = false, busy = false, onChange }: ToggleProps) {
   return (
     <button
       type="button"
@@ -14,8 +15,12 @@ export default function Toggle({ checked, label, disabled = false, onChange }: T
       aria-checked={checked}
       aria-label={label}
       disabled={disabled}
-      onClick={() => onChange(!checked)}
-      className={`relative h-6 w-11 shrink-0 rounded-full disabled:cursor-not-allowed disabled:opacity-45 ${checked ? "bg-[#4f6ef7]" : "bg-[#e5e7eb]"}`}
+      aria-disabled={busy || undefined}
+      aria-busy={busy || undefined}
+      onClick={() => {
+        if (!busy) onChange(!checked);
+      }}
+      className={`relative h-6 w-11 shrink-0 rounded-full disabled:cursor-not-allowed disabled:opacity-45 aria-disabled:cursor-wait aria-disabled:opacity-45 ${checked ? "bg-[#4f6ef7]" : "bg-[#e5e7eb]"}`}
     >
       <span className={`absolute top-1 size-4 rounded-full bg-white transition ${checked ? "left-6" : "left-1"}`} />
     </button>
diff --git a/web/src/components/onboarding/ConsentStep.tsx b/web/src/components/onboarding/ConsentStep.tsx
index dd005ad..c0c8fd5 100644
--- a/web/src/components/onboarding/ConsentStep.tsx
+++ b/web/src/components/onboarding/ConsentStep.tsx
@@ -101,7 +101,7 @@ export default function ConsentStep({ saving, onAgree }: ConsentStepProps) {
         ))}
       </div>
       <div className="mt-8 flex justify-end">
-        <Button onClick={() => onAgree({ incomeInfo: checked.income })} disabled={!requiredDone || saving}>
+        <Button onClick={() => onAgree({ incomeInfo: checked.income })} disabled={!saving && !requiredDone} busy={saving}>
           {saving ? "저장 중" : "동의하고 계속"}
         </Button>
       </div>
diff --git a/web/src/components/onboarding/IncomeConsentStep.tsx b/web/src/components/onboarding/IncomeConsentStep.tsx
index a5e0e36..635310b 100644
--- a/web/src/components/onboarding/IncomeConsentStep.tsx
+++ b/web/src/components/onboarding/IncomeConsentStep.tsx
@@ -25,7 +25,7 @@ export default function IncomeConsentStep({ saving, onAgree, onSkip, onBack }: I
         {onBack ? <Button secondary onClick={onBack}>이전</Button> : <span />}
         <div className="flex gap-2">
           <Button secondary onClick={onSkip}>건너뛰기</Button>
-          <Button onClick={onAgree} disabled={saving}>
+          <Button onClick={onAgree} busy={saving}>
             {saving ? "저장 중" : "동의하고 입력"}
           </Button>
         </div>
diff --git a/web/src/components/onboarding/ProfileStep.tsx b/web/src/components/onboarding/ProfileStep.tsx
index 7511892..8989432 100644
--- a/web/src/components/onboarding/ProfileStep.tsx
+++ b/web/src/components/onboarding/ProfileStep.tsx
@@ -66,7 +66,7 @@ export default function ProfileStep({
       </div>
       <div className="mt-8 flex justify-between gap-2">
         {onBack ? <Button secondary onClick={onBack}>이전</Button> : <span />}
-        <Button onClick={onNext} disabled={saving}>
+        <Button onClick={onNext} busy={saving}>
           {saving ? "저장 중" : last ? "저장하고 계속" : "다음"}
         </Button>
       </div>
diff --git a/web/src/screens/Notifications.tsx b/web/src/screens/Notifications.tsx
index c28a7a0..fcd052b 100644
--- a/web/src/screens/Notifications.tsx
+++ b/web/src/screens/Notifications.tsx
@@ -120,9 +120,12 @@ export default function Notifications({ navigate }: { navigate: (route: Route) =
         </div>
         <button
           type="button"
-          onClick={readEverything}
-          disabled={unread === null || unread === 0}
-          className="text-[13px] font-semibold text-[#4f6ef7] disabled:cursor-default disabled:text-[#a0a4ab]"
+          // 누르면 읽지 않은 알림이 0이 되어 disabled가 되고 초점이 빠졌다. 읽을 것이 없으면 aria-disabled로 알리고 누름은 무시한다
+          onClick={() => {
+            if (unread !== null && unread > 0) void readEverything();
+          }}
+          aria-disabled={unread === null || unread === 0 || undefined}
+          className="text-[13px] font-semibold text-[#4f6ef7] aria-disabled:cursor-default aria-disabled:text-[#a0a4ab]"
         >
           모두 읽음
         </button>
diff --git a/web/src/screens/Planner.tsx b/web/src/screens/Planner.tsx
index f818822..90d3c42 100644
--- a/web/src/screens/Planner.tsx
+++ b/web/src/screens/Planner.tsx
@@ -195,7 +195,7 @@ export default function Planner({ navigate }: { navigate: (route: Route) => void
             Google 캘린더를 쓰지 않으면 다가오는 마감일과 수업 일정을 파일로 받아 iPhone·삼성·Outlook 캘린더에 넣을 수 있어요.
           </p>
         </div>
-        <Button secondary onClick={downloadIcs} disabled={downloading}>
+        <Button secondary onClick={downloadIcs} busy={downloading}>
           {downloading ? "파일 만드는 중…" : ".ics 내려받기"}
         </Button>
       </section>
diff --git a/web/src/screens/Settings.tsx b/web/src/screens/Settings.tsx
index 2438b45..edf7f37 100644
--- a/web/src/screens/Settings.tsx
+++ b/web/src/screens/Settings.tsx
@@ -221,7 +221,7 @@ export default function Settings({ navigate, calendarResult = null }: SettingsPr
         {deleteError && <p role="alert" className="mt-3 text-[13px] text-[#c53030]">{deleteError}</p>}
         <div className="mt-6 flex justify-end gap-2">
           <Button secondary onClick={() => setDeleteOpen(false)}>취소</Button>
-          <Button disabled={deleteText !== "탈퇴" || deleting} onClick={removeAccount}>{deleting ? "지우는 중" : "탈퇴"}</Button>
+          <Button disabled={!deleting && deleteText !== "탈퇴"} busy={deleting} onClick={removeAccount}>{deleting ? "지우는 중" : "탈퇴"}</Button>
         </div>
       </Modal>
       {toast && (
```
