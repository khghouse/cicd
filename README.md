### 실습 목표

- 두 대의 운영 서버에 헬스 체크를 기반으로 안전하게 순차 배포하는 CI/CD 환경을 구축한다.

<br />

### 0. 이전 작업

- 이전에는 단일 서버에 Jenkins를 이용하여 배포하는 구조였습니다.
    - https://github.com/khghouse/study/blob/main/practice/jenkins/3.%20jenkins-build-deploy-separated.md
- 이번 실습은 운영 서버 2대에 무중단/순차 배포를 목표로 확장된 환경을 구축합니다.

<br />

### 1. Docker & Docker Compose 설치

```shell
# Docker 설치
sudo yum update -y
sudo yum install -y docker
sudo systemctl start docker
sudo systemctl enable docker
docker --version

# 현재 사용자(ec2-user)를 docker 그룹에 추가
sudo usermod -aG docker ec2-user

# Docker Compose 설치
sudo curl -L "https://github.com/docker/compose/releases/download/v2.24.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# 현재 도커 그룹 ID 확인
export DOCKER_GID=$(getent group docker | cut -d: -f3)

# .env 파일 생성
echo "DOCKER_GID=$DOCKER_GID" > .env

# Jenkins 빌드 및 실행
docker-compose up -d --build

# 변경된 그룹 권한을 현재 셀에 반영
newgrp docker
```

<br />

### 2. Dockerfile 생성

```dockerfile
# 베이스 이미지 : 공식 Jenkins LTS + JDK 17
FROM jenkins/jenkins:lts-jdk17

# 패키지 설치를 위해 root 사용자로 변경
USER root

# 필요한 패키지 업데이트 및 설치
RUN apt-get update && apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg2 \
    lsb-release \
    software-properties-common

# Docker CLI 설치
RUN curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add - && \
    echo "deb https://download.docker.com/linux/debian $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list && \
    apt-get update && \
    apt-get install -y docker-ce-cli

# Docker Compose 설치
RUN curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" \
    -o /usr/local/bin/docker-compose \
 && chmod +x /usr/local/bin/docker-compose

# Jenkins 사용 권한으로 되돌림
USER jenkins
```

<br />

### 3. 커스텀 Jenkins Docker 이미지 빌드

- Docker CLI, Docker Compose가 포함된 젠킨스 커스텀 이미지 빌드
- 현재 디렉토리에 Dockerfile 필요

```shell
docker build -t custom-jenkins:lts-jdk17-docker .
```

<br />

### 4. docker-compose.yml

```yaml
version: '3.8'

services:
  jenkins:
    build:
      context: ..
    container_name: jenkins
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
      - /etc/group:/etc/group:ro
    environment:
      - TZ=Asia/Seoul
    restart: unless-stopped
    group_add:
      - "${DOCKER_GID:-998}"

volumes:
  jenkins_home:
```

Jenkins 빌드 및 실행

```shell
docker-compose up -d --build
```

<br />

### 5. Jenkins 접속 및 설정

#### 접속 URL

```text
http://{EC2_PUBLIC_IP}:8080
```

#### 초기 비밀번호 확인

