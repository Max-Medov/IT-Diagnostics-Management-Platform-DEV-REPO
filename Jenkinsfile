pipeline {
    agent any

    environment {
        REGISTRY = "docker.io"
        REGISTRY_CREDENTIALS = "dockerhub-credentials-id" // Docker Hub creds in Jenkins
        DOCKER_ORG = "maxmedov"
        IMAGE_PREFIX = "it-diagnostics-management-platform"
        KUBE_NAMESPACE = "it-diagnostics"
        KUBECONFIG_CREDENTIALS_ID = "kubeconfig-credentials-id" // Kubeconfig file credential in Jenkins
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Max-Medov/IT-Diagnostics-Management-Platform.git'
            }
        }

        stage('Install & Test Backend') {
            steps {
                sh """
                   # Create and activate a Python virtual environment
                   python3 -m venv venv
                   . venv/bin/activate

                   # Upgrade pip
                   pip install --upgrade pip

                   # Auth Service Tests
                   cd backend/auth_service
                   pip install -r requirements.txt
                   python -m pytest tests/ --maxfail=1 --disable-warnings -v

                   # Case Service Tests
                   cd ../case_service
                   pip install -r requirements.txt
                   python -m pytest tests/ --maxfail=1 --disable-warnings -v

                   # Diagnostic Service Tests
                   cd ../diagnostic_service
                   pip install -r requirements.txt
                   python -m pytest tests/ --maxfail=1 --disable-warnings -v

                   # Deactivate the virtual environment
                   deactivate
                """
            }
        }

        stage('Install & Test Frontend') {
            steps {
                sh """
                   cd frontend
                   npm install
                   npm run test -- --watchAll=false
                """
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
                    # Ensure all services are rolled out successfully
                    kubectl rollout status deployment/auth-service -n ${KUBE_NAMESPACE} --timeout=180s
                    kubectl rollout status deployment/case-service -n ${KUBE_NAMESPACE} --timeout=180s
                    kubectl rollout status deployment/diagnostic-service -n ${KUBE_NAMESPACE} --timeout=180s
                    kubectl rollout status deployment/frontend -n ${KUBE_NAMESPACE} --timeout=180s

                    # Check frontend availability
                    curl -f http://frontend.local || (echo "Frontend not responding" && exit 1)
                    """
                }
            }
        }

        stage('Integration Tests') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'test-user-credentials', usernameVariable: 'TEST_USER', passwordVariable: 'TEST_PASS')]) {
                    script {
                        // Register a new user
                        sh """
                        curl -f -X POST -H 'Content-Type: application/json' \\
                            -d '{"username": "${TEST_USER}", "password": "${TEST_PASS}"}' \\
                            http://auth.local/register || (echo "User registration failed" && exit 1)
                        """

                        // Login and obtain token
                        sh """
                        TOKEN=\$(curl -f -X POST -H 'Content-Type: application/json' \\
                                 -d '{"username": "${TEST_USER}", "password": "${TEST_PASS}"}' \\
                                 http://auth.local/login | jq -r '.access_token')

                        if [ -z "\$TOKEN" ] || [ "\$TOKEN" = "null" ]; then
                          echo "Failed to obtain JWT token"
                          exit 1
                        fi

                        echo "TOKEN=\$TOKEN" > token_env.sh
                        """

                        sh '. ./token_env.sh'

                        // Create a case using the token
                        sh """
                        curl -f -X POST -H 'Content-Type: application/json' -H "Authorization: Bearer \$TOKEN" \\
                            -d '{"description": "Integration Test Case", "platform": "Linux Machine"}' \\
                            http://case.local/cases || (echo "Case creation failed" && exit 1)
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'All stages completed successfully: Build, test, push, deploy, and integration tests!'
        }
        failure {
            echo 'Some stage failed. Check the logs for details.'
        }
        always {
            cleanWs()
        }
    }
}
