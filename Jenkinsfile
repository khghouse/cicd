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
                        ssh -o StrictHostKeyChecking=no $REMOTE_HOST '
                            docker stop $CONTAINER_NAME || true &&
                            docker rm $CONTAINER_NAME || true &&
                            docker load < /tmp/docker-image.tar &&
                            docker run -d --name $CONTAINER_NAME -p 8080:8080 $IMAGE_NAME
                        '
                    """
                }
            }
        }
    }
}