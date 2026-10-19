# README — 레포 첫 화면과 체크리스트 정리

> 📝 기준: CLAUDE.md · PRD v0.9 14장 · 2026-10-19
>
> 적용할 곳: **캘린더 연결 해제(앞 PR)까지 합친 main** 위 새 브랜치. **문서만 바뀐다**(코드 변경 없음)

레포에 README가 없었다. GitHub 소스가 산출물이라 처음 여는 사람(심사·멘토)이 무엇인지·어떻게 켜는지·
어떻게 검사하는지를 한 화면에서 보게 했다. PRD 체크리스트에서 이미 끝난 항목 하나도 닫는다.

## 정한 것

1. **README는 길잡이만 한다.** 규칙은 `CLAUDE.md` 셋에, 계약은 명세에 있다. README는 "무엇인지 → 켜는 법
   → 설정 → 정기 작업 → 명령 → 검사"까지만 적고 나머지는 그 문서로 보낸다. 같은 말이 두 곳에 있으면
   하나가 썩는다.
2. **설정 표는 키 이름과 쓰는 곳만.** 값·예시는 `.env.example`에만 둔다(레포 규칙).
3. **체크리스트 "취업 상태를 프로필에 둘지"를 닫았다.** F-21(10/17)에서 연 소득과 함께 넣었는데
   체크리스트가 그대로였다. 다른 미결 항목은 아직 열려 있는 게 맞아서 손대지 않았다.

## 레포에 넣는 법

````text
[0] 캘린더 연결 해제 PR까지 main에 들어가 있어야 한다.
    git checkout main && git pull
    git checkout -b docs/readme

[1] "코드" 절의 README.md를 레포 루트에 그대로 만들어 줘.

[2] "문서에 반영하기" 절대로 docs/PRD_v0.9.md 1곳을 고쳐 줘.

[3] 이 문서도 docs/README_기록.md로 같이 넣어 줘.

[4] 검사: 문서만 바뀌었으니 테스트는 안 돌려도 된다. git status에 세 파일만 보이면 된다.

[5] "docs: README와 체크리스트 정리"로 커밋해서 PR을 올려 줘. CI는 그대로 통과한다. db push 없음.
````

## 확인한 것

| 확인 | 결과 |
| --- | --- |
| 명령 | README의 모든 명령은 레포에 있는 것이다(`app.youth.cli`·`app.crawl.cli`·`app.eval.cli`·`app.push.keys`, CI의 검사 명령) |
| 설정 이름 | 표의 키 이름은 `api/app/config.py`·`.env.example`과 같다 |
| 정기 작업 표 | `api/app/scheduler.py`의 네 작업·주기와 같다 |
| 체크리스트 | PRD F-02 항목에 "취업 상태"가 이미 들어 있어 닫는 게 맞다 |

## 남은 것

- 배포 주소·데모 계정 안내는 배포한 뒤 README에 한 절 더한다.
- 영어 README는 만들지 않았다(심사는 한국어다).

## 문서에 반영하기

````text
docs/PRD_v0.9.md — 1곳

[1] 14장 체크리스트. 이 줄을 찾아서

- [ ] 취업 상태를 프로필에 둘지 — 청년정책 410건 중 77건에 취업 상태 조건이 있다. 지금은 원문 확인 필요 (10/13)

이 줄로 바꿔 줘.

- [x] 취업 상태를 프로필에 둘지 → 뒀다. 연 소득(만 원)과 함께 프로필 항목이 됐고 판정에도 쓴다 (10/17, F-21, docs/F21_마무리.md)
````

## 코드

새로 만드는 파일 1개다.

#### `README.md`

