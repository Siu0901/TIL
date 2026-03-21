# AWS ECS Fargate로 서버 배포하기

Dockerfile로 감싼 FastAPI 서버를 AWS ECS Fargate에 배포함.
이게 배포가 딱봐도 복잡하잖아. 도커로 하면 간단하다 하는데 이거 클로드 형 말대로 따박따박 했는데도 2시간 걸렸다. 
나중에 또 고생할까봐 배포 과정 딸깍으로 일단 정리해놨다. 
나와 대화한 걸 토대로 클로드가 대부분은 정리한거라 뭐 용어나 외에 내가 모르는거도 좀 있는데 애초에 이걸 처음 해본 입장에서 정리하는 거 자체가 좀 뭐시기하기도 하고
일단 느낌, 과정 등을 파악하고 자세한 것들은 차츰 알아나가야지.

---

## 배포 과정에서 대충 굵직한거 개념 정리

### ECS

AWS의 컨테이너 오케스트레이션 서비스. Docker 이미지를 실행하고 관리해준다.

### Fargate

ECS 위에서 동작하는 **서버리스 컨테이너 엔진**이다. EC2 인스턴스를 직접 만들고 관리할 필요 없이, "이 이미지를 실행해줘"만 하면 알아서 띄워준다. Fargate 없이 ECS를 쓰면 EC2 인스턴스 생성, Docker 설치, 클러스터 등록, 용량 관리 등 귀찮은거 전부 직접 해야 한다.

### ECR

AWS 전용 Docker 이미지 저장소다. Docker Hub의 AWS 버전이라고 생각하면 됨. 여기에 이미지를 올려야 ECS가 가져다 쓸 수 있다.

### 태스크 정의

"이 이미지를, 이 CPU/메모리로, 이 환경변수로, 이 포트로 실행해"라는 **명세서**다. JSON 파일로 작성한다.

### 서비스

태스크를 **몇 개 띄울지, 어떤 네트워크에서 실행할지** 등을 설정하는 단위다. 태스크가 죽으면 자동으로 다시 띄워준다.

---

## 내 전체 배포 흐름

```
1. AWS CLI 설치 & 설정
2. ECR에 이미지 저장소 생성
3. Docker 이미지 빌드 → ECR에 푸시
4. ECS 클러스터 생성
5. IAM 역할 생성 (ecsTaskExecutionRole)
6. 태스크 정의 등록
7. 보안 그룹 생성
8. ECS 서비스 생성
9. 퍼블릭 IP 확인 → 접속 테스트
```

---

## 상세 과정

### 1. AWS CLI 설치 & 설정

```bash
# 설치 후 계정 연결
aws configure

# AWS Access Key ID:     [액세스 키]
# AWS Secret Access Key: [시크릿 키]
# Default region name:   us-east-1
# Default output format: json
```

액세스 키는 AWS 콘솔 → 보안 자격 증명 → 액세스 키에서 발급한다.

### 2. ECR 저장소 생성

```bash
aws ecr create-repository --repository-name instring-server --region us-east-1
```

결과로 나오는 `repositoryUri`를 메모해둔다:
```
[계정id].dkr.ecr.us-east-1.amazonaws.com/instring-server
```

### 3. Docker 이미지 빌드 & ECR 푸시

```bash
# ECR 로그인
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin [계정id].dkr.ecr.us-east-1.amazonaws.com

# 이미지 빌드
docker build -t instring-server .

# ECR 주소로 태그
docker tag instring-server:latest [계정id].dkr.ecr.us-east-1.amazonaws.com/instring-server:latest

# 푸시
docker push [계정id].dkr.ecr.us-east-1.amazonaws.com/instring-server:latest
```

> 첫 푸시는 모든 레이어를 올려야 해서 오래 걸린다. 다음부터는 변경된 레이어만 올라가서 빠르다.

### 4. ECS 클러스터 생성

