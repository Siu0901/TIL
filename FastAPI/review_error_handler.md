# FastAPI 에러 핸들러

이제와서 왜 이걸 하고있냐? 솔직히 당시 내가 쓸 일 잘 없다 생각해 그냥 대충 알고 넘어가서 머리속에 제대로된 내용이 없더라. 이번에 다시 숙지 겸 공부함.

## exception_handler 등록 문법

```python
# 데코레이터 방식
@app.exception_handler(AppException)
async def app_exception_handler(request, exc):
    ...

# 함수 등록 방식 (위에거랑 동일한거임)
app.add_exception_handler(AppException, app_exception_handler)
```

FastAPI 내부에서 `{예외클래스: 핸들러함수}` 딕셔너리를 관리한다.  
예외가 발생하면 **상속 순서** 를 따라 가장 구체적인 핸들러부터 탐색한다.  
`UserNotFoundException → AppException → Exception` 요런 순으로 올라가며 등록된 핸들러를 찾는다.


## 핸들러 함수 시그니처

```python
async def app_exception_handler(request: Request, exc: AppException) -> JSONResponse:
```

항상 저거 볼떄마다 request 안쓰는대 걍 빼면 안되나 했는데
FastAPI가 핸들러 호출 시 **무조건 `(request, exc)` 두 개를 인자로 넘긴다.** 그냥 규칙임.

### request는 왜 있냐

안 쓸 때도 있지만 로깅에 유용하다 함

```python
async def app_exception_handler(request: Request, exc: AppException):
    path      = request.url.path    # "/users/3"
    method    = request.method      # "GET"
    client_ip = request.client.host # "192.168.0.1"
    
    logger.error(f"[{method}] {path} - {exc.error_code}")
    
    return JSONResponse(...)
```
고로 뭐 로그 관련해서 할 때 쓰인다~~


## 왜 예외마다 핸들러를 따로 등록하냐

이거 궁금했던게 커스텀 exception이거에 다 넣으면 되는데 뭐 벨류에러 integrity에러 이런거는 따로 설정하더라고
함 알아보니 단순함.

`RequestValidationError`, `IntegrityError`는 **내가 만든 클래스가 아니다!!!**

```
AppException              ← 내가 만든 거
    └── UserNotFoundException
    └── BookNotFoundException

RequestValidationError    ← FastAPI/Pydantic이 만든 거
IntegrityError            ← SQLAlchemy가 만든 거
StarletteHTTPException    ← Starlette이 만든 거
```

그냥 상속 관계가 없으면 `AppException` 핸들러로 잡히지 않음.  
그래서 각각 따로 등록해야 한다

```python
app.add_exception_handler(AppException,          app_exception_handler)
app.add_exception_handler(StarletteHTTPException, http_exception_handler)
app.add_exception_handler(RequestValidationError, validation_exception_handler)
app.add_exception_handler(IntegrityError,         db_exception_handler)
```

억지로 AppException으로 통일하고 싶다면 **직접 감싸는** 방법이 있긴 함

```python
try:
    await db.commit()
except IntegrityError:
    raise DuplicateDataException()  # AppException 상속한 내 예외로 변환
```


## RequestValidationError의 exc.errors()와 detail 가공

좀 많이 많이 궁금했던거임.

저거 에러 핸들러가 보면 

```python
@app.exception_handler(RequestValidationError)
    def validation_exception_handler(request: Request, exc: RequestValidationError):
        spec = ErrorCodes.VALIDATION_ERROR
        errors = [
            {
                "field": " → ".join(str(loc) for loc in e["loc"]),
                "detail": e["msg"],
            }
            for e in exc.errors()
        ]
        return JSONResponse(...)
```
중간에 {} 안에 갑자기 좀 상당히 더럽게 생긴 코드가 나오는데 저 안에 대체 뭐가 들어있길래 저런식으로 .join하고 for문 돌리고 해서 출력하나 궁금했음.


우선 Pydantic 유효성 검사 실패 시 `exc.errors()`가 필드별 실패 정보를 리스트로 반환함.

`{"email": "not-an-email", "age": "abc"}` 요청이 들어오면:

```python
exc.errors()
# [
#     {"loc": ("body", "email"), "msg": "value is not a valid email", "type": "value_error.email"},
#     {"loc": ("body", "age"),   "msg": "value is not a valid integer", "type": "type_error.integer"}
# ]
```

이걸 가공해서 응답에 담는 코드가 아까 봤던 그거임.

```python
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    errors = exc.errors()
    
    return JSONResponse(
        status_code=422,
        content={
            "success": False,
            "error_code": "VALIDATION_ERROR",
            "message": "Request validation failed",
            "detail": [
                {
                    "field": " → ".join(str(loc) for loc in err["loc"]),
                    # ("body", "email") → "body → email"
                    "message": err["msg"],
                    "type": err["type"],
                }
                for err in errors
            ],
        }
    )
```

`loc`이 튜플이라 `.join()`에 쓰려면 `str()`로 변환이 필요하다.


## AppException의 detail 파라미터

```python
class AppException(Exception):
    def __init__(self, status_code, error_code, message, detail=None):
```

`message`는 사람이 읽는 간단한 설명.  
`detail`은 클라이언트가 분기 처리할 추가 구조화 데이터가 필요할 때 채운다 함.

```python
# 대부분은 detail 없이 충분
raise UserNotFoundException(user_id=3)
# → message: "User 3 does not exist", detail: None

# 추가 정보가 필요한 경우
class FileUploadException(AppException):
    def __init__(self, filename: str, reason: str):
        super().__init__(
            status_code=400,
            error_code="FILE_UPLOAD_FAILED",
            message="File upload failed",
            detail={
                "filename": filename,
                "reason": reason,
                "allowed_extensions": [".jpg", ".png", ".pdf"]  # 프론트가 꺼내 쓸 수 있음
            }
        )
```

응답
```json
{
  "error_code": "FILE_UPLOAD_FAILED",
  "message": "File upload failed",
  "detail": {
    "filename": "photo.exe",
    "reason": "extension not allowed",
    "allowed_extensions": [".jpg", ".png", ".pdf"]
  }
}
```


## 핸들러 등록 전체 구조

```python
# core/exceptions/handlers.py — 핸들러 함수 정의
# core/exceptions/__init__.py — 일괄 등록 함수

def register_exception_handlers(app: FastAPI) -> None:
    app.add_exception_handler(AppException,          app_exception_handler)
    app.add_exception_handler(StarletteHTTPException, http_exception_handler)
    app.add_exception_handler(RequestValidationError, validation_exception_handler)
    app.add_exception_handler(Exception,              unhandled_exception_handler)

# main.py
register_exception_handlers(app)
```

## 정리

| 궁금증 | 답 |
|---|---|
| `@app.exception_handler` | 예외 타입과 핸들러 함수를 연결하는 등록 문법 |
| 왜 각각 따로 등록? | 상속 관계 없으면 못 잡음. 내 클래스 아니면 따로 필수 |
| `request` 파라미터 | FastAPI 규칙. 경로/IP 추출해서 로깅에 활용 |
| `exc.errors()` + for문 | Pydantic이 담아준 필드별 실패 정보를 가공해서 응답에 담는 것 |
| `detail=None` | 추가 구조화 데이터가 필요할 때만 채움. 대부분은 None |
