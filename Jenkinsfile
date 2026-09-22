pipeline {
    agent any

    environment {
        DOCKERHUB_USERNAME = 'jeni2w04'
        IMAGE_NAME = 'kanban-dashboard'
        CONTAINER_NAME = 'kanban-app'
        APP_PORT = '8080'
        HOST_PORT = '3000'
    }

    stages {

        stage('Checkout Source') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    env.IMAGE_TAG = "${BUILD_NUMBER}"
                    sh """
                        docker build -t ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG} .
                    """
                }
            }
        }

        stage('Docker Login and Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_TOKEN" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    docker rm -f ${CONTAINER_NAME} || true

                    docker run -d \
                      --name ${CONTAINER_NAME} \
                      --restart unless-stopped \
                      --memory="512m" \
                      --cpus="1.0" \
                      -p ${HOST_PORT}:${APP_PORT} \
                      ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                    echo "Waiting for application to become healthy..."

                    for i in 1 2 3 4 5 6 7 8 9 10
                    do
                        STATUS=$(docker inspect \
                          --format='{{.State.Health.Status}}' \
                          ${CONTAINER_NAME} 2>/dev/null || echo "starting")

                        echo "Health status: $STATUS"

                        if [ "$STATUS" = "healthy" ]; then
                            echo "Application is healthy."
                            curl --fail http://localhost:${HOST_PORT}/
                            exit 0
                        fi

                        if [ "$STATUS" = "unhealthy" ]; then
                            echo "Application is unhealthy."
                            docker logs ${CONTAINER_NAME}
                            exit 1
                        fi

                        sleep 5
                    done

                    echo "Health check timed out."
                    docker logs ${CONTAINER_NAME}
                    exit 1
                '''
            }
        }
    }

    post {
        success {
            echo "CI/CD pipeline completed successfully."
        }

        failure {
            echo "CI/CD pipeline failed."
            sh '''
                echo "Showing container status and logs..."
                docker ps -a || true
                docker logs ${CONTAINER_NAME} || true
            '''
        }

        always {
            sh 'docker logout || true'
        }
    }
}
