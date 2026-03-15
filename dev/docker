# Docker 정리

## Docker란?

애플리케이션을 **컨테이너**라는 격리된 환경에서 실행할 수 있게 해주는 플랫폼.
핵심 개념은는 "내 앱이 돌아가는 데 필요한 모든 것을 하나의 상자(이미지)에 넣는다"는 것이다.

---

## 컨테이너 vs 가상머신

### 커널이란?

운영체제는 크게 두 부분으로 나뉜다.

- **커널**: 하드웨어(CPU, 메모리, 디스크, 네트워크)를 직접 제어하는 핵심 프로그램
- **유저 영역**: 파일 시스템, 시스템 라이브러리, 셸, 패키지 관리자 등

앱이 "파일을 저장해줘"라고 요청하면, 커널이 실제로 디스크에 데이터를 써준다.

### VM이 무거운 이유

- 하이퍼바이저가 가짜 하드웨어를 시뮬레이션
- 각 VM은 **커널 + 유저 영역을 통째로 설치**해야 함
- Node.js 앱 하나 돌리려고 Ubuntu 커널과 시스템 도구(수백 MB~수 GB)까지 올려야 함
- OS 부팅 과정도 거쳐야 해서 시작에 **수십 초~수 분** 소요

### 컨테이너가 가벼운 이유

- "커널은 이미 호스트에 있으니 또 설치할 필요 없다"
- 호스트 OS의 Linux 커널이 namespace와 cgroup으로 프로세스를 격리
- 컨테이너 안에는 **앱 + 최소한의 라이브러리만** 포함
- **수십 MB**, 시작 **수 초**

### 비유

- **VM** = 각 세대마다 자체 전기 발전기, 수도 펌프, 가스 배관을 독립 설치한 아파트
- **컨테이너** = 건물 인프라(커널)는 공유하고, 각 세대(컨테이너)는 내부 인테리어(앱+라이브러리)만 따로 꾸민 아파트

---

## Docker의 핵심 구성 요소

### Dockerfile

이미지를 만들기 위한 레시피. 텍스트 파일이며, 확장자 없이 `Dockerfile`이라는 이름으로 생성한다.

### Image

Dockerfile을 빌드한 결과물. 읽기 전용 레이어들이 쌓인 구조.
각 Dockerfile 명령이 하나의 레이어가 되고, 변경된 레이어만 다시 빌드하므로 캐싱이 효율적이다.

### Container

이미지 위에 쓰기 가능한 레이어를 얹어 실제로 실행한 인스턴스.
하나의 이미지에서 여러 개의 컨테이너를 동시에 띄울 수 있다.

### Docker Hub

앱스토어처럼 이미지를 저장하고 공유하는 원격 저장소.
PostgreSQL, Redis, MongoDB 등 대부분의 인프라 소프트웨어가 공식 이미지로 올라와 있다.

---

## Docker를 쓰는 이유 (AWS 배포 관점)

"내 컴퓨터에서 되는데?" 문제 해결이 핵심이 아니라, **서버 자체를 관리하는 고통**을 없애는 것이 핵심이다.

### 1. 서버가 죽었을 때

도커 없으면 새 EC2에 6단계 수동 설치를 처음부터 다시 해야 하지만, 도커가 있으면 `docker run` 한 줄로 복구 가능하다.

### 2. 서버를 여러 대로 확장할 때

수동 설치하면 서버마다 환경이 미묘하게 달라질 수 있지만, 같은 이미지를 사용하면 모든 서버가 100% 동일한 환경이 보장된다.

### 3. 롤백이 필요할 때

이미지에 버전 태그가 있어서 `docker run myapp:1.0` 한 줄로 이전 버전 복구가 가능하다.

### 4. AWS 자체가 도커 중심

ECS, Fargate, EKS 등 AWS 배포 서비스들이 도커 이미지를 기본 단위로 사용한다.

---

## 도커의 두 가지 사용법

### 내 코드를 감싸기 (docker build)

내가 Dockerfile을 작성하고 이미지를 빌드한다.

```bash
docker build -t myapp:1.0 .
```

### 남이 만든 소프트웨어 가져다 쓰기 (docker run)

Docker Hub에서 이미 만들어진 이미지를 다운받아 실행만 한다.

```bash
docker run postgres:16
docker run redis:7
```

> DB는 "감싸는" 단계가 없다. 해당 팀이 이미 이미지를 만들어 Docker Hub에 올려놨기 때문.

---

## Docker Desktop

Mac이나 Windows에서 도커를 쓸 수 있게 해주는 GUI 앱.
내부적으로 작은 Linux VM을 자동 생성하고, 그 안에서 도커 엔진을 돌린다.

- **Images 탭**: 빌드하거나 다운받은 이미지 목록
- **Containers 탭**: 실행 중이거나 멈춘 컨테이너 목록 (시작/중지 가능)
- **Volumes 탭**: 데이터 저장소 관리

---

## Dockerfile 작성법 (uv 기준)

