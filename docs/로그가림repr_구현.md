# 로그 가림 — `%r`로 찍은 예외의 `repr()`도 가린다(PRD 7장 토큰 보안)

> 📝 기준: PRD v0.9 7장 토큰 보안(로그에 토큰을 남기지 않는다) · `api/app/logging_redact.py` · 2026-10-20
>
> 적용할 곳: **탈퇴 저장소 가드 PR(#69)까지 합친 main** 위 새 브랜치. 앞 PR들과 겹치는 파일이 없다. **마이그레이션 없음** ·
> 명세·PRD·ERD 안 바뀜

로그 가림(`install_log_redaction`)은 문자열이 아닌 로그 인자를 `str()`로 바꿔 보고 가릴 것이 있으면 가렸다. 그런데 예외는
대개 `%r`로 찍는다(`logger.warning("… 실패: %r", exc)`가 15곳). `%r`은 `repr()`을 쓰는데, `str()`에는 없고 `repr()`에만
나오는 값(요청 주소·헤더를 든 예외)이 있으면 가림을 그대로 지나갔다(앞 PR들의 "남은 것"). 이 PR은 `str()`과 `repr()`을
따로 가린 자리(`_Redacted`)를 바꿔 넣어서 `%s`·`%r`·`format()` 어느 쪽으로 찍어도 가린 글이 나오게 한다.

| 값 | 결과 |
| --- | --- |
| 테스트 | **1556 passed** · DB 없이 1200 passed, 356 skipped |
| 되돌림 시험 | 돌연변이 **6개 전부 잡힘** |
| 검토 | 꼭 고칠 것 없음. repr만 깨진 값의 회귀 하나와 시험 둘을 고쳤다 |
| 마이그레이션 | 없음 |
| 파일 | 고친 파일 2개(`api/app/logging_redact.py`, `api/tests/test_security.py`). 새 파일 없음, web 안 바뀜 |

## 정한 것

1. **문자열이 아닌 인자는 `str()`과 `repr()`을 둘 다 본다.** 어느 쪽이든 가릴 것이 있으면 `_Redacted(가린 str, 가린 repr)`를
   그 자리에 넣는다. `__str__`·`__repr__`·`__format__`이 각각 가린 글을 내서 `%s`·`%r`·`%(이름)r`·`{}`·`{!r}` 모두 된다.
2. **가릴 것이 없으면 인자 객체를 그대로 둔다**(같은 객체). 인자를 직접 꺼내 쓰는 포매터(uvicorn 접근 로그의 str·int 인자)는
   전처럼 손대지 않는다. str·int·float·None은 전과 같다.
3. **`repr()`만 깨진 값은 `str()`로 찍은 것처럼 본다.** `str()`에 비밀이 있으면 전처럼 가린다(검토가 찾은 회귀를 막았다).
   `str()`이 깨진 값은 전처럼 그대로 둔다(부른 곳을 멈추지 않는다).
4. 서식 문자열에 민감한 말(`token`·`secret`…)이 있을 때 인자를 통째로 가리는 것(`mask_all`)은 전처럼 문자열 인자에만이다.
5. `repr()`을 매번 계산하는 비용은 받아들였다. 이 코드가 인자로 넘기는 것은 예외·UUID·주소·dataclass뿐이고, 로그 수준을
   지난 레코드에만 돈다. `UnicodeEncodeError`처럼 `repr()`이 입력을 통째로 담는 예외는 로그에 넘기지 않는다(storage.py가
   일부러 잇지 않는다).

## 레포에 넣는 법

````text
[0] #69가 main에 들어가 있어야 한다.
    git checkout main && git pull
    git checkout -b fix/log-redact-repr

[1] 새 파일은 없다.

[2] "패치" 절의 내용을 그대로 log-redact.patch로 저장해서 적용해 줘(고친 파일 2개).
    git apply log-redact.patch
    (적용한 뒤 log-redact.patch는 지워 줘)

[3] 명세·PRD·ERD는 바뀌지 않는다("문서에 반영하기" 없음).

[4] 이 문서도 docs/로그가림repr_구현.md로 같이 넣어 줘.

[5] 검사: api> uv run ruff check . && uv run ruff format --check . && uv run pytest
    DB 없이 `1200 passed, 356 skipped` 근처(네 PC는 보통 passed 하나 적고 skipped 하나 많다), CI에서는 `1556 passed`여야 한다.
    web은 바뀌지 않는다.

[6] 통과하면 "fix(security): 로그 가림이 %r로 찍은 인자의 repr()도 가린다"로 커밋해서 PR을 올려 줘.
````

## 합친 뒤 확인

````text
따로 없다. 서버를 켜고 평소처럼 쓰면 된다. 로그 모양이 바뀌는 것은 가릴 것이 있는 인자뿐이다.
````

## 구조

```
api/app/logging_redact.py   _Redacted(str·repr·format을 따로 가린 자리) · clean(): str()·repr()을 따로 try, 둘 다 깨끗하면 원래 객체
api/tests/test_security.py  repr에만 비밀이 있는 예외(%r·%s·%(이름)r·f-string·format) · 가릴 것 없는 인자는 같은 객체 · repr만 깨진 값
```

## 확인한 것

| 확인 | 결과 |
| --- | --- |
| 단위 | `__repr__`에만 서명 주소(`X-Amz-Credential=…`)가 든 예외를 `%r`로 찍으면 `[REDACTED]`, `%s`는 전처럼 "요청 실패", `%(error)r`(이름 붙인 인자)도 가림, 바꿔 넣은 자리의 `str()`·`repr()`·`f"{}"`·`format(…, ">6")`·`f"{!r}"`가 각각 맞다 · 가릴 것이 없는 `httpx.URL`·`ValueError`는 같은 객체(`is`)이고 `%r`이 전처럼 `ValueError('…')` · `__repr__`이 터지고 `__str__`에 Bearer 토큰이 있는 값은 `%s`·`%r` 둘 다 `Authorization: Bearer [REDACTED]` · 기존 시험(uvicorn 접근 로그 인자 5개·int 그대로, 주소 객체의 `apiKeyNm=` 가림, `__str__`이 터지는 값이 부른 곳을 멈추지 않음) 그대로 통과 |
| 로그 호출 조사(검토) | `LearningXError`·`GoogleCalendarError`·`StorageError`·`ApiError`는 고정 문구로 만들어져 `repr()`에 비밀이 없다. 가림 패턴이 못 잡는 비밀은 푸시 endpoint뿐인데 로그 인자로 넘기는 곳이 없다 |
| 테스트 | 1556 passed(DB 포함) · DB 없이 1200 passed, 356 skipped · ruff check·format 통과 |
| 문서만으로 다시 만들기 | 깨끗한 main(#69)에 이 문서의 패치만 적용하고 이 문서를 `docs/`에 넣은 트리를 작업 트리와 `diff`로 비교 → 같다. 그 트리로 다시: DB 포함 1556 passed · DB 없이 1200 passed, 356 skipped · ruff check·format 통과 |

## 되돌림 시험

| # | 고장 낸 것 | 잡은 시험 |
| --- | --- | --- |
| L1 | `repr()`을 보지 않는다(#69 그대로) | `test_a_secret_that_only_shows_in_repr_is_redacted` |
| L2 | 가릴 것이 없어도 자리를 바꿔 넣는다 | `test_an_argument_with_nothing_to_hide_is_passed_through_untouched` |
| L3 | 자리의 `repr()`이 `str` 쪽을 돌려준다 | `test_a_secret_that_only_shows_in_repr_is_redacted` |
| L4 | 자리의 `str()`이 `repr` 쪽을 돌려준다 | `test_a_secret_that_only_shows_in_repr_is_redacted` |
| L5 | `repr()`이 깨지면 통째로 지나간다 | `test_a_value_whose_repr_is_broken_is_still_redacted_by_str` |
| L6 | `format()`이 가리지 않은 값을 쓴다 | `test_a_secret_that_only_shows_in_repr_is_redacted` |

## 검토에서 고친 것

독립 검토(새 에이전트, 읽기만. C·D·E를 한 번에 봤다)에서 꼭 고칠 것은 없었다. `%s`·`%r`·`%a`·`%8s`·`%.4r`·`{}`·`{!r}`·`{:>12}`
모두 기대대로이고, f-string 로그는 인자 없이 `record.msg` 쪽이며, uvicorn 접근 로그 인자(str·int)에는 끼어들지 않고, `exc_info`
쪽은 바뀌지 않았다고 확인했다. 아래를 고친 뒤 시험·되돌림 시험을 다시 돌렸다.

| 본 것 | 고친 것 |
| --- | --- |
| **(좁은 회귀)** `str()`과 `repr()`을 한 try에 묶어서, `repr()`만 깨지고 `str()`에 비밀이 있는 값을 `%s`로 찍으면 전에는 가렸는데 이제 통째로 지나간다 | 둘을 따로 try 한다. `repr()`이 깨지면 `str()`로 찍은 것처럼 본다 + 시험(`test_a_value_whose_repr_is_broken_is_still_redacted_by_str`) |
| 시험의 `format(error)`는 로그 전에 원본 예외를 서식화한 것이라 `_Redacted.__format__`을 보지 않았다. 이름 붙인 인자(`%(e)r`) 경로도 시험이 없었다 | 바꿔 넣은 자리(`record.args[1]`)로 `f"{}"`·`format(…, ">6")`·`f"{!r}"`를 본다 + `%(error)r` 줄 |

그대로 둔 것

| 제안 | 판단 |
| --- | --- |
| `mask_all`(서식에 `token`·`secret` 같은 말)을 비문자열 인자에도 | 전과 같은 동작이라 두었다(남은 것) |
| `%s`로만 찍어도 `repr()`을 계산한다 — `UnicodeEncodeError`처럼 입력 전체가 `repr()`에 드는 예외는 커진다 | 이 코드는 그런 예외를 로그 인자로 넘기지 않고(storage.py가 일부러 잇지 않는다), 로그 수준을 지난 레코드에만 돈다. 받아들였다 |

## 남은 것

- `mask_all`(서식 문자열에 `token`·`secret` 같은 말이 있으면 인자를 통째로 가림)은 문자열 인자에만이다. 비문자열 인자는 패턴
  가림만 된다(전과 같다).
- 앞 PR의 남은 것(변수 이름 `SUPABASE_SECRET_KEY` 등)은 그대로다.

## 문서에 반영하기

````text
없다. 명세·PRD·ERD는 바뀌지 않는다.
````

## 코드

새 파일은 없다.

## 패치

고친 파일 2개다. 그대로 `log-redact.patch`로 저장해서 `git apply log-redact.patch`.

```diff
diff --git a/api/app/logging_redact.py b/api/app/logging_redact.py
index da47b79..194d599 100644
--- a/api/app/logging_redact.py
+++ b/api/app/logging_redact.py
@@ -4,7 +4,8 @@
 LearningX(Canvas) 개인 액세스 토큰, Fernet 암호문, Anthropic·Google API 키, Supabase secret key
 (`sb_secret_…`), 주소에 붙은 API 키(온통청년 apiKeyNm=… 등), 공지 첨부 주소의 열쇠(Canvas
 `verifier=…`, 파일 저장소 서명 `X-Amz-Signature=…`·`X-Amz-Credential=…`). 요청 주소처럼 문자열이
-아닌 인자도 문자열로 바꿔 보고 가릴 것이 있으면 가린다.
+아닌 인자도 `str()`·`repr()`로 바꿔 보고 가릴 것이 있으면 가린다 — 예외를 `%r`로 찍는 로그가 많고,
+`repr()`에만 나오는 값(요청 주소·헤더를 든 예외)도 있다.
 """
 
 import logging
@@ -56,6 +57,29 @@ def redact(text: str) -> str:
     return text
 
 
+class _Redacted:
+    """가릴 것이 있던 인자의 자리. `%s`로 찍든 `%r`로 찍든 가린 글이 나온다(원래 값은 들고 있지 않는다).
+
+    예외 인자는 보통 `%r`로 찍는데(`logger.warning("실패: %r", exc)`), 어떤 예외는 `str()`에는 없는
+    값이 `repr()`에 나온다(주소·헤더가 든 인자). 그래서 둘을 따로 가려 둔다.
+    """
+
+    __slots__ = ("_shown", "_text")
+
+    def __init__(self, text: str, shown: str) -> None:
+        self._text = text
+        self._shown = shown
+
+    def __str__(self) -> str:
+        return self._text
+
+    def __repr__(self) -> str:
+        return self._shown
+
+    def __format__(self, spec: str) -> str:
+        return format(self._text, spec)
+
+
 _installed = False
 
 
@@ -81,12 +105,18 @@ def install_log_redaction() -> None:
                     return _REDACTED if mask_all else redact(item)
                 if item is None or isinstance(item, int | float):
                     return item
-                try:
-                    text = str(item)  # httpx 요청 로그의 주소(httpx.URL) 같은 객체
+                try:  # httpx 요청 로그의 주소(httpx.URL) 같은 객체, `%r`로 찍는 예외
+                    text = str(item)
                 except Exception:
                     return item  # 문자열로 못 바꾸는 값은 로그 쪽이 알아서 처리한다(부른 곳을 멈추지 않는다)
-                cleaned = redact(text)
-                return cleaned if cleaned != text else item
+                try:
+                    shown = repr(item)
+                except Exception:
+                    shown = text  # repr만 깨진 값은 str로 찍은 것처럼 본다(가릴 것은 가린다)
+                cleaned, cleaned_shown = redact(text), redact(shown)
+                if cleaned == text and cleaned_shown == shown:
+                    return item
+                return _Redacted(cleaned, cleaned_shown)
 
             if isinstance(record.args, Mapping):
                 record.args = {key: clean(item) for key, item in record.args.items()}
diff --git a/api/tests/test_security.py b/api/tests/test_security.py
index 03cb8eb..3d704ca 100644
--- a/api/tests/test_security.py
+++ b/api/tests/test_security.py
@@ -109,6 +109,82 @@ def test_api_key_in_request_url_is_redacted(caplog: pytest.LogCaptureFixture) ->
     assert redact("rapikey=abc") == "rapikey=abc"  # 다른 낱말 속의 apikey는 아니다
 
 
+def test_a_secret_that_only_shows_in_repr_is_redacted(caplog: pytest.LogCaptureFixture) -> None:
+    """예외는 보통 `%r`로 찍는다. `str()`에는 없고 `repr()`에만 나오는 값(요청 주소를 든 예외)도 가린다."""
+    install_log_redaction()
+
+    class RequestFailed(Exception):
+        def __init__(self, url: str) -> None:
+            super().__init__("요청 실패")
+            self.url = url
+
+        def __repr__(self) -> str:
+            return f"RequestFailed(url={self.url!r})"
+
+    secret = "AKIAEXAMPLE0123"
+    error = RequestFailed(
+        f"https://s.example/f.pdf?X-Amz-Credential={secret}%2F2026&X-Amz-Expires=300"
+    )
+    logger = logging.getLogger("unipivot.test")
+    with caplog.at_level(logging.INFO, logger="unipivot.test"):
+        logger.warning("첨부 글자 추출 실패(%s): %r", "a.pdf", error)
+        logger.warning("같은 예외를 %%s로: %s", error)
+        logger.warning("이름 붙인 인자로: %(error)r", {"error": error})
+    assert secret not in caplog.text
+    assert (
+        "이름 붙인 인자로: RequestFailed(url='https://s.example/f.pdf?X-Amz-Credential=[REDACTED]"
+        in caplog.text
+    )
+    assert (
+        "RequestFailed(url='https://s.example/f.pdf?X-Amz-Credential=[REDACTED]&X-Amz-Expires=300')"
+        in caplog.text
+    )
+    assert "같은 예외를 %s로: 요청 실패" in caplog.text  # str()은 그대로다
+    record = caplog.records[0]
+    stand_in = record.args[1]
+    assert stand_in is not error  # 바꿔 넣은 자리다. str()·repr()·format()을 따로 낸다
+    assert str(stand_in) == "요청 실패"
+    assert "[REDACTED]" in repr(stand_in) and secret not in repr(stand_in)
+    assert f"{stand_in}" == "요청 실패" and format(stand_in, ">6") == " 요청 실패"
+    assert f"{stand_in!r}" == repr(stand_in)
+
+
+def test_an_argument_with_nothing_to_hide_is_passed_through_untouched(
+    caplog: pytest.LogCaptureFixture,
+) -> None:
+    """가릴 것이 없으면 인자 객체를 그대로 둔다(인자를 직접 꺼내 쓰는 포매터를 위해)."""
+    install_log_redaction()
+    url = httpx.URL("https://www.youthcenter.go.kr/go/ythip/getPlcy?pageNum=1")
+    error = ValueError("읽을 수 없는 값")
+    logger = logging.getLogger("unipivot.test")
+    with caplog.at_level(logging.INFO, logger="unipivot.test"):
+        logger.info("HTTP Request: %s %r", url, error)
+    assert caplog.records[0].args[0] is url
+    assert caplog.records[0].args[1] is error
+    assert "ValueError('읽을 수 없는 값')" in caplog.text
+
+
+def test_a_value_whose_repr_is_broken_is_still_redacted_by_str(
+    caplog: pytest.LogCaptureFixture,
+) -> None:
+    """repr()만 깨진 값은 str()로 찍은 것처럼 본다 — 전처럼 str()의 비밀은 가린다(회귀 방지)."""
+    install_log_redaction()
+
+    class BrokenRepr:
+        def __str__(self) -> str:
+            return f"Authorization: Bearer {JWT_LIKE}"
+
+        def __repr__(self) -> str:
+            raise RuntimeError("repr 없음")
+
+    logger = logging.getLogger("unipivot.test")
+    with caplog.at_level(logging.INFO, logger="unipivot.test"):
+        logger.info("값 %s", BrokenRepr())
+        logger.info("값 %r", BrokenRepr())
+    assert JWT_LIKE not in caplog.text
+    assert caplog.text.count("Authorization: Bearer [REDACTED]") == 2
+
+
 def test_log_argument_that_cannot_be_stringified_does_not_break_the_caller() -> None:
     """가릴 것을 찾으려고 인자를 문자열로 바꾸다 실패해도 로그를 부른 곳(요청 처리)은 멈추지 않는다."""
     install_log_redaction()
```
