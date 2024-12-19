pipeline {
    agent any

    environment {
        REGISTRY = "docker.io"
        REGISTRY_CREDENTIALS = "dockerhub-credentials-id" // Docker Hub creds in Jenkins
        DOCKER_ORG = "maxmedov"
        IMAGE_PREFIX = "it-diagnostics-management-platform"
        KUBE_NAMESPACE = "it-diagnostics"
        KUBECONFIG_CREDENTIALS_ID = "kubeconfig-credentials-id" // Kubeconfig file credential
    }

    stages {
        stage('Checkout') {
            steps {
                // Clones the APP repo where code and Dockerfiles reside
                git branch: 'main', url: 'https://github.com/Max-Medov/IT-Diagnostics-Management-Platform.git'
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    sh """
                        #!/bin/sh
                        set -e
                        echo "Building Docker images..."

                        docker build -t ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:auth_service -f backend/auth_service/Dockerfile backend/auth_service
                        docker build -t ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:case_service -f backend/case_service/Dockerfile backend/case_service
                        docker build -t ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:diagnostic_service -f backend/diagnostic_service/Dockerfile backend/diagnostic_service
                        docker build -t ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:frontend -f frontend/Dockerfile frontend

                        echo "Docker images built successfully."
                    """
                }
            }
        }

        stage('Pre-Push Sanity Tests') {
            steps {
                script {
                    // Function to run container, perform test, and clean up
                    def runSanityTest = { containerName, image, containerPort, testUrl, errorMsg ->
                        sh """
                            #!/bin/sh
                            set -e

                            echo "Running sanity test for ${containerName}..."
                            docker run -d --name ${containerName} -p 0:${containerPort} ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:${image}
                            CONTAINER_PORT=\$(docker inspect --format='{{(index (index .NetworkSettings.Ports "${containerPort}/tcp") 0).HostPort}}' ${containerName})
                            
                            # Wait until the service is available
                            for i in \$(seq 1 10); do
                                if curl -f http://localhost:\$CONTAINER_PORT/; then
                                    break
                                fi
                                sleep 1
                            done

                            # Final check
                            curl -f http://localhost:\$CONTAINER_PORT/ || (echo "${errorMsg}" && docker rm -f ${containerName} && exit 1)
                            echo "Sanity test for ${containerName} passed."

                            # Clean up
                            docker rm -f ${containerName}
                        """
                    }

                    // Test auth_service image locally
                    runSanityTest('test_auth', 'auth_service', '5000', 'http://localhost:9000/', 'Auth service sanity test failed')

                    // Test case_service image locally
                    runSanityTest('test_case', 'case_service', '5001', 'http://localhost:9001/', 'Case service sanity test failed')

                    // Test diagnostic_service image locally
                    runSanityTest('test_diag', 'diagnostic_service', '5002', 'http://localhost:9002/', 'Diagnostic service sanity test failed')

                    // Test frontend image locally
                    runSanityTest('test_frontend', 'frontend', '3000', 'http://localhost:9003/', 'Frontend sanity test failed')
                }
            }
        }

        stage('Push Images to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${env.REGISTRY_CREDENTIALS}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        #!/bin/sh
                        set -e
                        echo "Logging into Docker Hub..."
                        echo ${DOCKER_PASS} | docker login ${REGISTRY} -u ${DOCKER_USER} --password-stdin

                        echo "Pushing Docker images to Docker Hub..."
                        docker push ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:auth_service
                        docker push ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:case_service
                        docker push ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:diagnostic_service
                        docker push ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:frontend

                        echo "Docker images pushed successfully."
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIALS_ID}", variable: 'KUBECONFIG_FILE')]) {
                    sh """
                        #!/bin/sh
                        set -e
                        echo "Using KUBECONFIG at \$KUBECONFIG_FILE"
                        export KUBECONFIG=\$KUBECONFIG_FILE

                        echo "Deploying to Kubernetes..."
                        kubectl apply -f kubernetes/namespace.yaml
                        kubectl apply -f kubernetes/secrets-configmap.yaml
                        kubectl apply -f kubernetes/postgres.yaml
                        kubectl apply -f kubernetes/auth-service.yaml
                        kubectl apply -f kubernetes/case-service.yaml
                        kubectl apply -f kubernetes/diagnostic-service.yaml
                        kubectl apply -f kubernetes/frontend.yaml
                        kubectl apply -f kubernetes/ingress.yaml

                        echo "Deployment to Kubernetes completed successfully."
                    """
                }
            }
        }

        stage('Wait for Pods') {
            steps {
                script {
                    sh """
                        #!/bin/sh
                        set -e
                        echo "Waiting for Kubernetes pods to be ready..."
                        kubectl rollout status deployment/auth-service -n ${KUBE_NAMESPACE} --timeout=180s
                        kubectl rollout status deployment/case-service -n ${KUBE_NAMESPACE} --timeout=180s
                        kubectl rollout status deployment/diagnostic-service -n ${KUBE_NAMESPACE} --timeout=180s
                        kubectl rollout status deployment/frontend -n ${KUBE_NAMESPACE} --timeout=180s

                        echo "Checking frontend availability via ingress..."
                        curl -f http://frontend.local || (echo "Frontend not responding after deploy" && exit 1)
                        echo "All pods are ready and frontend is available."
                    """
                }
            }
        }

        stage('Integration Tests') {
            steps {
                withCredentials([
                    usernamePassword(credentialsId: 'test-user-credentials', usernameVariable: 'TEST_USER', passwordVariable: 'TEST_PASS')
                ]) {
                    sh """
                        #!/bin/sh
                        set -e

                        echo "Registering a new user..."
                        curl -f -X POST -H "Content-Type: application/json" \\
                            -d '{"username": "${TEST_USER}", "password": "${TEST_PASS}"}' \\
                            http://auth.local/register || (echo "User registration failed" && exit 1)

                        echo "Logging in to obtain JWT token..."
                        TOKEN=\$(curl -f -X POST -H "Content-Type: application/json" \\
                                 -d '{"username":"${TEST_USER}","password":"${TEST_PASS}"}' \\
                                 http://auth.local/login | jq -r '.access_token')

                        if [ -z "\$TOKEN" ] || [ "\$TOKEN" = "null" ]; then
                          echo "Failed to obtain JWT token"
                          exit 1
                        fi

                        echo "Creating a case using the JWT token..."
                        curl -f -X POST -H "Content-Type: application/json" -H "Authorization: Bearer \$TOKEN" \\
                            -d '{"description": "Integration Test Case", "platform": "Linux Machine"}' \\
                            http://case.local/cases || (echo "Case creation failed" && exit 1)

                        echo "Verifying that the created case is listed..."
                        CASES=\$(curl -f -H "Authorization: Bearer \$TOKEN" http://case.local/cases)
                        echo "Received cases: \$CASES"
                        echo "\$CASES" | jq 'map(select(.description == "Integration Test Case"))' | grep "Integration Test Case" || (echo "Created case not found in case list" && exit 1)

                        echo "Checking diagnostic service endpoint..."
                        curl -f http://diagnostic.local/download_script/1 || (echo "Diagnostic service not responding" && exit 1)

                        echo "Integration tests completed successfully."
                    """
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully with pre-push sanity tests and integration tests!'
        }
        failure {
            echo 'Some stage failed. Check logs.'
        }
        always {
            cleanWs()
        }
    }
}
