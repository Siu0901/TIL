# ECS Fargate 도커 이미지 재배포 및 ECR 푸시 

로컬에서 수정한 코드를 Docker 이미지로 다시 빌드하고, AWS ECR에 푸시한 뒤 ECS(Fargate) 서비스에 재배포하는 작업을 진행했다.
솔직히 걍 aws 터미널 접속해서 명령어 딸깍딸깍하면 될줄 알았는데 좀 어이없는 실수들이 많아서 20분 날림. 걍 일단 써봄.
사실 그냥 재배포 과정 쓸거고 다시 제미나이 찾기 귀찮아서 걍 여따 쓴다.

---

## ECS + Fargate 무중단 재배포 과정

### 1. 로컬에서 도커 이미지 빌드

요로코롬 도커 이미지 빌드해줌
```bash
docker build -t instring-service:latest .
```

빌드가 완료되면 로컬 도커에 `[프로젝트명]:latest` 형태의 이미지가 생성된다.

---

### 2. AWS ECR 로그인

새 이미지를 ECR에 올리기 위해 로그인을 진행한다. (AWS CLI 필요한데 이미 해놨다)
```bash
# 주의: 본인의 진짜 ECR 리전을 정확히 입력해야 한다!
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  [AWS_계정ID].dkr.ecr.us-east-1.amazonaws.com
```
저거 리전 서울인줄 알고 계속 시도하다 끝에서 막혀서 왜인지 찾는데 고생했다. 리전 확인 잘해라.
---

### 3. 이미지 태그 변경

로컬 이미지에 ECR 주소 규격에 맞는 태그를 붙여준다.
```bash
docker tag instring-service:latest \
  [AWS_계정ID].dkr.ecr.us-east-1.amazonaws.com/instring-server:latest
```

---

### 4. ECR로 이미지 푸시
```bash
docker push [AWS_계정ID].dkr.ecr.us-east-1.amazonaws.com/instring-server:latest
```

---

### 5. ECS 서비스 업데이트 (재배포)

1. ECS 클러스터 -> 서비스(`instring-service`) 선택 → **[업데이트]**
2. 태스크 정의 개정을 방금 만든 **최신 버전**으로 선택
3. **[새 배포 강제 실행 (Force new deployment)]** 체크
4. 기존 태스크가 천천히 종료되고 새 태스크가 `RUNNING` 상태가 되면 배포된거임

---

# 데이터 큰일날뻔
그 도커 푸쉬하는데 하스팟 키고 하나가 도커 그 용량 보니까 4GB 인거임;; 중간에 바로 ctrl + C 누르긴 했는데 내 데이터 10% 날아감 tlqkf
