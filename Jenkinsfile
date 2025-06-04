pipeline {
    agent any
    
    environment {
        PATH = "/opt/java/openjdk/bin:/usr/local/bin:/usr/bin:/bin:$PATH"
        DOCKER_PATH = "/usr/local/bin/docker"
        
        DOCKER_IMAGE = "calculator-app"
        DOCKER_TAG = "${BUILD_NUMBER}"
        CONTAINER_NAME = "calculator-container"
        APP_PORT = "8081"
        INTERNAL_PORT = "8080"
        
        // Health check settings
        MAX_WAIT_TIME = "180" // 3 minutes in seconds
        CHECK_INTERVAL = "10"  // 10 seconds between checks
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
                    echo "Building Docker image: ${DOCKER_IMAGE}:${DOCKER_TAG}"
                    sh """
                        docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                        docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest
                    """
                    echo "✅ Docker image built successfully"
                }
            }
        }
        
        stage('Stop Previous Container') {
            steps {
                script {
                    echo "Checking for existing container: ${CONTAINER_NAME}"
                    sh '''
                        # Stop running container if exists
                        if docker ps -q -f name=${CONTAINER_NAME} | grep -q .; then
                            echo "Stopping existing container..."
                            docker stop ${CONTAINER_NAME}
                        fi
                        
                        # Remove container if exists
                        if docker ps -aq -f name=${CONTAINER_NAME} | grep -q .; then
                            echo "Removing existing container..."
                            docker rm ${CONTAINER_NAME}
                        fi
                        
                        echo "✅ Previous container cleanup completed"
                    '''
                }
            }
        }
        
        stage('Deploy Container') {
            steps {
                script {
                    echo "Deploying new container: ${CONTAINER_NAME}"
                    sh """
                        docker run -d \
                            --name ${CONTAINER_NAME} \
                            -p ${APP_PORT}:${INTERNAL_PORT} \
                            --restart=unless-stopped \
                            ${DOCKER_IMAGE}:latest
                    """
                    echo "✅ Container deployed successfully"
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    echo "Starting deployment verification..."
                    sh '''
                        echo "Container status:"
                        docker ps | grep ${CONTAINER_NAME}
                        
                        echo "Waiting for application to start (up to ${MAX_WAIT_TIME} seconds)..."
                        COUNTER=0
                        MAX_ATTEMPTS=$((MAX_WAIT_TIME / CHECK_INTERVAL))
                        
                        while [ $COUNTER -lt $MAX_ATTEMPTS ]; do
                            ATTEMPT=$((COUNTER + 1))
                            echo "Health check attempt $ATTEMPT/$MAX_ATTEMPTS..."
                            
                            # Check if application responds
                            if curl -f -s --connect-timeout 5 --max-time 10 http://localhost:${APP_PORT}/ > /dev/null 2>&1; then
                                echo "✅ SUCCESS: Application is responding!"
                                
                                # Show application response preview
                                echo "Application response preview:"
                                curl -s --connect-timeout 5 --max-time 10 http://localhost:${APP_PORT}/ | head -10
                                
                                # Verify Java process is running
                                echo "Verifying Java process:"
                                if docker exec ${CONTAINER_NAME} ps aux | grep -q java; then
                                    echo "✅ Java process is running"
                                else
                                    echo "⚠️  Warning: Java process not found, but application is responding"
                                fi
                                
                                echo "✅ Deployment verification completed successfully!"
                                exit 0
                            fi
                            
                            echo "Application not ready yet, waiting ${CHECK_INTERVAL} seconds..."
                            sleep ${CHECK_INTERVAL}
                            COUNTER=$((COUNTER + 1))
                        done
                        
                        # If we get here, the application failed to start
                        echo "❌ FAILED: Application did not respond within ${MAX_WAIT_TIME} seconds"
                        echo "Container logs:"
                        docker logs --tail 50 ${CONTAINER_NAME}
                        echo "Container processes:"
                        docker exec ${CONTAINER_NAME} ps aux || echo "Could not check container processes"
                        echo "System resources:"
                        docker stats --no-stream ${CONTAINER_NAME} || echo "Could not get container stats"
                        exit 1
                    '''
                }
            }
        }
        
        stage('Post-Deploy Cleanup') {
            steps {
                script {
                    echo "Cleaning up old Docker images..."
                    sh '''
                        # Clean up old images (keep last 5 builds)
                        OLD_IMAGES=$(docker images ${DOCKER_IMAGE} --format "{{.Repository}}:{{.Tag}}" | \
                                   grep -v latest | tail -n +6)
                        
                        if [ -n "$OLD_IMAGES" ]; then
                            echo "Removing old images: $OLD_IMAGES"
                            echo "$OLD_IMAGES" | xargs -r docker rmi || echo "Some images could not be removed"
                        else
                            echo "No old images to clean up"
                        fi
                        
                        # Clean up dangling images
                        DANGLING_IMAGES=$(docker images -f "dangling=true" -q)
                        if [ -n "$DANGLING_IMAGES" ]; then
                            echo "Removing dangling images..."
                            echo "$DANGLING_IMAGES" | xargs -r docker rmi || echo "Some dangling images could not be removed"
                        fi
                    '''
                }
            }
        }
    }
    
    post {
        always {
            script {
                echo "Pipeline execution completed at: ${new Date()}"
                // Archive build artifacts if needed
                // archiveArtifacts artifacts: 'logs/*.log', allowEmptyArchive: true
            }
        }
        success {
            echo """
            🎉 SUCCESS: Calculator app deployed successfully!
            📍 Application URL: http://localhost:${APP_PORT}
            🐳 Container: ${CONTAINER_NAME}
            🏷️  Image: ${DOCKER_IMAGE}:${DOCKER_TAG}
            """
        }
        failure {
            script {
                echo """
                ❌ FAILURE: Pipeline failed!
                📋 Check the logs above for details.
                🐳 Container logs: docker logs ${CONTAINER_NAME}
                """
                
                // Optional: Send notification or cleanup on failure
                sh '''
                    echo "Failure cleanup - stopping failed container if running..."
                    docker stop ${CONTAINER_NAME} || true
                    docker logs --tail 20 ${CONTAINER_NAME} || true
                '''
            }
        }
        unstable {
            echo "⚠️  UNSTABLE: Pipeline completed with warnings"
        }
    }
}