```markdown
# UNIPIVOT — 학내 정보 에이전트

대학생이 장학금·청년정책·공모전 공고에 **지원할 수 있는지 판정**받고, **준비 일정**을 만들고, **LMS 과제·자료**를
챙기는 웹 서비스다. SW창업캡스톤(AOP) 과제로, LLM Tool Calling 기반 에이전트 MVP · 최종 데모 · 테스트 시나리오
측정(성공률·응답시간·비용)이 산출물이다.

- 공고는 세 길로 들어온다 — 저장한 학교 공지(명령), 온통청년 청년정책(매일 자동), 학생이 올린 포스터(화면)
- LLM은 **공고 원문에서 요건을 구조화**하는 데까지만 쓰고, **자격 판정은 코드**가 한다(`docs/판정엔진_규칙.md`)
- 모든 LLM·외부 호출은 실행 로그(`agent_runs`·`tool_calls`·`llm_calls`)에 남고, 그 로그로 지표를 센다

## 구조

| 폴더 | 내용 | 규칙 |
| --- | --- | --- |
| `web/` | Next.js 16 + Tailwind 4 화면 | `web/CLAUDE.md` |
| `api/` | FastAPI(Python 3.12, uv) API·에이전트·정기 작업 | `api/CLAUDE.md` |
| `supabase/migrations/` | DB 스키마(파일 이름 순으로 적용) | `docs/ERD_v0.9.md` |
| `docs/` | 기준 문서와 구현 기록 | `CLAUDE.md` |

기준 문서는 충돌하면 위가 이긴다: `docs/API_명세_v0.4.md`(계약) → `docs/ERD_v0.9.md`(DB) → `docs/PRD_v0.9.md`(기능·우선순위)
→ `docs/판정엔진_규칙.md`. 기능마다 `docs/*_구현.md`에 정한 것·확인한 것·되돌림 시험을 적어 둔다.

## 시작하기

필요한 것: Python 3.12 + [uv](https://docs.astral.sh/uv/), Node 22, Supabase 프로젝트(Postgres + Google 로그인),
[Supabase CLI](https://supabase.com/docs/guides/cli).

    # 1. DB — Supabase 프로젝트에 마이그레이션을 적용한다
    npx supabase link --project-ref <ref>
    npx supabase db push

    # 2. API
    cd api
    cp .env.example .env            # 값을 채운다(아래 "설정")
    uv sync
    uv run uvicorn app.main:app --reload     # http://localhost:8000, 문서는 /docs

    # 3. 화면
    cd web
    cp .env.local.example .env.local
    npm ci
    npm run dev                     # http://localhost:3000

`NEXT_PUBLIC_USE_MOCKS=true`면 화면이 API 없이 목데이터로 뜬다(상태 미리보기 패널 포함). 실제 API를 붙이면 `false`.

### 설정 (`api/.env`)

| 값 | 쓰는 곳 |
| --- | --- |
| `DATABASE_URL` · `SUPABASE_URL` · `SUPABASE_JWT_SECRET` | DB, 로그인 토큰 검증 |
| `TOKEN_ENCRYPTION_KEY` | Google·LMS 토큰 암호화 |
| `GOOGLE_CLOUD_PROJECT` | Gemini(요건 추출·공지 분류). 없으면 모델이 필요한 일만 실패한다 |
| `GOOGLE_OAUTH_CLIENT_ID` · `GOOGLE_OAUTH_CLIENT_SECRET` | Google 캘린더(증분 승인) |
| `SUPABASE_SERVICE_ROLE_KEY` · 버킷 | 포스터 파일 저장 |
| `YOUTH_API_KEY` | 온통청년 수집 |
| `VAPID_PRIVATE_KEY` · `VAPID_SUBJECT` | 웹 푸시(`uv run python -m app.push.keys`로 만든다) |
| `INTERNAL_API_KEY` | `/internal/*` 서비스 키 |
| `SCHEDULER_ENABLED` | 서버 안 정기 작업(아래) |

키·`.env`·`.env.local`은 레포에 넣지 않는다. 예시는 `.env.example`에만 적는다.

## 정기 작업

`SCHEDULER_ENABLED=true`면 서버가 직접 돈다(`api/app/scheduler.py`). 같은 일을 `POST /internal/jobs/*`로 손으로
부를 수도 있다(둘은 같은 함수다). 워커는 하나로 둔다.

| 작업 | 주기 |
| --- | --- |
| 알림 발송(알림함 → 웹 푸시) | 5분 |
| LMS 동기화(과목·과제·자료·공지) | 30분 |
| 공고 알림 예약(새 적격·마감 임박·정보 필요) | 매일 KST 08:30 |
| 온통청년 정책 수집 | 매일 KST 04:00 |

## 명령 (`api` 폴더에서)

    uv run python -m app.youth.cli fetch              # 온통청년 정책 수집(정기 작업과 같은 함수)
    uv run python -m app.crawl.cli import <공지 폴더>   # 저장한 학교 공지 넣기 + 요건 추출
    uv run python -m app.eval.cli                      # 테스트 시나리오 전부 실행(시험용 DB에서만)
    uv run python -m app.eval.cli --out eval_baseline.json   # 베이스라인 저장(결과 JSON은 레포에 넣지 않는다)
    uv run python -m app.push.keys                     # 웹 푸시 VAPID 키 만들기

## 검사

| 위치 | 명령 |
| --- | --- |
| api | `uv run ruff check . && uv run ruff format --check . && uv run pytest` |
| web | `npm run typecheck && npm run build` |

`@pytest.mark.db` 테스트는 `DATABASE_URL`이 있을 때만 돈다. CI는 Postgres 컨테이너에 마이그레이션을 적용해서
돌리고, 실제 Supabase DB로는 돌리지 않는다. 테스트 시나리오(지금 18개)는 `python -m app.eval.cli`로 돌리며
모델·Google·파일 저장소·LMS는 녹화한 응답을 쓴다. 측정 결과는 `docs/베이스라인.md`에 절을 더해 비교한다.

## 작업 방식

작업 하나 = 브랜치 하나 = PR 하나. 문서에 없는 엔드포인트·필드·컬럼을 만들지 않고, 필요하면 기준 문서를 같은
PR에서 고친다. 자세한 규칙은 `CLAUDE.md`·`api/CLAUDE.md`·`web/CLAUDE.md`에 있다.
```
