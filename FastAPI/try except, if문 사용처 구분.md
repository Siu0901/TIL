# try/except vs if문 사용처 구분

이게 좀 햇갈리더라고. 갑자기 생각나서 찾아봄. 애매하게 알바에 확실히 정리하는게 낫지.

## 기준

> **내가 제어할 수 없는 외부 동작**이 실패할 수 있을 때 -> `try/except`  
> **결과를 코드로 예측/확인할 수 있을 때** -> `if` + `raise`

사실 저게 내용 다긴 함...


## if + raise — 비즈니스 로직 검증

결과를 실행 전에 코드로 확인할 수 있는 상황에서 쓴다.
if raise 쓰는 경우는 내가 만든 로직 등 이면 무슨 출력을 하고 어떨 때 오류가 나는지 알거 아녀? 이럴때 if + raise 쓴다.

```python
async def get_user(user_id: int):
    user = await repo.get_by_id(user_id)
    if not user:
        raise UserNotFoundException(user_id)  # 없으면 내가 명시적으로 던짐
    
    if user.id != current_user.id:
        raise PermissionDeniedException()     # 권한 체크도 마찬가지
    
    return user
```

- DB 조회 결과가 None인지 -> `if`로 확인 가능
- 권한이 있는지 -> `if`로 확인 가능
- **예측 가능한 상황이므로 try/except 불필요**함


## try/except — 외부 동작 실패 처리

이건 실행해봐야 성공/실패를 알 수 있는 상황에서 쓰는거임. 
외부 api나 파일 읽거나 뭐 저런 류들은 뭐 내가 컨트롤 할 수 있는게 아니잖음? 이럴때 try/except를 쓴다~~

```python
# 외부 API 호출
async def get_weather(city: str):
    try:
        response = await httpx.get(f"https://weather.api.com/{city}")
        response.raise_for_status()
        return response.json()
    except httpx.TimeoutException:
        raise WeatherAPITimeoutException()
    except httpx.HTTPStatusError:
        raise WeatherAPIException()

# 파일 I/O
async def read_config(path: str):
    try:
        with open(path, "r") as f:
            return json.load(f)
    except FileNotFoundError:
        raise ConfigNotFoundException(path)
    except json.JSONDecodeError:
        raise InvalidConfigException(path)

# 외부 SDK (S3, Redis 등)
async def upload_to_s3(file, key: str):
    try:
        await s3_client.put_object(Bucket="my-bucket", Key=key, Body=file)
    except ClientError as e:
        raise FileUploadException(key, reason=str(e))
```

**공통점**: 네트워크, 파일 시스템, 외부 SDK — 전부 내가 제어 못 하는 코드가 예외를 던짐.  
**목적**: 외부 예외를 내 `AppException`으로 변환하는 것. 예외를 삼키는 게 아님.

---

## DB 케이스 — 세션 의존성과 IntegrityError

보통 세션을 이렇게 의존성으로 주입한다

```python
async def get_session() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()   # 요청 정상 처리 후 여기서 commit
        except Exception:
            await session.rollback() # 예외 발생 시 rollback
            raise                    # 그대로 다시 위로 던짐
```

`commit()` 도중 `IntegrityError`가 발생하면:
1. `rollback()` 실행
2. `raise`로 그대로 다시 던짐
3. FastAPI 핸들러가 받아서 응답 반환

### 방법 1 — 핸들러에서 잡기 (단순할 때)

```python
app.add_exception_handler(IntegrityError, db_integrity_error_handler)
```

단점: unique 위반인지, FK 위반인지, not null 위반인지 구분이 안 된다.

### 방법 2 — flush()로 미리 유발 (명확하게 하고 싶을 때)

```python
async def create_user(self, session: AsyncSession, email: str):
    session.add(User(email=email))
    
    try:
        await session.flush()  # commit 전에 SQL만 날려서 에러 미리 유발
    except IntegrityError:
        raise UserAlreadyExistsException(email)  # 내 예외로 변환
```

`flush()`는 commit은 하지 않고 DB에 SQL만 실행해본다.  
어떤 제약 조건이 위반됐는지 파악해서 **의미 있는 내 예외로 변환**할 수 있다.  
이후 세션 의존성이 `rollback()`을 알아서 처리한다.

### 선택 기준

| 상황 | 방식 |
|---|---|
| 단순한 테이블, 에러 구분 불필요 | 핸들러만 등록 |
| 컬럼 많고 어떤 에러인지 메시지 줘야 함 | `flush()` + `except` |

---

## 정리

| 상황 | 패턴 |
|---|---|
| None 체크, 권한 체크 등 결과 확인 가능 | `if` → `raise` |
| 외부 API, 파일, SDK 등 실행 전 결과 모름 | `try/except` → `raise 내 예외` |
| DB commit 중 IntegrityError | 세션이 rollback 처리, 핸들러 또는 flush()로 변환 |
