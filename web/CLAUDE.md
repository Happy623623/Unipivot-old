# web — Next.js 화면

Figma Make 시안을 옮긴 코드다. 아래 화면·데이터 규칙을 지키며 이어서 만든다.

기준 문서는 `docs/API_명세_v0.4.md`(계약), `docs/PRD_v0.9.md`(기능·우선순위), `docs/판정엔진_규칙.md`(판정 라벨·화면 문구)다.

## 구조

- `src/app/` 라우트. `(app)`은 사이드바가 있는 화면(`AppFrame`), `(auth)`는 로그인·온보딩, `auth/callback`은 로그인 콜백
- `src/screens/` 화면, `src/components/` 공통 조각. 페이지 파일은 화면에 `navigate`와 주소에서 읽은 값만 넘긴다
- `src/lib/routes.ts` Route ↔ URL 변환, `src/components/AppFrame.tsx` 내 정보·동의 확인과 헤더 배지 숫자
- `src/lib/push.ts` 브라우저 푸시(권한·구독, 목 모드 흉내), `src/lib/usePushSubscription.ts` 켜기·끄기 상태와 앱 열 때 되올림, `public/sw.js` 서비스 워커(번들을 타지 않는다)
- `src/lib/useDocuments.ts` 문서함 목록(더 보기). 받기는 `components/DownloadButton.tsx` → `downloadDocument`(누를 때마다 5분짜리 주소를 새로 받는다) → `lib/saveFile.ts`의 `fetchDownload`(그 주소에서 fetch로 받아 원래 이름으로 저장, 저장소가 거절하면 `DOWNLOAD_FAILED` 안내)다. 받기 주소는 화면에 들고 있지 않는다
- `src/lib/useNotifications.ts` 알림함 목록·읽음 처리. 헤더 패널(`NotificationPanel`)과 전체 목록(`screens/Notifications.tsx`)이 각각 하나씩 쓰고, 배지 숫자는 `AppShell`의 `setUnreadNotifications`로 올린다
- `src/lib/useLoadMoreFocus.ts` 두 목록의 "더 보기" 뒤 초점과 화면 읽기 안내. 마지막 쪽이 와서 버튼이 사라지면 새로 온 줄 중 화면 첫 줄(`data-row`)로, 422로 처음부터 다시 받으면(`generation`) 첫 줄로 초점을 옮긴다. 누를 때 초점이 있었고 지금 `<body>`일 때만 옮긴다
- `src/api/client.ts` 화면이 쓰는 데이터 함수. 계정 함수는 `accountClient.ts`, LMS 함수는 `lmsClient.ts`, 문서함 함수는 `documentsClient.ts`에 두고 client.ts가 다시 내보낸다
- `src/lib/useSessionWatch.ts` 다른 탭과 로그인 맞추기. `AppFrame`·온보딩이 쓴다 — 다른 탭에서 로그아웃·탈퇴하거나 로그인이 끝나면(SIGNED_OUT) 바로 로그인 화면으로, 다시 보일 때 세션이 없으면 로그인 화면·다른 계정이면 홈으로 간다(`location.replace`). 로그아웃·탈퇴하는 쪽은 세션을 지우기 직전에 `announceLeaving`으로 이유를 알린다(로그인 화면 안내가 이유를 따른다)
- `src/components/Modal.tsx` 공용 창. 초점은 `lib/useDialogFocus.ts`가 맡는다 — 열리면 창으로, Tab은 창 안에서 돌고, Esc로 닫히며, 닫히면 연 버튼으로 돌아간다(Safari처럼 누른 버튼에 초점을 주지 않는 브라우저는 1초 안에 누른 버튼을 연 버튼으로 본다). 창을 직접 만들지 말고 이것을 쓴다. 창 안의 일(저장·탈퇴 등)이 진행 중이면 `busy`를 넘긴다 — 끝나거나 15초가 지날 때까지 Esc·닫기·덮개·취소로 닫히지 않는다(`lib/useCloseLock.ts`, 닫힌 뒤 실패하면 오류가 안 보여서). 창 안의 취소는 `ModalCancelButton`을 쓴다. 되돌릴 수 없는 일의 경고처럼 창을 열 때 함께 읽혀야 하는 글은 `ModalDescription`으로 쓴다(`aria-describedby`). 옆에서 나오는 헤더 알림 패널(`NotificationPanel`)도 같은 훅을 쓴다
- `src/lib/useRouteFocus.ts` 다른 화면으로 옮겨 초점이 `<body>`로 빠지면 새 화면 제목(`main`의 h1, `Header`가 `tabIndex=-1`로 그린다)으로 옮긴다. 처음 열 때, 누른 것에 초점이 남아 있을 때(Chrome 사이드바), 창이 열려 있을 때는 그대로 둔다
- `src/lib/useFocusHandoff.ts` 누른 버튼이 일을 마치고 다른 것으로 바뀌어 사라질 때 초점을 그 자리로 넘긴다(`mark`·`target`, 사라지기만 하면 `focusIfLost`). `src/lib/useLateAnnouncement.ts` 화면을 열 때부터 정해진 안내를 live 영역에 늦춰 넣는다(그래야 읽힌다)
- `src/components/PosterOriginal.tsx` 내가 올린 포스터의 원문 보기 창. 열 때 `getPosterUrl`(`posterClient.ts`)로 5분짜리 주소를 받고 닫으면 버린다. 사진은 창 안에, PDF는 새 탭 링크로 연다
- `src/api/http.ts` 실제 API 호출과 오류 도구(`apiFetch`, `errorMessage`, `fieldErrors`, `waitForRun`), `src/api/mockClient.ts` 목 모드 구현
- `src/mocks/` 목데이터, `src/dev/` 상태 미리보기(목 모드에서만 보인다)
- `src/types/api.ts` API 명세와 1:1인 타입. LMS·강의자료 타입은 `lms.ts`에 두고 api.ts가 다시 내보낸다