```bash
aws ecs create-cluster --cluster-name instring-cluster --region us-east-1
```

### 5. IAM 역할 생성

ECS가 ECR에서 이미지를 당겨오고 CloudWatch에 로그를 쓰려면 실행 역할이 필요하다.

```json
// trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

```bash
# 역할 생성
aws iam create-role --role-name ecsTaskExecutionRole --assume-role-policy-document file://trust-policy.json

# 권한 부여
aws iam attach-role-policy --role-name ecsTaskExecutionRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

### 6. 태스크 정의 등록

```json
// task-definition.json
{
  "family": "instring-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::[계정ID]:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "instring-server",
      "image": "[계정ID].dkr.ecr.us-east-1.amazonaws.com/instring-server:latest",
      "portMappings": [
        {
          "containerPort": 8000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {"name": "DB_HOST", "value": "[RDS 엔드포인트]"},
        {"name": "DB_PORT", "value": "5432"},
        {"name": "DB_USER", "value": "postgres"},
        {"name": "DB_PASSWORD", "value": "[비밀번호]"},
        {"name": "DB_NAME", "value": "Instring"},
        {"name": "REDIS_HOST", "value": "localhost"}
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/instring",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

```bash
# 로그 그룹 먼저 생성
aws logs create-log-group --log-group-name /ecs/instring --region us-east-1

# 태스크 정의 등록
aws ecs register-task-definition --cli-input-json file://task-definition.json --region us-east-1
```

### 7. 보안 그룹 생성

기본 보안 그룹은 외부에서 8000 포트 접근이 안 되므로, 서버용 보안 그룹을 별도 생성한다.

```bash
aws ec2 create-security-group --group-name instring-server-sg --description "ECS server security group" --region us-east-1

aws ec2 authorize-security-group-ingress --group-id [sg-ID] --protocol tcp --port 8000 --cidr 0.0.0.0/0 --region us-east-1
```

### 8. ECS 서비스 생성

```bash
aws ecs create-service \
  --cluster instring-cluster \
  --service-name instring-service \
  --task-definition instring-task \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx,subnet-yyy],securityGroups=[sg-xxx],assignPublicIp=ENABLED}" \
  --region us-east-1
```

- `subnets`: 기본 VPC의 서브넷 ID (최소 2개 권장)
- `securityGroups`: 위에서 만든 서버용 보안 그룹
- `assignPublicIp=ENABLED`: 외부 접근을 위한 퍼블릭 IP 할당

### 9. 퍼블릭 IP 확인 & 접속

```bash
# 태스크 ID 확인
aws ecs list-tasks --cluster instring-cluster --service-name instring-service --region us-east-1

# 네트워크 인터페이스 ID 확인
aws ecs describe-tasks --cluster instring-cluster --tasks [태스크ID] --query "tasks[0].attachments[0].details[?name=='networkInterfaceId'].value" --output text --region us-east-1

# 퍼블릭 IP 확인
aws ec2 describe-network-interfaces --network-interface-ids [eni-ID] --query "NetworkInterfaces[0].Association.PublicIp" --output text --region us-east-1
```

`http://[퍼블릭IP]:8000`으로 접속하여 확인.

---

## 재배포 방법

코드 수정 후 재배포는 세 줄이면 된다:

```bash
# 이미지 빌드
docker build -t instring-server .

# ECR에 태그 & 푸시
docker tag instring-server:latest [계정id].dkr.ecr.us-east-1.amazonaws.com/instring-server:latest
docker push [계정id].dkr.ecr.us-east-1.amazonaws.com/instring-server:latest

# ECS 업데이트
aws ecs update-service --cluster instring-cluster --service instring-service --force-new-deployment --region us-east-1
```

> ECR 로그인이 만료되었으면 푸시 전에 다시 로그인 필요.

---

## 트러블슈팅들

### RDS 보안 그룹 미설정으로 DB 연결 실패

```
sqlalchemy.exc.OperationalError: connection to server ... failed: Connection timed out
```

