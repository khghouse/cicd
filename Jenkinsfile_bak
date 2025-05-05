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