### 실습 목표

- AWS EC2 서버에 Docker 기반 Jenkins를 설치하고 설정한다.
- Jenkins를 활용해 Spring Boot 프로젝트를 자동 빌드, 배포하는 CI/CD 파이프라인을 구축한다.

<br />

### 1. Docker 설치

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
```

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

# Jenkins 사용 권한으로 되돌림
USER jenkins
```

### 3. 커스텀 Jenkins Docker 이미지 빌드

- Docker CLI가 포함된 젠킨스 커스텀 이미지 빌드
- 현재 디렉토리에 Dockerfile 필요

```shell
docker build -t custom-jenkins:lts-jdk17-docker .
```

### 4. Jenkins 컨테이너 실행

```shell
# 기존 Jenkins 컨테이너 중지 및 삭제
docker stop jenkins || true
docker rm jenkins || true

# jenkins_home 디렉토리 생성 및 퍼미션 설정
mkdir -p ~/jenkins_home

# Jenkins UID(1000)에 디렉토리 소유권 부여
sudo chown -R 1000:1000 ~/jenkins_home

# Jenkins 컨테이너 실행
docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v ~/jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /etc/group:/etc/group:ro \
  --restart unless-stopped \
  --group-add $(getent group docker | cut -d: -f3) \
  custom-jenkins:lts-jdk17-docker
```

### 5. Jenkins 접속 및 설정

#### 접속 URL

```text
http://{EC2_PUBLIC_IP}:8080
```

#### 초기 비밀번호 확인

```shell
cat /jenkins_home/secrets/initialAdminPassword
```

#### 플러그인 설치

- Suggested plugins 선택
- 추가 플러그인 설치 : Docker Pipeline, Docker Commons Plugin, Docker API Plugin

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

### 6. 프로젝트 내부에 Jenkinsfile, Dockerfile 생성

#### 디렉토리 구조

```text
/cicd
├── Dockerfile                  ✅
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
        IMAGE_NAME = "springboot-app-image"
        CONTAINER_NAME = "springboot-app"
    }

    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/khghouse/cicd.git'
            }
        }

        stage('Build JAR') {
            steps {
                sh './gradlew clean build'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${IMAGE_NAME} ."
            }
        }

        stage('Run Container') {
            steps {
                sh """
                docker rm -f ${CONTAINER_NAME} || true
                docker run -d --name ${CONTAINER_NAME} -p 8081:8080 ${IMAGE_NAME}
                """
            }
        }
    }
}
```

#### Dockerfile

```dockerfile
FROM amazoncorretto:17

ARG JAR_FILE=build/libs/cicd-0.0.1-SNAPSHOT.jar

COPY ${JAR_FILE} app.jar

ENTRYPOINT ["java","-jar","/app.jar"]
```

### (선택) 7. SwapFile 생성

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

### 8. 빌드 및 배포

- Jenkins 접속 -> Job 선택 -> 지금 빌드 버튼 클릭
    - 성공 시 콘솔에 Spring Boot 프로젝트 빌드 → Docker 이미지 생성 → 컨테이너 실행

<br />

#### 참고 자료

- ChatGPT 대화 내용
- https://velog.io/@chang626/AWS-EC2-free%EC%97%90%EC%84%9C-%EB%B0%9C%EC%83%9D%ED%95%9C-%EB%A9%94%EB%AA%A8%EB%A6%AC-%EB%AC%B8%EC%A0%9C-jenkins-build-%EB%B0%B0%ED%8F%AC
- https://okky.kr/articles/884329
