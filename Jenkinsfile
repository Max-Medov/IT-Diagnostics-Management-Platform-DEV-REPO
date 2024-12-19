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
                // Clones the APP repository
                git branch: 'main', url: 'https://github.com/Max-Medov/IT-Diagnostics-Management-Platform.git'
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    sh "docker build -t ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:auth_service -f backend/auth_service/Dockerfile backend"
                    sh "docker build -t ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:case_service -f backend/case_service/Dockerfile backend"
                    sh "docker build -t ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:diagnostic_service -f backend/diagnostic_service/Dockerfile backend"
                    sh "docker build -t ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:frontend -f frontend/Dockerfile frontend"
                }
            }
        }

        stage('Pre-Push Sanity Tests') {
            steps {
                script {
                    // Test auth_service
                    sh """
                    docker run -d --name test_auth -p 9000:5000 ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:auth_service
                    for i in \$(seq 1 3); do
                      sleep 10
                      if curl -f http://localhost:9000/; then
                        echo "Auth service sanity test passed"
                        break
                      else
                        echo "Auth service not ready yet, retrying..."
                      fi
                      if [ \$i -eq 3 ]; then
                        echo "Auth service failed after 3 retries"
                        docker rm -f test_auth
                        exit 1
                      fi
                    done
                    docker rm -f test_auth
                    """

                    // Test case_service
                    sh """
                    docker run -d --name test_case -p 9001:5001 ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:case_service
                    for i in \$(seq 1 3); do
                      sleep 10
                      if curl -f http://localhost:9001/; then
                        echo "Case service sanity test passed"
                        break
                      else
                        echo "Case service not ready yet, retrying..."
                      fi
                      if [ \$i -eq 3 ]; then
                        echo "Case service failed after 3 retries"
                        docker rm -f test_case
                        exit 1
                      fi
                    done
                    docker rm -f test_case
                    """

                    // Test diagnostic_service
                    sh """
                    docker run -d --name test_diag -p 9002:5002 ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:diagnostic_service
                    for i in \$(seq 1 3); do
                      sleep 10
                      if curl -f http://localhost:9002/; then
                        echo "Diagnostic service sanity test passed"
                        break
                      else
                        echo "Diagnostic service not ready yet, retrying..."
                      fi
                      if [ \$i -eq 3 ]; then
                        echo "Diagnostic service failed after 3 retries"
                        docker rm -f test_diag
                        exit 1
                      fi
                    done
                    docker rm -f test_diag
                    """

                    // Test frontend
                    sh """
                    docker run -d --name test_frontend -p 9003:3000 ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:frontend
                    for i in \$(seq 1 3); do
                      sleep 10
                      if curl -f http://localhost:9003/; then
                        echo "Frontend sanity test passed"
                        break
                      else
                        echo "Frontend not ready yet, retrying..."
                      fi
                      if [ \$i -eq 3 ]; then
                        echo "Frontend failed after 3 retries"
                        docker rm -f test_frontend
                        exit 1
                      fi
                    done
                    docker rm -f test_frontend
                    """
                }
            }
        }

        stage('Push Images to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${env.REGISTRY_CREDENTIALS}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh "echo ${DOCKER_PASS} | docker login ${REGISTRY} -u ${DOCKER_USER} --password-stdin"
                    sh "docker push ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:auth_service"
                    sh "docker push ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:case_service"
                    sh "docker push ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:diagnostic_service"
                    sh "docker push ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:frontend"
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIALS_ID}", variable: 'KUBECONFIG')]) {
                    // Ensure the kubeconfig has inline certs and no permission issues
                    sh 'kubectl apply -f kubernetes/namespace.yaml'
                    sh 'kubectl apply -f kubernetes/secrets-configmap.yaml'
                    sh 'kubectl apply -f kubernetes/postgres.yaml'
                    sh 'kubectl apply -f kubernetes/auth-service.yaml'
                    sh 'kubectl apply -f kubernetes/case-service.yaml'
                    sh 'kubectl apply -f kubernetes/diagnostic-service.yaml'
                    sh 'kubectl apply -f kubernetes/frontend.yaml'
                    sh 'kubectl apply -f kubernetes/ingress.yaml'
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
                    // Check frontend
                    sh 'curl -f http://frontend.local || (echo "Frontend not responding after deploy" && exit 1)'

                    // Register user
                    sh """
                    curl -f -X POST -H 'Content-Type: application/json' \
                        -d '{"username": "${TEST_USER}", "password": "${TEST_PASS}"}' \
                        http://auth.local/register || (echo "User registration failed" && exit 1)
                    """

                    // Login and get token
                    sh """
                    TOKEN=\$(curl -f -X POST -H 'Content-Type: application/json' -d '{"username":"${TEST_USER}","password":"${TEST_PASS}"}' http://auth.local/login | jq -r '.access_token')
                    if [ -z "\$TOKEN" ] || [ "\$TOKEN" = "null" ]; then
                      echo "Failed to obtain JWT token"
                      exit 1
                    fi
                    echo "TOKEN=\$TOKEN" > token_env.sh
                    """

                    sh '. ./token_env.sh'

                    // Create a case
                    sh """
                    curl -f -X POST -H 'Content-Type: application/json' -H "Authorization: Bearer \$TOKEN" \
                        -d '{"description": "Integration Test Case", "platform": "Linux Machine"}' \
                        http://case.local/cases || (echo "Case creation failed" && exit 1)
                    """

                    // Verify case
                    sh """
                    CASES=\$(curl -f -H "Authorization: Bearer \$TOKEN" http://case.local/cases)
                    echo "Received cases: \$CASES"
                    echo "\$CASES" | jq 'map(select(.description == "Integration Test Case"))' | grep "Integration Test Case" || (echo "Created case not found in case list" && exit 1)
                    """

                    // Diagnostic check
                    sh 'curl -f http://diagnostic.local/download_script/1 || (echo "Diagnostic service not responding" && exit 1)'
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
