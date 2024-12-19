pipeline {
    agent any

    environment {
        REGISTRY = "docker.io"
        REGISTRY_CREDENTIALS = "dockerhub-credentials-id" // Docker Hub creds in Jenkins
        DOCKER_ORG = "maxmedov"
        IMAGE_PREFIX = "it-diagnostics-management-platform"
        KUBE_NAMESPACE = "it-diagnostics"
        KUBECONFIG_CREDENTIALS_ID = "kubeconfig-credentials-id" // Kubeconfig file credential
        TEST_USER = "testuser"
        TEST_PASS = "testpass"
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
                    // Test auth_service image locally
                    sh """
                    docker run -d --name test_auth -p 9000:5000 ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:auth_service
                    sleep 5
                    curl -f http://localhost:9000/ || (echo "Auth service sanity test failed" && docker rm -f test_auth && exit 1)
                    docker rm -f test_auth
                    """

                    // Test case_service image locally
                    sh """
                    docker run -d --name test_case -p 9001:5001 ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:case_service
                    sleep 5
                    curl -f http://localhost:9001/ || (echo "Case service sanity test failed" && docker rm -f test_case && exit 1)
                    docker rm -f test_case
                    """

                    // Test diagnostic_service image locally
                    sh """
                    docker run -d --name test_diag -p 9002:5002 ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:diagnostic_service
                    sleep 5
                    curl -f http://localhost:9002/ || (echo "Diagnostic service sanity test failed" && docker rm -f test_diag && exit 1)
                    docker rm -f test_diag
                    """

                    // Test frontend image locally
                    sh """
                    docker run -d --name test_frontend -p 9003:3000 ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:frontend
                    sleep 5
                    curl -f http://localhost:9003/ || (echo "Frontend sanity test failed" && docker rm -f test_frontend && exit 1)
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
                    // Check frontend availability via ingress
                    sh 'curl -f http://frontend.local || (echo "Frontend not responding after deploy" && exit 1)'

                    // Register user
                    sh """
                    curl -f -X POST -H 'Content-Type: application/json' \
                        -d '{"username": "${TEST_USER}", "password": "${TEST_PASS}"}' \
                        http://auth.local/register || (echo "User registration failed" && exit 1)
                    """

                    // Login and obtain token
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

                    // Verify case is listed
                    sh """
                    CASES=\$(curl -f -H "Authorization: Bearer \$TOKEN" http://case.local/cases)
                    echo "Received cases: \$CASES"
                    echo "\$CASES" | jq 'map(select(.description == "Integration Test Case"))' | grep "Integration Test Case" || (echo "Created case not found in case list" && exit 1)
                    """

                    // Check diagnostic service endpoint (adjust if needed)
                    sh 'curl -f http://diagnostic.local/download_script/1 || (echo "Diagnostic service not responding" && exit 1)'
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
