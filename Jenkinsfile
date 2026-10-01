stage('Deploy to Azure') {
    steps {
        echo "Deploying build ${BUILD_NUMBER} to Azure..."

        sh '''
            docker pull ${DOCKER_IMAGE}:${BUILD_NUMBER}

            docker rm -f mywebsite3-container || true

            docker run -d \
                --name mywebsite3-container \
                -p 80:80 \
                --restart unless-stopped \
                ${DOCKER_IMAGE}:${BUILD_NUMBER}

            sleep 5

            curl -f http://localhost
        '''
    }
}
