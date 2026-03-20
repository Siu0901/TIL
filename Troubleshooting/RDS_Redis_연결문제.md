# FastAPI + RDS + Redis 로컬 개발 트러블슈팅

## 당시 환경 (Instring 개발할 때 당시 환경)

- FastAPI 로컬 실행
- PostgreSQL: AWS RDS
- Redis: Docker 컨테이너

---

## 문제 있었던거 1 - RDS 연결 타임아웃

전에 RDS에 배포 후 정상 테스트 완료 → DB 인스턴스 중지했다 후에 재시작했는데 → 동일 설정인데 `Connection timed out` 발생함 

```
# 요딴식으로 떴음
psycopg2.OperationalError: connection to server at "xxx.rds.amazonaws.com", port 5432 failed: Connection timed out
```

### 원인

RDS 보안 그룹의 인바운드 규칙에 허용된 IP가 **이전 공인 IP**로 되어 있었음. 가정용 인터넷은 공인 IP가 수시로 바뀔 수 있기 때문에, DB를 껐다 켜는 사이에 내 IP가 변경된 것이였음.
원래는 보통 일정한거로 알고 있는데, 딱 생각해보니 이거 하기 바로 전에 뭐 공유기 할거 있어서 끄고 옮기고 다시 뭐 하는 등 일이 있었는데, 아마 그거 때문일 듯

### 어케 해결했나

AWS RDS 보안 그룹 인바운드 규칙에서 PostgreSQL 허용 IP를 **현재 내 공인 IP**로 다시 바꿈.

---

## 문제 있었던거 2 - Redis 연결 실패

RDS 문제 해결 후 로그인 API 호출 했는데 갑자기 Redis에서 에러 발생함.

```
redis.exceptions.ConnectionError: Error 11001 connecting to redis:6379. getaddrinfo failed.
```

### 원인

`.env`에 `REDIS_HOST=redis`로 설정되어 있었음. 찾아보니 `redis`라는 호스트명은 **Docker Compose 내부 네트워크**에서만 DNS가 해석되는 서비스명이라, Docker 밖에서 직접 실행하는 FastAPI 서버에서는 해당 이름을 찾을 수 없다함.
전에는 host=redis 했을 때 되서 왜지 했는데, 지금 보니 그떄 레디스랑 같이 도커 컴포즈로 감싸서 돌려서 저게 작동된듯. 지금은 서버 로컬로 하니 localhost가 맞음

### 그래서 해결법

`.env`에서 Redis 호스트를 `localhost`로 변경함.
---

# 잡생각
진짜 사실 개발자 오류의 고치기 힘든 90%는 이런 연결 문제 아닐까. 나만 봐도 가장 시간 많이 잡아먹은 오류들 보면 다 연결쪽 밖에 없더라.
뭐 코드는 내가 대충 해석해서 뭐 어떻게 짤 수는 있으니까 내가 해결이라도 하는데 이번 사례는 아니지만 연결쪽은 뭐 해결하려고 질문하면 기본으로 갑자기 듣도보도못한 터미널 명령어들 치고 이러니
기분이가 그럴수밖에.
