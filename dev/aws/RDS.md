# AWS RDS로 로컬 Docker DB 마이그레이션하기

음 로컬 Docker Compose로 운영하던 PostgreSQL을 AWS RDS로 이전했다.

---

## 왜 RDS로 옮김?

Docker Compose로 띄운 DB는 **내 컴퓨터에서만 접근 가능**하다.
배포 환경에서는 서버(ECS)와 DB를 분리해야 하는데, 그 이유는 대충 요러함.

- **데이터 안전성**: 컨테이너는 일회용이다. 죽으면 데이터 boom될 수 있음. RDS는 자동 백업과 장애 복구를 지원한다.
- **스케일링 방향이 다르다**: 서버는 수평 확장(인스턴스 추가)하지만, DB는 함부로 복제하면 데이터 정합성 문제가 생긴다.
- **독립적인 생명주기**: 서버는 자주 재배포하지만, DB는 한 번 세팅하면 거의 건드리지 않는다.

---

## RDS 생성 시 중요 설정 (돈 가장 안나오는 설정? 암튼 그럼)

| 항목 | 설정값 | 비고 |
|---|---|---|
| 엔진 | PostgreSQL | 로컬과 동일한 버전으로 맞출 것 |
| 인스턴스 | db.t4g.micro | 가장 저렴 (월 약 15달러) |
| 스토리지 | gp3, 20GB | 자동 조정은 체크 해제 |
| 퍼블릭 액세스 | 예 | 로컬에서 테스트하려면 필요. 배포 후 "아니오"로 변경 |
| 보안 그룹 | PostgreSQL 5432 포트 → 내 IP만 허용 | |

---

## 내 마이그레이션 과정 (내가 도커로 db띄워서 그걸로 쓰는 바람에 더 복잡함)

### 1. 로컬 Docker DB 덤프

```bash
# 컨테이너 안에서 덤프 파일 생성
docker exec intring-postgres pg_dump -U postgres -d Instring -f /tmp/backup.sql

# 컨테이너에서 로컬로 복사
docker cp intring-postgres:/tmp/backup.sql ./backup.sql
```

### 2. RDS에 빈 데이터베이스 생성

```bash
docker run -it --rm postgres:latest psql \
  -h [RDS엔포] -U postgres -d postgres -p 5432
```

```sql
CREATE DATABASE "Instring";
\q
```

> DB 이름이 대문자로 시작하면 반드시 쌍따옴표로 감싸야됨.

### 3. 덤프 복원

```bash
docker run -i --rm -v ${PWD}:/work postgres:latest psql \
  -h [RDS엔포] -U postgres -d Instring -f /work/backup.sql
```

### 4. 확인

```bash
docker run -it --rm postgres:latest psql \
  -h [RDS엔포] -U postgres -d Instring -p 5432
```

```sql
\dt  -- 테이블 목록 확인
```

---

## 하면서 생긴 문제들과 삽질들

### psql이 로컬에 없어서 실행 안 됨

Windows에 PostgreSQL 클라이언트가 설치되어 있지 않았다.
→ Docker로 해결:

```bash
docker run -it --rm postgres:latest psql -h [RDS엔드포인트] ...
```

### pg_dump 파일 인코딩 깨짐

```
ERROR: invalid byte sequence for encoding "UTF8": 0xff
```

Windows PowerShell에서 `>` 리다이렉트를 쓰면 파일이 **UTF-16**으로 저장된다.
PostgreSQL은 UTF-8만 읽을 수 있어서 에러 발생.

→ 컨테이너 안에서 직접 파일을 생성한 후 `docker cp`로 꺼내면 해결:

```bash
docker exec [컨테이너] pg_dump -U postgres -d [DB명] -f /tmp/backup.sql
docker cp [컨테이너]:/tmp/backup.sql ./backup.sql
```

### RDS 인스턴스 식별자 ≠ 데이터베이스 이름이더라

```
FATAL: database "instring-database" does not exist
```

`instring-database`는 RDS 인스턴스 이름이지, 내부 데이터베이스 이름이 아니라 함. 
RDS 생성 시 "초기 데이터베이스 이름"을 설정하지 않으면 기본 `postgres` DB만 존재한다함.
→ 직접 `CREATE DATABASE`로 생성해야 한다. ㅇㄴ 근데 난 분명 그 이름 적는 칸에 instring-database 썼던걸로 기억 하는데 씁.
나중에 세팅할 때 한번 더 봐보자.

---

## .env 파일 관련 혼란 정리

### docker-compose의 `${변수}` 와 `env_file`

```yaml
# 이건 docker-compose가 호스트의 .env에서 읽는 것임
environment:
  POSTGRES_USER: ${DB_USER}

# 이건 컨테이너 내부에 환경변수를 넣는 것
env_file:
  - .env.docker
```

둘은 **완전히 다른 메커니즘**이다.
`--env-file .env.docker` 옵션으로 통일할 수 있다.
이거 때문에 많이많이 고생했다. 자꾸 뭐 꼬이더라고. 
뭔가 나는 이런 경로 오류나 뭐 안맞는거에 많이 약한거 같다. 이런거도 뭐 따로 공부하는게 있나?
걍 어려움.


## Docker DB의 역할은 끝났다

RDS로 이전이 완료되면 로컬 Docker DB는 더 이상 필요 없다.

- Docker DB 컨테이너 → 삭제
- Docker DB 볼륨 → `docker volume rm`으로 삭제
- Redis만 Docker로 유지 → `docker run -d --name redis -p 6379:6379 redis:7`

---

## RDS 특징 정리

- 그냥 **일반 PostgreSQL**이다. pgAdmin에서 연결하면 로컬 DB 쓰듯이 테이블 조회/수정 가능하다.
- 테이블 수정하면 **즉시 반영**된다. 별도 배포 과정이 필요 없다.
- **비용 관리**: 안 쓸 때는 RDS 일시 중지 (최대 7일까지만 됨). 걍 7일마다 정지해라 그런 말임

---

## 현재 아키텍처

```
[로컬 개발]
  uvicorn (서버) → .env → RDS (PostgreSQL)
                        → localhost:6379 (Redis, Docker)

[배포 예정]
  ECS (서버 컨테이너) → RDS (PostgreSQL)
                      → ElastiCache (Redis)
```

---

## 다음 할거

- 서버를 EC2 또는 ECS에 Docker로 배포하기
- Redis를 ElastiCache로 이전하기
- CI/CD 파이프라인 구축하기

# 느낀거
RDS 뭔가 딱 생성할 때 뭔가 많긴 한데, 막 어렵진 않음. 그냥 뭐 얘네가 뭔가 애매하게 써놔서 뭘 말하는지 모르는 경우가 많을 뿐임. 근데 저 내 로컬 db에서 rds로 마이그레이션 하는 저 명령어들은 좀 숙지해 둬야겠다. 솔직히 도커 파일 만들고 뭐 그런것도 어떤식으로 하는지만 알고 막상 하려니 명령어나 세팅 기억 안나서 다 찾아서 쓰는데, 좀 왜워둬야겠음.
