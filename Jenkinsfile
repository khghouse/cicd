pipeline {
    agent any

    environment {
        REMOTE_HOST = 'ec2-user@3.35.149.79'
        TARGET_DIR = '/home/ec2-user/app'
        IMAGE_NAME = "springboot-app-image"
        CONTAINER_NAME = "spring-app"
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

        stage('Deploy') {
            steps {
                sshagent (credentials: ['ec2-cicd-key']) {
                    sh """
                        scp -o StrictHostKeyChecking=no docker-image.tar $REMOTE_HOST:/tmp/
                        scp -o StrictHostKeyChecking=no docker-compose.yml $REMOTE_HOST:$TARGET_DIR/

                        ssh -o StrictHostKeyChecking=no $REMOTE_HOST '
                            mkdir -p $TARGET_DIR &&
                            mv /tmp/docker-image.tar $TARGET_DIR &&
                            cd $TARGET_DIR &&
                            docker-compose down || true &&
                            docker load < docker-image.tar &&
                            docker-compose up -d
                        '
                    """
                }
            }
        }
    }
}