## 데이터

- 화면은 `src/api/client.ts` 함수로만 데이터를 받는다. 함수 이름·인자·반환 타입은 그대로 두고 안쪽만 목에서 `apiFetch`로 바꾼다(스킬 `mock-to-api`).
- 오류 문구는 `errorMessage(error, 화면 문구)`로 만들고, 422 항목 오류는 `fieldErrors`로 입력란 아래에 보여 준다. 실제 모드에서 아직 API가 없는 계정·LMS 기능은 목데이터 대신 빈 값이나 `NOT_READY`(준비 중) 안내를 준다(알림 설정 P1만 목 값).
- 타입 필드 이름은 API 명세와 같아야 한다. 없는 값이 필요하면 만들지 말고 그 자리에 `// TODO(api):`를 남긴다.
- `NEXT_PUBLIC_USE_MOCKS=true`면 목데이터와 상태 미리보기를 쓴다. 목 상태는 메모리에만 있어 새로고침하면 초기화된다.
- Supabase는 로그인과 세션 토큰에만 쓴다. 데이터는 DB를 직접 읽지 않고 FastAPI로만 받는다.
- 날짜·D-day·카테고리 이름은 `src/lib/format.ts`로만 만든다. 시간대는 KST, 마감일이 없으면 "상시"다. 공고 카드의 D-day는 서버가 준 `d_day`를 쓴다.
- 오늘·지금은 `client.ts`의 `todayKst()`·`nowIso()`로만 얻는다. 목 모드는 목데이터 기준일(2026-10-05)을 돌려준다.
- 페이지 파일은 `"use client"`로 쓰고, 주소 쿼리를 읽으면 `Suspense`로 감싼다. 화면 코드에서 `window`·`localStorage`를 직접 쓰지 않는다. 한 파일은 300줄을 넘지 않게 나눈다.

## 판정 라벨 (4종 고정)