딱 서버 배포된거 확인하고 docs 들어가서 api 테스트 했는데 안되는거임!!! 뭐 host 문제도 없고 생각나는게 없어서 클로드 물어봤더니
RDS 보안 그룹에 내 IP만 허용해뒀고, ECS 서버의 보안 그룹은 허용하지 않은 거 같다 함.
→ RDS 보안 그룹에 ECS 보안 그룹으로부터의 5432 포트 인바운드를 추가했더니 해결:

```bash
aws ec2 authorize-security-group-ingress \
  --group-id [RDS보안그룹ID] \
  --protocol tcp --port 5432 \
  --source-group [ECS보안그룹ID] \
  --region us-east-1
```

### 프라이빗 IP로 접속 시도

`describe-tasks`에서 나오는 `privateIPv4Address`는 VPC 내부 주소라서 외부에서 접속 불가.
→ `networkInterfaceId`를 통해 **퍼블릭 IP**를 별도로 조회해야 한다.

### 재배포 시 IP가 바뀜

테스트 다 하고 껐다가 다시 켰는데 접속이 안되는 상황 발생함. 아니 바뀐게 없는데 왜 이러냐 하면서 또 물어봤는데
Fargate는 태스크가 재시작되면 **퍼블릭 IP가 바뀐다**함. 그래서 실행할때마다 명령어 쳐서 퍼블릭 ip 조회해서 그거로 들어가야됨.
→ 나중에 ALB(로드밸런서)를 붙이면 고정 주소를 사용할 수 있다 한다.

### AWS CLI 결과가 `-- More --`로 멈춤

AWS CLI가 긴 출력을 페이저로 보여주는 것이다. 첨에 뭐 더 있는 줄 알고 어물쩡거리다가 아닌거 같아서 물어보니 걍 나가면 되는거라더라.
→ `q`로 빠져나옴. `$env:AWS_PAGER=""`을 설정하면 이후 페이저 비활성화.

---

## 비용 참고

| 항목 | 스펙 | 예상 비용 |
|---|---|---|
| Fargate (256 CPU, 512MB) | 24시간 상시 운영 시 | 월 약 10~15달러 |
| RDS (db.t3.micro) | 24시간 상시 운영 시 | 월 약 15~20달러 |

> 안 쓸 때는 ECS 서비스의 `desired-count`를 0으로, RDS는 일시 중지하면 과금을 줄일 수 있다.

---

## 현재 아키텍처

```
사용자
  │
  ▼
[퍼블릭 IP]:8000  ← 재배포 시 변경됨
  │
  ▼
ECS Fargate (instring-server 컨테이너)
  │
  ├──→ RDS PostgreSQL (instring-database)
  └──→ Redis (미연결 — ElastiCache 예정)
```

---

## 다음에 할거

- [ ] ALB 연결 (고정 주소 + 도메인)
- [ ] ElastiCache Redis 생성 및 연결
- [ ] CI/CD 파이프라인 구축 (GitHub 푸시 → 자동 배포) 이건 좀 생각해보고(아직 저거 할만한 그런 엄 그런게 없음)
- [ ] RDS 퍼블릭 액세스 비활성화

# 생각
배포 과정을 알게 되었다 이거 하나로도 충분히 얻음. 이게 뭐 과정을 되게 뭐 많은데 결국 보면 
그냥 컴터 하나 빌려서 거따가 내가 띄울거 세팅하는데 뭐 ECS Fargate 이런거 하면 직접 ec2에서 다 설치해가며 할 필요 없게 하는거고 뭐 저장 장소 생성하고 보안 뭐 설정하고 그냥 이런거임.
뭐 아직 얘네들이 뭐하는 건지 정확히 모르고 명령어도 걍 찾아서 따라 치는 수준이긴 한데, 난 데브옵스는 아니므로 일단 과정 정도만 알아두고 차츰차츰 알아나가자. 
