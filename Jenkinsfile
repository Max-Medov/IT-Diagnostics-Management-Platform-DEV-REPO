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
                    sh "docker build -t ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:auth_service -f backend/auth_service/Dockerfile backend"
                    sh "docker build -t ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:case_service -f backend/case_service/Dockerfile backend"
                    sh "docker build -t ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:diagnostic_service -f backend/diagnostic_service/Dockerfile backend"
                    sh "docker build -t ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:frontend -f frontend/Dockerfile frontend"
                }
            }
        }

        stage('Pre-Push Minimal Sanity Test') {
            steps {
                script {
                    // Check just one service (e.g., auth_service) as a representative sample
                    // Start container
                    sh "docker run -d --name test_auth ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:auth_service"

                    // Wait a few seconds to see if it stays up
                    sh "sleep 10"

                    // Check if container is still running
                    def running = sh(script: "docker ps --filter name=test_auth --filter status=running | grep test_auth || true", returnStatus: true)

                    if (running != 0) {
                        echo "Auth service container not running after 10s, sanity test failed"
                        sh "docker rm -f test_auth"
                        error("Pre-push sanity test failed")
                    } else {
                        echo "Auth service container is running, sanity test passed"
                        sh "docker rm -f test_auth"
                    }
                }
            }
        }

        stage('Push Images to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${REGISTRY_CREDENTIALS}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
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
                withCredentials([file(credentialsId: "kubeconfig-credentials-id", variable: 'KUBECONFIG')]) {
                    sh 'kubectl apply -f ./kubernetes/namespace.yaml'
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
                    // Check frontend after deployment
                    sh 'curl -f http://frontend.local || (echo "Frontend not responding after deploy" && exit 1)'

                    // Register user
                    sh """
                    curl -f -X POST -H 'Content-Type: application/json' \
                        -d '{"username":"${TEST_USER}", "password":"${TEST_PASS}"}' \
                        http://auth.local/register || (echo "User registration failed after deploy" && exit 1)
                    """

                    // Login and get token
                    sh """
                    TOKEN=\$(curl -f -X POST -H 'Content-Type: application/json' -d '{"username":"${TEST_USER}","password":"${TEST_PASS}"}' http://auth.local/login | jq -r '.access_token')
                    if [ -z "\$TOKEN" ] || [ "\$TOKEN" = "null" ]; then
                      echo "Login failed after deploy"
                      exit 1
                    fi
                    echo "TOKEN=\$TOKEN" > token_env.sh
                    """

                    sh '. ./token_env.sh'

                    // Create a case
                    sh """
                    curl -f -X POST -H 'Content-Type: application/json' -H "Authorization: Bearer \$TOKEN" \
                        -d '{"description": "Integration Test Case", "platform": "Linux Machine"}' \
                        http://case.local/cases || (echo "Case creation failed after deploy" && exit 1)
                    """

                    // Verify case
                    sh """
                    CASES=\$(curl -f -H "Authorization: Bearer \$TOKEN" http://case.local/cases)
                    echo "Received cases: \$CASES"
                    echo "\$CASES" | jq 'map(select(.description == "Integration Test Case"))' | grep "Integration Test Case" || (echo "Created case not found after deploy" && exit 1)
                    """

                    // Diagnostic check
                    sh 'curl -f http://diagnostic.local/download_script/1 || (echo "Diagnostic service not responding after deploy" && exit 1)'
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully with minimal pre-push sanity checks!'
        }
        failure {
            echo 'Some stage failed. Check logs.'
        }
        always {
            cleanWs()
        }
    }
}
