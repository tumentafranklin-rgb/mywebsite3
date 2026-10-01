pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "tumenta3322/mywebsite3"
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'

                sh '''
                    docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .
                    docker tag ${DOCKER_IMAGE}:${BUILD_NUMBER} ${DOCKER_IMAGE}:latest
                '''
            }
        }

        stage('Test Docker Image') {
            steps {
                echo 'Testing Docker container...'

                sh '''
                    docker rm -f mywebsite3-test || true

                    docker run -d \
                        --name mywebsite3-test \
                        -p 8081:80 \
                        ${DOCKER_IMAGE}:${BUILD_NUMBER}

                    sleep 5

                    curl -f http://localhost:8081

                    docker rm -f mywebsite3-test
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                echo 'Pushing image to Docker Hub...'

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {

                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login \
                            --username "$DOCKER_USERNAME" \
                            --password-stdin

                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                        docker push ${DOCKER_IMAGE}:latest

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Azure') {
            steps {
                echo 'Deploying website to Azure...'

                sh '''
                    docker pull ${DOCKER_IMAGE}:latest

                    docker rm -f mywebsite3-container || true

                    docker run -d \
                        --name mywebsite3-container \
                        -p 80:80 \
                        --restart unless-stopped \
                        ${DOCKER_IMAGE}:latest

                    sleep 5

                    curl -f http://localhost
                '''
            }
        }
    }

    post {
        always {
            sh 'docker rm -f mywebsite3-test || true'
        }
    }
}
