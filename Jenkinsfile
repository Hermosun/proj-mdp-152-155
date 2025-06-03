pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = "calculator-app"
        DOCKER_TAG = "${BUILD_NUMBER}"
        CONTAINER_NAME = "calculator-container"
        APP_PORT = "8081"
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'project1', url: 'https://github.com/Hermosun/proj-mdp-152-155.git'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    // Build Docker image
                    sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
                    sh "docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest"
                }
            }
        }
        
        stage('Stop Previous Container') {
            steps {
                script {
                    // Stop and remove previous container if exists
                    sh '''
                        if [ $(docker ps -q -f name=${CONTAINER_NAME}) ]; then
                            docker stop ${CONTAINER_NAME}
                        fi
                        if [ $(docker ps -aq -f name=${CONTAINER_NAME}) ]; then
                            docker rm ${CONTAINER_NAME}
                        fi
                    '''
                }
            }
        }
        
        stage('Deploy Container') {
            steps {
                script {
                    // Run new container
                    sh """
                        docker run -d \
                        --name ${CONTAINER_NAME} \
                        -p ${APP_PORT}:8080 \
                        ${DOCKER_IMAGE}:latest
                    """
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    // Wait for application to start
                    sleep(30)
                    
                    // Check if container is running
                    sh "docker ps | grep ${CONTAINER_NAME}"
                    
                    // Optional: Health check
                    sh "curl -f http://localhost:${APP_PORT}/calculator/ || exit 1"
                }
            }
        }
    }
    
    post {
        always {
            // Clean up old images (keep last 5)
            sh '''
                docker images ${DOCKER_IMAGE} --format "table {{.Repository}}:{{.Tag}}" | \
                grep -v latest | tail -n +6 | xargs -r docker rmi || true
            '''
        }
        success {
            echo 'Pipeline succeeded! Calculator app deployed successfully.'
        }
        failure {
            echo 'Pipeline failed! Check the logs for details.'
        }
    }
}
