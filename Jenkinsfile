pipeline {
    agent any

    environment {
        SERVER_1 = 'ec2-user@15.164.48.111'
        SERVER_2 = 'ec2-user@3.39.192.107'
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