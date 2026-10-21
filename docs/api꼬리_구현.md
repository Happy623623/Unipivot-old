# api 설정·로그 꼬리 구현 (`fix/api-tails`)

> 📝 기준: docs/로그인진단_구현.md · docs/로그가림repr_구현.md · 인수인계 "앞에서 남긴 것"
> 적용할 곳: `main`(6948ebf, #82) 위. 그 트리에서 만들고 시험했다. web 쪽 PR(접근성 꼬리)과 겹치는 파일은 없다.
> **자리를 비운 동안 만든 PR이다.** 사용자에게 묻지 않고 범위와 방식은 모두 권장안으로 골랐다. 고른 것은 1장에 적었다.

앞 PR들이 남긴 작은 api 꼬리 두 개를 고친다.

1. **로그인 설정 검사**: 대시보드의 API 주소(`https://<ref>.supabase.co/rest/v1`)를 그대로 `SUPABASE_URL`에 넣으면, 공개키 주소가 `…/rest/v1/auth/v1/.well-known/jwks.json`이 된다. 그러면 모든 로그인이 503이 되는데, 시작 로그에는 아무 말이 없었다. 저장소 쪽(`storage_settings_problem`)에는 이 검사가 이미 있고, 로그인 쪽(`auth_settings_problem`)에만 빠져 있었다.
2. **로그 가리기**: 서식에 `token`·`secret`·`password` 같은 말이 든 로그는 그 자리 인자를 통째로 가리는데, **문자열 인자일 때만** 그랬다. bytes나 객체(SecretStr·주소 객체 등)로 넘긴 토큰은 패턴에 맞지 않으면 그대로 찍혔다.

| 값 | 결과 |
| --- | --- |
| 브랜치 | `fix/api-tails` |
| 파일 | 새 파일 없음, 고친 파일 4개(`api/app/auth.py`, `api/app/logging_redact.py`, 시험 2개) |
| web·DB | 안 바뀜(마이그레이션 없음) |
| api(DB 있이) | 1608 passed |
| api(DB 없이) | 1244 passed, 364 skipped(샌드박스 — 네 PC는 passed 하나 적고 skipped 하나 많다), ruff check·format 통과 |
| 되돌림 시험 | 8개 모두 시험이 잡음 — 6장 |

## 1. 정한 것 (사용자 부재 — 모두 권장안)

| 고른 것 | 이유 · 버린 선택지 |
| --- | --- |
| 주소는 `urlsplit`으로 본다 | 전에는 `startswith("https://")`만 봤다. 주소 부분이 비었거나(`http://`) 모양이 틀린 값(`http://[`)도 이제 잡는다. 전에 되던 값은 모두 그대로 통과한다: 끝 `/`, 대문자 스킴, `http://localhost:54321`, IPv6 `http://[::1]:54321`, 앞뒤 공백(검토가 직접 돌려 확인) |
| 막는 경로는 **Supabase API 경로**만 | `/rest/v1`·`/auth/v1`·`/storage/v1`·`/functions/v1`·`/realtime/v1`·`/graphql/v1`가 붙으면 막는다. `?`·`#`가 있어도 막는다(비어 있어도 공개키 주소를 틀어 놓는다). 처음엔 저장소 검사처럼 경로를 모두 막았다. 그런데 그러면 앞단에 경로를 붙인 자체 호스팅(`https://example.com/supabase`)의 로그인이 503이 된다. 저장소에는 `SUPABASE_STORAGE_BASE`라는 우회로가 있지만 로그인에는 없다(검토) |
| 문구는 저장소 검사와 같은 꼴, 값은 안 보인다 | "SUPABASE_URL에 경로가 붙어 있어요(…/rest/v1 같은 것. 값은 보여 주지 않음). 프로젝트 주소(https://<ref>.supabase.co)만 넣으세요". 모양이 틀린 값은 "주소 모양이 아니에요"(저장소와 같다) |
| 비밀키(HS256)를 쓰면 검사하지 않는다 | 그때는 로그인이 주소를 쓰지 않는다(전과 같다) |
| 민감 서식이면 숫자·None 말고는 모두 가린다 | 숫자(`numbers.Number` — bool·Decimal 포함)는 `%d`·`%f` 자리라, 바꾸면 서식이 깨져 그 로그 줄이 사라진다(검토가 Decimal로 재현). 앱 코드의 로그는 대부분 한국어 "토큰"을 써서 영향이 없다. 라이브러리 전체를 훑어도 걸리는 것은 pydantic-settings의 `Loading secret file: %s`(경로) 하나다(검토) |
| 문서 | 로그인 진단 구현 기록의 점검 목록 두 줄(9장). 명세 1·13장의 `AUTH_UNAVAILABLE` 설명("SUPABASE_URL이 비었거나 틀림")은 이 경우를 이미 담고 있어 그대로 둔다 |

## 2. 레포에 넣는 법

````text
[0] git checkout main && git pull
    git checkout -b fix/api-tails

[1] 새 파일은 없다.

[2] "패치" 절의 내용을 그대로 api-tails.patch로 저장해서 적용해 줘(고친 파일 4개, 전부 api).
    git apply api-tails.patch
    (적용한 뒤 api-tails.patch는 지워 줘)

[3] "문서에 반영하기" 절대로 docs/로그인진단_구현.md 2곳을 고쳐 줘.

[4] 이 문서도 docs/api꼬리_구현.md로 같이 넣어 줘.

[5] 검사: api> uv run ruff check . && uv run ruff format --check . && uv run pytest
    (DB 없이: 1243 passed, 365 skipped — 메모리가 빠듯하면 반씩 나눠 돌려도 된다)
    web은 바뀌지 않는다.

[6] 통과하면 "fix(api): SUPABASE_URL의 API 경로를 알리고, 민감 서식 로그는 문자열이 아닌 인자도 가린다"로 커밋해서 PR을 올려 줘.
    CI(api·web)가 통과하면(api 1608 passed) squash로 합치고 브랜치를 지운 뒤, main을 pull해서 최신 커밋 해시를 알려 줘.
    CI가 실패하면 기다리지 말고 바로 멈추고 실패한 시험 이름을 알려 줘.
````

## 3. 합친 뒤 확인

PowerShell에서 붙여 넣는다. `api/.env`는 읽지 않고, 설정을 바꿔 가며 검사 함수만 불러 본다(키·주소 값은 출력하지 않는다).

````text
cd api
@'
from app.auth import auth_settings_problem
from app.config import Settings
for url in ["https://abcdefghijklmnopqrst.supabase.co",
            "https://abcdefghijklmnopqrst.supabase.co/rest/v1",
            "https://example.com/supabase"]:
    problem = auth_settings_problem(Settings(_env_file=None, supabase_url=url, supabase_jwt_secret=""))
    print("OK" if problem is None else "PROBLEM")
'@ | uv run python -
````

예상 출력:

````text
OK
PROBLEM
OK
````

내 `api/.env`의 실제 설정은 API를 켤 때 로그로 본다. `로그인 설정 확인: …` 줄이 없으면 문제없다.

## 4. 구조

```text
api/app/auth.py   auth_settings_problem: 비었음 → @ → urlsplit(ValueError면 "주소 모양이 아니에요") → 스킴·주소 부분
                  → 예시 값(<…>) → Supabase API 경로(_API_PATH)·?·# → None
api/app/logging_redact.py  clean(): None·숫자(numbers.Number)는 그대로 → 민감 서식이면 _Redacted([REDACTED]) → 아니면 패턴으로 가림
```

## 5. 확인한 것

| 확인 | 결과 |
| --- | --- |
| `auth_settings_problem` 표 시험 | 새 경우 통과: `/rest/v1`, `/auth/v1/`, `?x=1`, 빈 `?`, 자체 호스팅 `https://example.com/supabase`(문제 없음), 자체 호스팅 + `/rest/v1`, `http://`, `http://[`(주소 모양), 비밀키를 쓰면 경로가 있어도 문제 없음. 값이 보이는지도 함께 본다 |
| 로그인 요청 | `SUPABASE_URL=…/rest/v1`이면 `GET /me`가 503 `AUTH_UNAVAILABLE`이고, 로그에 "경로가 붙어 있어요"가 남는다. 공개키는 받으러 가지 않는다 |
| 로그 가리기 | `token=%s`·`%r`에 객체, `secret %s`에 bytes → 가림. `token count %d, secret %s`(3, None), `token cost %.2f, %d`(Decimal, True) → 그대로 찍힌다 |
| 전체 | DB 있이 1608 passed, DB 없이 1244 passed, 364 skipped, ruff check·format 통과 |

## 6. 되돌림 시험

| 되돌린 것 | 잡은 시험 |
| --- | --- |
| C1 경로 검사 없음 | 표 시험(`/rest/v1`·`/auth/v1/`·`?x=1`), 503 로그 시험 |
| C2 경로를 모두 막음(저장소처럼) | 자체 호스팅 `https://example.com/supabase` |
| C3 빈 `?`는 통과(`parts.query`만 봄) | 빈 `?` |
| C4 주소 부분(netloc) 안 봄 | `http://` |
| C5 `urlsplit`의 ValueError를 안 잡음 | `http://[` |
| C6 민감 서식이어도 문자열만 가림(원래대로) | 객체·bytes 가리기 시험 |
| C7 숫자는 int·float만 그대로 | Decimal·bool 서식 시험 |
| C8 숫자도 가림 | 숫자·None 그대로 시험 |

## 7. 검토에서 고친 것

새 에이전트가 읽기만 하고 검토했다(시험은 직접 돌려 45 passed 확인).

| 지적 | 처리 |
| --- | --- |
| (중간) 경로를 모두 막으면 앞단 경로가 붙은 자체 호스팅의 로그인이 503이 된다(전에는 됐다). 저장소 문구가 안내하는 우회로(`SUPABASE_STORAGE_BASE`)도 로그인에는 소용없다 | Supabase API 경로만 막는다(C2) |
| (낮음) 민감 서식에서 Decimal 같은 숫자가 가려지면 `%d` 자리에서 서식이 깨져 그 로그 줄이 사라진다 | `numbers.Number`는 그대로 둔다(C7) |
| (낮음) `http://[`에 "https://로 시작하지 않아요"라고 답해 헷갈린다 | 저장소와 같은 "주소 모양이 아니에요"(C5) |
| (정보) 빈 `?`·`#`는 통과하지만 공개키 주소를 틀어 놓는다 | `?`·`#`가 있으면 막는다(C3). 저장소 검사도 같지만 이번엔 손대지 않았다(8장) |
| (문서) 로그인 진단 기록의 점검 목록, 로그 가리기 머리말 | 9장, 머리말은 패치에 들어 있다 |

검토가 문제없다고 본 것:
- **시작 경고**: 시작 경고는 서버를 막지 않는다.
- **경로 검사 위치**: 공개키를 받기 전에만 막는다.
- **값 노출**: 예시 값 말고는 값을 보이지 않는다.
- **로그 가리기**: bool·IntEnum·None은 그대로다. datetime·Enum은 `%s`에서 `[REDACTED]`로 찍히고 서식은 깨지지 않는다. uvicorn 접근 로그는 인자 개수·타입이 그대로다.

## 8. 남은 것

- 저장소 주소 검사(`storage.py` `_url_problem`)도 빈 `?`·`#`를 통과시킨다 — 같은 한 줄을 넣을지
- 탈퇴 중에 LMS 동기화가 새 파일을 넣으면 그 파일은 행만 지워지고 저장소에 남는다(드문 경쟁, 인수인계 7장)
- 환경 변수 이름을 `SUPABASE_SECRET_KEY`로 바꿀지(사용자가 정한다)

## 9. 문서에 반영하기

````text
[1] 파일 위치: docs/로그인진단_구현.md
이 글을 찾아서:
| `SUPABASE_URL` 점검 | 비었음, 계정 정보(`@`)가 있음, `http(s)://`로 시작하지 않음, 예시 값(`<…>`)이 남음. 값은 `http(s)://`로 시작하고 `@`가 없을 때만 로그에 보여 준다. 문제가 있으면 공개키를 받으러 가지 않는다 | DB 주소나 키를 잘못 넣었을 때 비밀번호·키가 로그에 남지 않게 한다 |
이렇게 바꿔 줘.
| `SUPABASE_URL` 점검 | 비었음, 계정 정보(`@`)가 있음, 주소 모양이 아님·`http(s)://`로 시작하지 않음, 예시 값(`<…>`)이 남음, Supabase API 경로(`/rest/v1` 등)나 `?`·`#`가 붙음(10/21, docs/api꼬리_구현.md). 값은 `http(s)://`로 시작하고 `@`가 없을 때만 로그에 보여 준다(경로 문구에는 값을 넣지 않는다). 문제가 있으면 공개키를 받으러 가지 않는다 | DB 주소나 키를 잘못 넣었을 때 비밀번호·키가 로그에 남지 않게 한다 |

[2] 파일 위치: docs/로그인진단_구현.md
이 글을 찾아서:
  auth_settings_problem    SUPABASE_URL 점검(비었음·계정 정보·http 아님·예시 값). 주소처럼 보이는 값만 보여 준다
이렇게 바꿔 줘.
  auth_settings_problem    SUPABASE_URL 점검(비었음·계정 정보·http 아님·예시 값·API 경로). 주소처럼 보이는 값만 보여 준다
````

## 10. 코드와 패치

새 파일은 없다.

### 패치

```diff
diff --git a/api/app/auth.py b/api/app/auth.py
index 329dfc6..669c729 100644
--- a/api/app/auth.py
+++ b/api/app/auth.py
@@ -6,10 +6,12 @@
 """
 
 import logging
+import re
 from dataclasses import dataclass
 from datetime import timedelta
 from functools import lru_cache
 from typing import Annotated, Any
+from urllib.parse import urlsplit
 
 import jwt
 from fastapi import Depends, Header
@@ -39,6 +41,10 @@ def _jwks_client(jwks_url: str) -> jwt.PyJWKClient:
     return jwt.PyJWKClient(jwks_url, cache_keys=True, timeout=JWKS_TIMEOUT_SECONDS)
 
 
+# 프로젝트 주소 뒤에 붙으면 안 되는 Supabase API 경로(대시보드에서 복사할 때 따라온다)
+_API_PATH = re.compile(r"(?i)/(rest|auth|storage|functions|realtime|graphql)/v1(/|$)")
+
+
 def auth_settings_problem(settings: Settings) -> str | None:
     """로그인 확인 설정에서 고칠 점. 없으면 None이다. 앱을 시작할 때와 공개키를 받기 전에 본다.
 
@@ -54,10 +60,22 @@ def auth_settings_problem(settings: Settings) -> str | None:
             "SUPABASE_URL에 계정 정보(@)가 들어 있어요(값은 보여 주지 않음). DB 주소 대신"
             " Supabase 프로젝트 주소(https://….supabase.co)를 넣으세요"
         )
-    if not url.lower().startswith(("https://", "http://")):
+    try:
+        parts = urlsplit(url)
+    except ValueError:  # "http://[" 같은 값. 여기서 터지면 서버가 뜨지 않는다
+        return "SUPABASE_URL이 주소 모양이 아니에요(값은 보여 주지 않음). Supabase 프로젝트 주소를 넣으세요"
+    if parts.scheme.lower() not in ("https", "http") or not parts.netloc:
         return "SUPABASE_URL이 https://로 시작하지 않아요(값은 보여 주지 않음). Supabase 프로젝트 주소를 넣으세요"
     if "<" in url or ">" in url:
         return f"SUPABASE_URL에 예시 값이 남아 있어요: {url}. Supabase 프로젝트 주소로 바꾸세요"
+    # 공개키 주소는 {SUPABASE_URL}/auth/v1/…라서 대시보드의 API 주소(…/rest/v1)를 그대로 넣으면 엉뚱한 곳이 404로
+    # 답하고 로그인이 모두 503이 된다. 원인이 안 보여서 여기서 알린다. 경로는 Supabase API 경로일 때만 막는다 —
+    # 앞단에 경로를 붙인 자체 호스팅(https://example.com/supabase)은 그대로 된다(검토). ?·#는 늘 막는다
+    if _API_PATH.search(parts.path) or "?" in url or "#" in url:
+        return (
+            "SUPABASE_URL에 경로가 붙어 있어요(…/rest/v1 같은 것. 값은 보여 주지 않음). 프로젝트 주소"
+            "(https://<ref>.supabase.co)만 넣으세요"
+        )
     return None
 
 
diff --git a/api/app/logging_redact.py b/api/app/logging_redact.py
index 194d599..51e8ad1 100644
--- a/api/app/logging_redact.py
+++ b/api/app/logging_redact.py
@@ -5,10 +5,12 @@ LearningX(Canvas) 개인 액세스 토큰, Fernet 암호문, Anthropic·Google A
 (`sb_secret_…`), 주소에 붙은 API 키(온통청년 apiKeyNm=… 등), 공지 첨부 주소의 열쇠(Canvas
 `verifier=…`, 파일 저장소 서명 `X-Amz-Signature=…`·`X-Amz-Credential=…`). 요청 주소처럼 문자열이
 아닌 인자도 `str()`·`repr()`로 바꿔 보고 가릴 것이 있으면 가린다 — 예외를 `%r`로 찍는 로그가 많고,
-`repr()`에만 나오는 값(요청 주소·헤더를 든 예외)도 있다.
+`repr()`에만 나오는 값(요청 주소·헤더를 든 예외)도 있다. 서식 문자열에 token·secret·password 같은 말이
+있으면 그 인자는 숫자·None 말고는 무엇이든 통째로 가린다.
 """
 
 import logging
+import numbers
 import re
 from collections.abc import Mapping
 from typing import Any
@@ -47,7 +49,7 @@ _PATTERNS: tuple[tuple[re.Pattern[str], str], ...] = (
         r"\1" + _REDACTED,
     ),
 )
-# 이런 말이 든 서식 문자열이면 인자(문자열)를 통째로 가린다: logger.info("Authorization: %s", value)
+# 이런 말이 든 서식 문자열이면 인자를 통째로 가린다(숫자·None 말고는 종류를 가리지 않는다): logger.info("Authorization: %s", value)
 _SENSITIVE_FORMAT = re.compile(r"(?i)authorization|bearer|token|secret|password|api[_-]?key")
 
 
@@ -103,8 +105,12 @@ def install_log_redaction() -> None:
             def clean(item: Any) -> Any:
                 if isinstance(item, str):
                     return _REDACTED if mask_all else redact(item)
-                if item is None or isinstance(item, int | float):
-                    return item
+                if item is None or isinstance(item, numbers.Number):
+                    return item  # 숫자(bool·Decimal 포함)는 %d·%f 자리라 바꾸면 서식이 깨진다
+                if mask_all:
+                    # 서식에 token·secret이 들어 있으면 그 자리의 값은 무엇이든 가린다. 문자열만 가리면 bytes·
+                    # SecretStr·주소 객체로 넘긴 토큰이 패턴에 안 맞을 때 그대로 찍혔다(숫자·None은 그대로 둔다)
+                    return _Redacted(_REDACTED, _REDACTED)
                 try:  # httpx 요청 로그의 주소(httpx.URL) 같은 객체, `%r`로 찍는 예외
                     text = str(item)
                 except Exception:
diff --git a/api/tests/test_auth.py b/api/tests/test_auth.py
index c4790ab..ac721da 100644
--- a/api/tests/test_auth.py
+++ b/api/tests/test_auth.py
@@ -249,6 +249,7 @@ def test_empty_or_wrong_project_url_is_503_without_showing_secrets(
             "postgresql://postgres.abcd:pw-1234@pooler.example.com:5432/postgres",
             "로그인 확인 실패(503): SUPABASE_URL에 계정 정보(@)가 들어 있어요",
         ),
+        (PROJECT_URL + "/rest/v1", "로그인 확인 실패(503): SUPABASE_URL에 경로가 붙어 있어요"),
     ):
         jwks = StaticJwks(JWKS)
         with jwks_app(monkeypatch, jwks, supabase_url=url) as client:
@@ -272,6 +273,25 @@ def test_empty_or_wrong_project_url_is_503_without_showing_secrets(
         ("sb_secret_pasted-by-mistake", "", "https://로 시작하지 않아요", False),
         ("postgresql://postgres:pw-1234@db.example.com:5432/postgres", "", "계정 정보(@)", False),
         ("https://user:pw-1234@abcdefghijklmnopqrst.supabase.co", "", "계정 정보(@)", False),
+        (
+            PROJECT_URL + "/rest/v1",
+            "",
+            "경로가 붙어 있어요",
+            False,
+        ),  # 대시보드 API 주소를 그대로 복사
+        (PROJECT_URL + "/auth/v1/", "", "경로가 붙어 있어요", False),
+        (PROJECT_URL + "?x=1", "", "경로가 붙어 있어요", False),
+        (PROJECT_URL + "?", "", "경로가 붙어 있어요", False),  # 빈 ?도 공개키 주소를 틀어 놓는다
+        ("https://example.com/supabase", "", None, True),  # 앞단 경로가 붙은 자체 호스팅은 그대로
+        ("https://example.com/supabase/rest/v1", "", "경로가 붙어 있어요", False),
+        ("http://", "", "https://로 시작하지 않아요", False),  # 주소가 없다
+        ("http://[", "", "주소 모양이 아니에요", False),  # urlsplit이 ValueError를 낸다
+        (
+            PROJECT_URL + "/rest/v1",
+            "legacy-secret-legacy-secret-legacy-secret",
+            None,
+            True,
+        ),  # 비밀키면 안 쓴다
     ],
 )
 def test_auth_settings_problem(url: str, secret: str, problem: str | None, shown: bool) -> None:
diff --git a/api/tests/test_security.py b/api/tests/test_security.py
index 3d704ca..ab61f58 100644
--- a/api/tests/test_security.py
+++ b/api/tests/test_security.py
@@ -1,6 +1,7 @@
 """토큰 암호화 키 교체와 로그 마스킹 (PRD 7장 토큰 보안)."""
 
 import logging
+from decimal import Decimal
 
 import httpx
 import pytest
@@ -197,3 +198,28 @@ def test_log_argument_that_cannot_be_stringified_does_not_break_the_caller() ->
     logger.setLevel(logging.INFO)  # 레코드를 만든다(가리기는 여기서 돈다)
     logger.propagate = False  # 처리기가 없어 서식은 만들지 않는다
     logger.info("값 %s", Unprintable())
+
+
+def test_a_sensitive_format_hides_any_kind_of_argument(caplog: pytest.LogCaptureFixture) -> None:
+    """서식에 token·secret이 있으면 그 자리의 값은 문자열이 아니어도 가린다(숫자·None은 그대로)."""
+    install_log_redaction()
+
+    class Holder:  # 패턴에 안 맞는 토큰을 든 객체(SecretStr·주소 객체 같은 것)
+        def __str__(self) -> str:
+            return "opaque-value-123"
+
+        def __repr__(self) -> str:
+            return "Holder(opaque-value-123)"
+
+    logger = logging.getLogger("unipivot.test.sensitive")
+    with caplog.at_level(logging.INFO, logger="unipivot.test.sensitive"):
+        logger.info("refresh token=%s", Holder())
+        logger.info("refresh token=%r", Holder())
+        logger.info("client secret %s", b"opaque-bytes-456")
+        logger.info("token count %d, secret %s", 3, None)
+        logger.info("token cost %.2f, %d", Decimal("1.5"), True)  # 숫자는 서식이 깨지지 않게 그대로
+    for value in ("opaque-value-123", "opaque-bytes-456"):
+        assert value not in caplog.text
+    assert caplog.text.count("[REDACTED]") == 3
+    assert "token count 3, secret None" in caplog.text  # 숫자·None은 가리지 않는다
+    assert "token cost 1.50, 1" in caplog.text
```
