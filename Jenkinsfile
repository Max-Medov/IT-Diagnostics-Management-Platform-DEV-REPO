pipeline {
    agent any

    environment {
        REGISTRY = "docker.io"
        REGISTRY_CREDENTIALS = "dockerhub-credentials-id"
        DOCKER_ORG = "maxmedov"
        IMAGE_PREFIX = "it-diagnostics-management-platform"
        KUBE_NAMESPACE = "it-diagnostics"
        KUBECONFIG_CREDENTIALS_ID = "kubeconfig-credentials-id"
        TEST_USER = "testuser"
        TEST_PASS = "testpass"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Max-Medov/IT-Diagnostics-Management-Platform.git'
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    sh "docker build -t ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:auth_service -f backend/auth_service/Dockerfile backend/auth_service"
                    sh "docker build -t ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:case_service -f backend/case_service/Dockerfile backend/case_service"
                    sh "docker build -t ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:diagnostic_service -f backend/diagnostic_service/Dockerfile backend/diagnostic_service"
                    sh "docker build -t ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:frontend -f frontend/Dockerfile frontend"
                }
            }
        }

        stage('Pre-Push Sanity Tests') {
            steps {
                script {
                    def runSanityCheck = { serviceName, imageTag, containerPort, testEndpoint ->
                        sh """
                        echo "Starting sanity check for ${serviceName}..."
                        
                        # Run the container
                        docker run -d --name test_${serviceName} -p 0:${containerPort} ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:${imageTag}
                        
                        # Get the dynamically assigned host port
                        CONTAINER_PORT=\$(docker inspect --format='{{(index (index .NetworkSettings.Ports "${containerPort}/tcp") 0).HostPort}}' test_${serviceName})
                        echo "Testing ${serviceName} on port \$CONTAINER_PORT"
                        
                        # Retry up to 10 times with a delay of 10 seconds
                        for i in \$(seq 1 10); do
                            if curl -f http://localhost:\$CONTAINER_PORT${testEndpoint}; then
                                echo "${serviceName} sanity test passed"
                                docker rm -f test_${serviceName}
                                exit 0
                            else
                                echo "${serviceName} not responding yet (attempt \$i), retrying in 10 seconds..."
                                sleep 10
                            fi
                        done
                        
                        # If we reach here, the service did not respond in time
                        echo "${serviceName} failed sanity test after 10 attempts"
                        docker logs test_${serviceName}
                        docker rm -f test_${serviceName}
                        exit 1
                        """
                    }

                    // Auth Service Sanity Check
                    runSanityCheck('auth_service', 'auth_service', 5000, '/health')

                    // Case Service Sanity Check
                    runSanityCheck('case_service', 'case_service', 5001, '/health')

                    // Diagnostic Service Sanity Check
                    runSanityCheck('diagnostic_service', 'diagnostic_service', 5002, '/health')

                    // Frontend Sanity Check
                    runSanityCheck('frontend', 'frontend', 3000, '/')
                }
            }
        }

        stage('Push Images to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${env.REGISTRY_CREDENTIALS}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                    echo ${DOCKER_PASS} | docker login ${REGISTRY} -u ${DOCKER_USER} --password-stdin
                    docker push ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:auth_service
                    docker push ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:case_service
                    docker push ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:diagnostic_service
                    docker push ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:frontend
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIALS_ID}", variable: 'KUBECONFIG')]) {
                    sh """
                    kubectl apply -f kubernetes/namespace.yaml
                    kubectl apply -f kubernetes/secrets-configmap.yaml
                    kubectl apply -f kubernetes/postgres.yaml
                    kubectl apply -f kubernetes/auth-service.yaml
                    kubectl apply -f kubernetes/case-service.yaml
                    kubectl apply -f kubernetes/diagnostic-service.yaml
                    kubectl apply -f kubernetes/frontend.yaml
                    kubectl apply -f kubernetes/ingress.yaml
                    """
                }
            }
        }

        stage('Wait for Pods') {
            steps {
                script {
                    sh """
                    kubectl rollout status deployment/auth-service -n ${KUBE_NAMESPACE} --timeout=180s
                    kubectl rollout status deployment/case-service -n ${KUBE_NAMESPACE} --timeout=180s
                    kubectl rollout status deployment/diagnostic-service -n ${KUBE_NAMESPACE} --timeout=180s
                    kubectl rollout status deployment/frontend -n ${KUBE_NAMESPACE} --timeout=180s
                    """
                }
            }
        }

        stage('Integration Tests') {
            steps {
                script {
                    sh """
                    # Frontend Availability
                    curl -f http://frontend.local || (echo "Frontend not responding after deploy" && exit 1)

                    # User Registration
                    curl -f -X POST -H "Content-Type: application/json" \
                        -d '{"username": "${TEST_USER}", "password": "${TEST_PASS}"}' \
                        http://auth.local/register || (echo "User registration failed" && exit 1)

                    # Login and Obtain Token
                    TOKEN=\$(curl -f -X POST -H "Content-Type: application/json" \
                        -d '{"username": "${TEST_USER}", "password": "${TEST_PASS}"}' \
                        http://auth.local/login | jq -r '.access_token')

                    if [ -z "\$TOKEN" ] || [ "\$TOKEN" = "null" ]; then
                        echo "Login failed"
                        exit 1
                    fi
                    echo "TOKEN=\$TOKEN"

                    # Create a Case
                    curl -f -X POST -H "Content-Type: application/json" -H "Authorization: Bearer \$TOKEN" \
                        -d '{"description": "Integration Test Case", "platform": "Linux Machine"}' \
                        http://case.local/cases || (echo "Case creation failed" && exit 1)

                    # Verify Case
                    CASES=\$(curl -f -H "Authorization: Bearer \$TOKEN" http://case.local/cases)
                    echo "Received cases: \$CASES"
                    echo "\$CASES" | jq 'map(select(.description == "Integration Test Case"))' | grep "Integration Test Case" || (echo "Case not found" && exit 1)

                    # Diagnostic Service Check
                    curl -f http://diagnostic.local/download_script/1 || (echo "Diagnostic service not responding" && exit 1)
                    """
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Some stage failed. Check logs.'
        }
        always {
            cleanWs()
        }
    }
}