```shell
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

#### 플러그인 설치

- Suggested plugins 선택
- 추가 플러그인 설치 : Docker Pipeline, Docker Commons Plugin, Docker API Plugin, SSH Agent Plugin

#### Job 생성

- 새로운 Item -> Pipeline
- 추가 설정 (Dashboard > {job_name} > Configuration > General)
    - GitHub project 체크
        - Project url : https://github.com/{계정아이디}/{리포지토리명}
    - Pipeline -> Definition -> Pipeline script from SCM 선택
        - SCM : Git
        - Repository URL : https://github.com/{계정아이디}/{리포지토리명}.git
        - Branch : master (또는 현재 사용중인 브랜치)
        - Script Path : Jenkinsfile (디폴트)

#### Credential 등록

Jenkins에서 운영 서버 접근을 위해 .pem 파일을 등록해야 합니다.

- Jenkins 관리 -> Credentials -> (global) -> Add Credentials
    - Kind : SSH Username with private key
    - Username : ec2-user (Amazon Linux2의 기본 사용자 이름)
    - ID : ec2-cicd-key (원하는 ID, Jenkinsfile에서 사용)
    - Enter directly 체크
        - .pem 파일 내용 복사

<br />

### 6. 프로젝트 내부에 Jenkinsfile, docker-compose.yml 생성 및 health-check

#### 디렉토리 구조

```text
/cicd
├── docker-compose-yml          ✅
├── Jenkinsfile                 ✅
├── src/
│   ├── main/
│   └── test/
```

#### Jenkinsfile

```groovy
pipeline {
    agent any

    environment {
        SERVER_1 = 'ec2-user@{ec2-ip-1}'
        SERVER_2 = 'ec2-user@{ec2-ip-2}'
        TARGET_DIR = '/home/ec2-user/app'
        IMAGE_NAME = "springboot-app-image"
        CONTAINER_NAME = "spring-app"
        PORT = "8080"
        HEALTH_ENDPOINT = "/actuator/health"
    }

    stages {
        stage('Clone Source') {
            steps {
                git url: 'https://github.com/khghouse/cicd.git', branch: 'master'
            }
        }

        stage('Build JAR') {
            steps {
                sh './gradlew clean build'
            }
        }

        stage('Docker Build & Save') {
            steps {
                sh """
                    docker build -t $IMAGE_NAME .
                    docker save $IMAGE_NAME > docker-image.tar
                """
            }
        }

        stage('Deploy to Server 1') {
            steps {
                sshagent(credentials: ['ec2-cicd-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no $SERVER_1 'mkdir -p $TARGET_DIR'
                        scp -o StrictHostKeyChecking=no docker-image.tar $SERVER_1:/tmp/
                        scp -o StrictHostKeyChecking=no docker-compose.yml $SERVER_1:$TARGET_DIR/

                        ssh -o StrictHostKeyChecking=no $SERVER_1 '
                            mkdir -p $TARGET_DIR &&
                            mv /tmp/docker-image.tar $TARGET_DIR &&
                            cd $TARGET_DIR &&
                            docker-compose down || true &&
                            docker load < docker-image.tar &&
                            docker-compose up -d
                        '
                    """

                    echo "Waiting for Server 1 to become healthy..."
                    retry(5) {
                        sleep(time: 5, unit: "SECONDS")
                        sh """
                            curl --fail http://${SERVER_1.split('@')[1]}:${PORT}${HEALTH_ENDPOINT} | grep '\"status\":\"UP\"'
                        """
                    }
                }
            }
        }

        stage('Deploy to Server 2') {
            steps {
                sshagent(credentials: ['ec2-cicd-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no $SERVER_2 'mkdir -p $TARGET_DIR'
                        scp -o StrictHostKeyChecking=no docker-image.tar $SERVER_2:/tmp/
                        scp -o StrictHostKeyChecking=no docker-compose.yml $SERVER_2:$TARGET_DIR/

                        ssh -o StrictHostKeyChecking=no $SERVER_2 '
                            mkdir -p $TARGET_DIR &&
                            mv /tmp/docker-image.tar $TARGET_DIR &&
                            cd $TARGET_DIR &&
                            docker-compose down || true &&
                            docker load < docker-image.tar &&
                            docker-compose up -d
                        '
                    """

                    echo "Waiting for Server 2 to become healthy..."
                    retry(5) {
                        sleep(time: 5, unit: "SECONDS")
                        sh """
                            curl --fail http://${SERVER_2.split('@')[1]}:${PORT}${HEALTH_ENDPOINT} | grep '\"status\":\"UP\"'
                        """
                    }
                }
            }
        }
    }
}
```

#### docker-compose.yml

```yaml
version: '3.8'

services:
  spring-app:
    image: springboot-app-image
    container_name: spring-app
    ports:
      - "8080:8080"
    restart: unless-stopped
```

#### health-check

```groovy
// build.gradle
implementation 'org.springframework.boot:spring-boot-starter-actuator'
```

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health
```

<br />

### 7. 운영 서버 Docker & Docker Compose 설치

```shell
# Docker 설치
sudo yum update -y
sudo yum install -y docker
sudo systemctl start docker
sudo systemctl enable docker
docker --version

# 현재 사용자(ec2-user)를 docker 그룹에 추가
sudo usermod -aG docker ec2-user

# 변경된 그룹 권한을 현재 셀에 반영
newgrp docker

# Docker Compose 설치
sudo curl -L "https://github.com/docker/compose/releases/download/v2.24.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```

<br />

### (선택) 8. SwapFile 생성

- EC2 메모리가 부족한 경우 스왑 메모리 추가
    - EC2(프리티어 버전)을 사용할 경우 메모리 부족 현상 발생

```shell
# 2GB 스왑 파일 생성 (128MB x 16)
sudo dd if=/dev/zero of=/swapfile bs=128M count=16

# 권한 설정
sudo chmod 600 /swapfile

# 스왑 영역 설정
sudo mkswap /swapfile

# 스왑 활성화
sudo swapon /swapfile

# 현재 스왑 상태 확인
sudo swapon -s
```

[추가 작업]

```shell
sudo vi /etc/fstab
```

```text
[insert] 마지막 라인에 추가
/swapfile swap swap defaults 0 0
```

<br />

### 9. 실습 요약

이번 실습을 통해 Jenkins를 활용하여 하나의 애플리케이션을 두 대의 운영 서버에 순차적으로 자동 배포하는 환경을 구축하였습니다.

<br />

### 관련 코드

- https://github.com/khghouse/cicd

<br />

#### 참고 자료

- ChatGPT 대화 내용
- https://velog.io/@chang626/AWS-EC2-free%EC%97%90%EC%84%9C-%EB%B0%9C%EC%83%9D%ED%95%9C-%EB%A9%94%EB%AA%A8%EB%A6%AC-%EB%AC%B8%EC%A0%9C-jenkins-build-%EB%B0%B0%ED%8F%AC
- https://okky.kr/articles/884329
