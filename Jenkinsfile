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
                   #!/bin/sh
                   set -e  # Exit immediately if a command exits with a non-zero status

                   echo "Creating a Python virtual environment..."
                   python3 -m venv venv

                   echo "Activating the virtual environment..."
                   . venv/bin/activate

                   echo "Python path inside virtualenv: \$(which python)"
                   echo "Pip path inside virtualenv: \$(which pip)"

                   echo "Upgrading pip..."
                   pip install --upgrade pip

                   echo "Running Auth Service Tests..."
                   cd backend/auth_service
                   pip install -r requirements.txt
                   python -m pytest tests/ --maxfail=1 --disable-warnings -v

                   echo "Running Case Service Tests..."
                   cd ../case_service
                   pip install -r requirements.txt
                   python -m pytest tests/ --maxfail=1 --disable-warnings -v

                   echo "Running Diagnostic Service Tests..."
                   cd ../diagnostic_service
                   pip install -r requirements.txt
                   python -m pytest tests/ --maxfail=1 --disable-warnings -v

                   echo "Deactivating the virtual environment..."
                   deactivate

                   echo "Backend installation and testing completed successfully."
                """
            }
        }

        stage('Install & Test Frontend') {
            steps {
                sh """
                   #!/bin/sh
                   set -e  # Exit immediately if a command exits with a non-zero status

                   echo "Installing frontend dependencies..."
                   cd frontend
                   npm install

                   echo "Running frontend tests..."
                   npm run test -- --watchAll=false

                   echo "Frontend installation and testing completed successfully."
                """
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    sh """
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

        stage('Push Images to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${env.REGISTRY_CREDENTIALS}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                       #!/bin/sh
                       set -e  # Exit immediately if a command exits with a non-zero status

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
                withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIALS_ID}", variable: 'KUBECONFIG')]) {
                    sh """
                       #!/bin/sh
                       set -e  # Exit immediately if a command exits with a non-zero status

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
                    set -e  # Exit immediately if a command exits with a non-zero status

                    echo "Waiting for Kubernetes pods to be ready..."
                    kubectl rollout status deployment/auth-service -n ${KUBE_NAMESPACE} --timeout=180s
                    kubectl rollout status deployment/case-service -n ${KUBE_NAMESPACE} --timeout=180s
                    kubectl rollout status deployment/diagnostic-service -n ${KUBE_NAMESPACE} --timeout=180s
                    kubectl rollout status deployment/frontend -n ${KUBE_NAMESPACE} --timeout=180s

                    echo "Checking frontend availability..."
                    curl -f http://frontend.local || (echo "Frontend not responding" && exit 1)
                    echo "All pods are ready and frontend is available."
                    """
                }
            }
        }

        stage('Integration Tests') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'test-user-credentials', usernameVariable: 'TEST_USER', passwordVariable: 'TEST_PASS')]) {
                    script {
                        sh """
                        #!/bin/sh
                        set -e  # Exit immediately if a command exits with a non-zero status

                        echo "Registering a new user..."
                        curl -f -X POST -H 'Content-Type: application/json' \\
                            -d '{"username": "${TEST_USER}", "password": "${TEST_PASS}"}' \\
                            http://auth.local/register || (echo "User registration failed" && exit 1)

                        echo "Logging in to obtain JWT token..."
                        TOKEN=\$(curl -f -X POST -H 'Content-Type: application/json' \\
                                 -d '{"username": "${TEST_USER}", "password": "${TEST_PASS}"}' \\
                                 http://auth.local/login | jq -r '.access_token')

                        if [ -z "\$TOKEN" ] || [ "\$TOKEN" = "null" ]; then
                          echo "Failed to obtain JWT token"
                          exit 1
                        fi

                        echo "TOKEN=\$TOKEN" > token_env.sh

                        echo "Creating a case using the JWT token..."
                        curl -f -X POST -H 'Content-Type: application/json' -H "Authorization: Bearer \$TOKEN" \\
                            -d '{"description": "Integration Test Case", "platform": "Linux Machine"}' \\
                            http://case.local/cases || (echo "Case creation failed" && exit 1)

                        echo "Integration tests completed successfully."
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