| display_status | 라벨 | 톤 | 주 행동 버튼 |
| --- | --- | --- | --- |
| eligible | 지원 가능 | green | 준비 일정 만들기 |
| missing_info | 정보 필요 | yellow | 정보 입력 |
| needs_review | 원문 확인 필요 | gray | 원문 보기 |
| ineligible | 지원 어려움 | red | 대체 공고 보기 |

- "확인 필요"는 홈 요약 카드에서 정보 필요와 원문 확인 필요를 합쳐 부를 때만 쓴다.
- 지원 어려움 공고는 홈 목록 맨 아래 "지원 어려움 N개" 묶음 안에만 둔다. 기본은 접힘이다.
- `needs_review`가 true인 지원 가능 공고는 라벨 옆에 작은 "원문 확인 필요" 배지를 붙인다.

## 기준값

- 마감 임박은 D-3 이내(당일 포함), 새 공고는 서버가 준 `is_new`를 그대로 쓴다.
- 카테고리 칩: 전체, 장학금, 교내 프로그램, 청년정책, 공모전, 대외활동, 기타(research·exam·etc). 상세에서는 research를 "연구 참여", exam을 "시험·자격"으로 쓴다.
- 포스터는 JPG·PNG·WEBP와 2쪽 이하 PDF, 20MB 이하다. HEIC(아이폰 기본 사진 형식)는 아직 받지 않고, 고르면 JPG로 바꿔 올리라고 안내한다.
- 포스터 공고는 "내가 올린 포스터"로 표시하고 업로더 이름은 보여주지 않는다(`uploaded_by_me`). LMS 토큰은 끝 4자리만 보여준다.

## 디자인 토큰 (새 색·글꼴 추가 금지)

- 주색 #4f6ef7(hover #425fe0), 연한 주색 #eef2ff, 강조 테두리 #cfd7ff
- 글자 #181a20, 보조 #667085, 본문 보조 #344054, 비활성 #a0a4ab
- 배경 #f7f8fa, 카드 흰색, 테두리 #e5e7eb. 모서리는 버튼 10px, 카드 14–18px, 칩은 완전히 둥글게
- 상태 톤(글자/배경): green #15803d/#ecfdf3, yellow #b7791f/#fff8e1, red #c53030/#fef2f2, gray #475467/#f2f4f7
- 글꼴은 `src/app/layout.tsx`의 Inter + Noto Sans KR(next/font). `prefers-reduced-motion` 규칙을 유지하고, 움직임은 사용자 동작(열기·펼치기·완료)에 대한 반응에만 쓴다.

## 문구

- 해요체 평서문. 버튼은 실제로 일어나는 동작을 쓰고, 같은 동작은 흐름 내내 같은 이름을 쓴다("준비 일정 만들기" → 토스트 "준비 일정을 만들었어요").
- 오류는 무엇이 잘못됐고 어떻게 고치는지 말한다. 사과하거나 뭉뚱그리지 않는다. API 에러는 `code`로 분기한다(API 명세 13장).
- 빈 화면은 다음 행동 하나를 버튼과 함께 안내한다.
- 소득·수급 같은 민감 항목 위에는 "자격 판정에만 쓰고, 탈퇴하면 바로 지워요"를 붙인다.

## 하지 않는 것

- 라우터·상태관리 라이브러리 추가(아이콘은 lucide-react만), 채팅 UI
- 실명이나 토큰 원문 표시
- 화면 컴포넌트 안에서 `fetch` 직접 호출(데이터는 `client.ts`로만)
- 진행 중인 버튼을 `disabled`로 막기(키보드 초점이 빠진다). `Button`·`Toggle`의 `busy`(aria-disabled·aria-busy, 초점 유지)를 쓰고(다른 일이 진행 중이라 못 누르는 버튼은 `Button`의 `inactive` — aria-busy 없이), `disabled`는 지금 할 수 없는 일(입력이 덜 됨)에만 쓴다. 둘을 같이 주지 않는다. 직접 만든 버튼·체크박스·스위치도 `aria-disabled`로 알리고 누름(체크박스는 `onChange`)을 무시한다. 누르면 할 일이 없어지는 버튼("모두 읽음")도 같다
