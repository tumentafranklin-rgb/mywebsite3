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
                echo "Deploying build ${BUILD_NUMBER} to Azure..."

                sh '''
                    set -e

                    echo "======================================"
                    echo "Finding currently running version..."
                    echo "======================================"

                    PREVIOUS_IMAGE=$(docker inspect \
                        --format='{{.Config.Image}}' \
                        mywebsite3-container 2>/dev/null || true)

                    echo "Previous image: ${PREVIOUS_IMAGE}"

                    echo "======================================"
                    echo "Pulling new image..."
                    echo "======================================"

                    docker pull ${DOCKER_IMAGE}:${BUILD_NUMBER}

                    echo "======================================"
                    echo "Stopping current website..."
                    echo "======================================"

                    docker rm -f mywebsite3-container || true

                    echo "======================================"
                    echo "Starting new website..."
                    echo "======================================"

                    docker run -d \
                        --name mywebsite3-container \
                        -p 80:80 \
                        --restart unless-stopped \
                        ${DOCKER_IMAGE}:${BUILD_NUMBER}

                    echo "Waiting for website..."
                    sleep 5

                    echo "======================================"
                    echo "Running health check..."
                    echo "======================================"

                    if curl -f http://localhost; then

                        echo "======================================"
                        echo "DEPLOYMENT SUCCESSFUL"
                        echo "Build ${BUILD_NUMBER} is healthy."
                        echo "======================================"

                    else

                        echo "======================================"
                        echo "DEPLOYMENT FAILED"
                        echo "Starting rollback..."
                        echo "======================================"

                        docker rm -f mywebsite3-container || true

                        if [ -n "$PREVIOUS_IMAGE" ]; then

                            echo "Restoring previous image:"
                            echo "${PREVIOUS_IMAGE}"

                            docker run -d \
                                --name mywebsite3-container \
                                -p 80:80 \
                                --restart unless-stopped \
                                ${PREVIOUS_IMAGE}

                            echo "Waiting for rollback..."
                            sleep 5

                            echo "Checking restored website..."

                            if curl -f http://localhost; then

                                echo "======================================"
                                echo "ROLLBACK SUCCESSFUL"
                                echo "Previous version restored."
                                echo "======================================"

                            else

                                echo "======================================"
                                echo "ROLLBACK FAILED"
                                echo "======================================"

                                exit 1
                            fi

                        else

                            echo "======================================"
                            echo "NO PREVIOUS IMAGE FOUND"
                            echo "Cannot perform rollback."
                            echo "======================================"

                            exit 1
                        fi

                        exit 1
                    fi
                '''
            }
        }

        stage('Docker Cleanup') {
            steps {
                echo 'Cleaning up old Docker images...'

                sh '''
                    echo "======================================"
                    echo "DOCKER IMAGE CLEANUP"
                    echo "======================================"

                    CURRENT_IMAGE=$(docker inspect \
                        --format='{{.Config.Image}}' \
                        mywebsite3-container)

                    echo "Current production image:"
                    echo "$CURRENT_IMAGE"

                    echo ""
                    echo "Images before cleanup:"
                    docker images ${DOCKER_IMAGE}

                    echo ""
                    echo "Removing unused Docker images..."

                    docker image prune -f

                    echo ""
                    echo "Images after cleanup:"
                    docker images ${DOCKER_IMAGE}

                    echo ""
                    echo "======================================"
                    echo "CLEANUP COMPLETED"
                    echo "======================================"
                '''
            }
        }
    }

    post {

        always {
            echo 'Cleaning up test container...'

            sh '''
                docker rm -f mywebsite3-test || true
            '''
        }

        success {
            echo '======================================'
            echo 'PIPELINE COMPLETED SUCCESSFULLY'
            echo '======================================'
        }

        failure {
            echo '======================================'
            echo 'PIPELINE FAILED'
            echo 'Check the Jenkins console log.'
            echo '======================================'
        }
    }
}