```dockerfile
FROM python:3.13-slim

# uv 설치
COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/uv

WORKDIR /app

# psycopg2 빌드에 필요한 시스템 패키지
RUN apt-get update && apt-get install -y libpq-dev gcc && rm -rf /var/lib/apt/lists/

# 의존성 먼저 설치 (캐싱)
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

# 내 코드 전체 복사
COPY . .

EXPOSE 8000

# --reload는 개발용이므로 도커에서는 빼야 함
CMD ["uv", "run", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 빌드 과정 (`docker build -t myapp:1.0 .`)

1. 마지막의 `.` → 현재 폴더를 도커 엔진에게 전달 (빌드 컨텍스트)
2. `FROM` → 베이스 이미지 다운로드 (빈 Linux)
3. `WORKDIR` → 작업 폴더 생성
4. `COPY pyproject.toml uv.lock` → 의존성 파일만 먼저 복사
5. `RUN uv sync` → 라이브러리 설치
6. `COPY . .` → 내 코드 전체 복사
7. 이 상태를 스냅샷으로 저장 → **이미지 완성**

### 의존성을 먼저 복사하는 이유 (레이어 캐싱)

- 코드만 수정했을 때: 라이브러리 레이어는 캐시 사용 → **빌드 3~5초**
- 라이브러리 추가했을 때: uv sync 다시 실행 → **빌드 1~2분**

### .dockerignore

도커에 안 넣어도 되는 파일을 제외한다. `.gitignore`와 같은 개념.

```
.venv
.idea
.git
.env
__pycache__
*.pyc
temp
```

> `.env`를 제외하는 게 중요. 비밀번호 같은 민감한 정보가 이미지에 들어가면 안 된다.

---

## Docker Compose

여러 컨테이너를 한 파일에 정의하고 한 번에 실행하는 도구.

### docker-compose.yml 예시

```yaml
services:
  app:
    build: .
    container_name: intring-app
    ports:
      - "8000:8000"
    env_file:
      - .env
    depends_on:
      - db
      - redis

  db:
    image: postgres:16
    container_name: intring-postgres
    ports:
      - "5434:5432"
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: mydb
    volumes:
      - db-data:/var/lib/postgresql/data
    restart: unless-stopped

  redis:
    image: redis:7
    container_name: intring-redis
    ports:
      - "6379:6379"
    restart: unless-stopped

volumes:
  db-data:
```

### 주요 설정 설명

| 설정 | 의미 |
|------|------|
| `build: .` | Dockerfile로 이미지를 빌드 |
| `image: postgres:16` | Docker Hub에서 이미지 가져옴 |
| `ports: "5434:5432"` | 내 컴퓨터 5434 → 컨테이너 5432 연결 |
| `depends_on` | 지정한 서비스가 먼저 시작된 후 실행 |
| `volumes` | 데이터를 컨테이너 바깥에 보존 (삭제해도 데이터 유지) |
| `env_file` | 환경변수 파일 주입 |
| `restart: unless-stopped` | Docker Desktop 켜지면 자동 시작 |
| `container_name` | 컨테이너에 붙이는 별명 (안 적으면 자동 생성) |

### Compose 자동 네트워킹

Compose로 띄우면 각 컨테이너가 같은 네트워크에 연결되고, **서비스 이름이 곧 주소**가 된다.

```bash
# Compose 안에서 (컨테이너끼리 통신)
DATABASE_URL=postgresql://myuser:secret@db:5432/mydb

# Compose 밖에서 (내 PC에서 접속)
DATABASE_URL=postgresql://myuser:secret@localhost:5434/mydb
```

### 핵심 명령어

```bash
docker compose up -d           # 전부 시작
docker compose up -d --build   # 코드 수정 후 재빌드하면서 시작
docker compose down            # 전부 종료 (데이터 유지)
docker compose down -v         # 전부 종료 + 볼륨(DB 데이터)까지 삭제
docker compose logs -f app     # 특정 컨테이너 로그 보기
docker compose ps              # 실행 상태 확인
```

---

## 포트 충돌 주의

같은 포트를 두 프로그램이 동시에 쓸 수 없다.

- 로컬 PostgreSQL: **5432**
- pgvector (다른 프로젝트): **5433**
- 이번 프로젝트 PostgreSQL: **5434**
- Redis: **6379**

---

## 배포 시 구조

### 로컬 (개발)

docker-compose로 한 번에 관리. 전부 내 컴퓨터에서 실행.

### AWS (배포)

역할별로 분리.

- **백엔드** → ECS/Fargate (내 도커 이미지 실행)
- **PostgreSQL** → RDS (AWS 관리형 DB)
- **Redis** → ElastiCache (AWS 관리형 캐시)

코드는 안 바뀌고, `.env`의 접속 주소만 변경.

```bash
# 로컬
DATABASE_URL=postgresql://user:pass@localhost:5434/mydb

# 배포
DATABASE_URL=postgresql://user:pass@mydb.rds.amazonaws.com:5432/mydb
```

### 배포 순서

1. DB를 먼저 만든다 (RDS) → 주소 받음
2. 환경변수에 그 주소를 넣는다 (코드 수정 아님)
3. 백엔드 도커 이미지를 배포한다 (ECS)

### CI/CD 파이프라인

도커 때문에 자동 배포가 안 되는 게 아니라, 도커 덕분에 더 쉬워진다.

```
git push → CI가 docker build → docker push → AWS에서 새 이미지로 교체
```

---

## 트러블슈팅 메모

### psycopg2 빌드 에러

`python:3.13-slim`에는 C 빌드 도구가 없어서 psycopg2 설치가 실패한다.
Dockerfile에서 `uv sync` 전에 시스템 패키지를 설치해야 한다.

```dockerfile
RUN apt-get update && apt-get install -y libpq-dev gcc && rm -rf /var/lib/apt/lists/
```

### PostgreSQL 18 볼륨 에러

postgres:18은 데이터 저장 경로가 변경되어 기존 볼륨과 충돌한다.
postgres:16으로 변경하면 해결. (AWS RDS에서도 16이 가장 널리 쓰이는 안정 버전)

### 포트 충돌

`port is already allocated` 에러가 나면 해당 포트를 이미 다른 프로그램이 쓰고 있는 것.
다른 포트 번호로 변경하면 해결.
