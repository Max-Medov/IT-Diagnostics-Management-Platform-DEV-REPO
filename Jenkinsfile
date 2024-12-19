pipeline {
    agent any

    environment {
        REGISTRY = "docker.io"
        REGISTRY_CREDENTIALS = "dockerhub-credentials-id" // Jenkins credentials for Docker Hub
        DOCKER_ORG = "maxmedov"
        IMAGE_PREFIX = "it-diagnostics-management-platform"
        KUBE_NAMESPACE = "it-diagnostics"
        KUBECONFIG_CREDENTIALS_ID = "kubeconfig-credentials-id" // Jenkins credential for kubeconfig
        TEST_USER = "testuser"
        TEST_PASS = "testpass"
    }

    stages {
        stage('Checkout') {
            steps {
                // Clones the APP repository containing the code and Dockerfiles
                git branch: 'main', url: 'https://github.com/Max-Medov/IT-Diagnostics-Management-Platform.git'
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    // Build images for all services and frontend
                    sh "docker build -t ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:auth_service -f backend/auth_service/Dockerfile backend"
                    sh "docker build -t ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:case_service -f backend/case_service/Dockerfile backend"
                    sh "docker build -t ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:diagnostic_service -f backend/diagnostic_service/Dockerfile backend"
                    sh "docker build -t ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:frontend -f frontend/Dockerfile frontend"
                }
            }
        }

        stage('Push Images to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${env.REGISTRY_CREDENTIALS}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    // Login to Docker registry
                    sh "echo ${DOCKER_PASS} | docker login ${REGISTRY} -u ${DOCKER_USER} --password-stdin"
                    
                    // Push all built images
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
                    // Apply Kubernetes manifests from the DEV repository (assumes Jenkinsfile and k8s/ are in DEV REPO)
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
                    // Wait for each service to roll out successfully
                    sh "kubectl rollout status deployment/auth-service -n ${KUBE_NAMESPACE} --timeout=180s"
                    sh "kubectl rollout status deployment/case-service -n ${KUBE_NAMESPACE} --timeout=180s"
                    sh "kubectl rollout status deployment/diagnostic-service -n ${KUBE_NAMESPACE} --timeout=180s"
                    sh "kubectl rollout status deployment/frontend -n ${KUBE_NAMESPACE} --timeout=180s"
                }
            }
        }

        stage('Integration Tests') {
            steps {
                script {
                    // Check frontend availability
                    sh 'curl -f http://frontend.local || (echo "Frontend not responding" && exit 1)'

                    // Register a new user
                    sh """
                    curl -f -X POST -H 'Content-Type: application/json' \
                        -d '{"username": "${TEST_USER}", "password": "${TEST_PASS}"}' \
                        http://auth.local/register || (echo "User registration failed" && exit 1)
                    """

                    // Login and get token
                    sh """
                    TOKEN=\$(curl -f -X POST -H 'Content-Type: application/json' \
                             -d '{"username": "${TEST_USER}", "password": "${TEST_PASS}"}' \
                             http://auth.local/login | jq -r '.access_token')

                    if [ -z "\$TOKEN" ] || [ "\$TOKEN" = "null" ]; then
                      echo "Failed to obtain JWT token"
                      exit 1
                    fi
                    echo "TOKEN=\$TOKEN" > token_env.sh
                    """

                    // Source the token into the current shell
                    sh '. ./token_env.sh'

                    // Create a case using the token
                    sh """
                    curl -f -X POST -H 'Content-Type: application/json' -H "Authorization: Bearer \$TOKEN" \
                        -d '{"description": "Integration Test Case", "platform": "Linux Machine"}' \
                        http://case.local/cases || (echo "Case creation failed" && exit 1)
                    """

                    // List cases to ensure the created case exists
                    sh """
                    CASES=\$(curl -f -H "Authorization: Bearer \$TOKEN" http://case.local/cases)
                    echo "Received cases: \$CASES"
                    echo "\$CASES" | jq 'map(select(.description == "Integration Test Case"))' | grep "Integration Test Case" || (echo "Created case not found in case list" && exit 1)
                    """

                    // Check diagnostic service endpoint (adjust endpoint if needed)
                    sh 'curl -f http://diagnostic.local/download_script/1 || (echo "Diagnostic service not responding" && exit 1)'
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully with inline curl-based integration tests!'
        }
        failure {
            echo 'Some stage failed. Check logs.'
        }
        always {
            cleanWs()
        }
    }
}
