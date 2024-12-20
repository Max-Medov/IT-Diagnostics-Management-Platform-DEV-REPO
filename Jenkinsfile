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
        // Checkout the main application repository
        stage('Checkout Application Code') {
            steps {
                git branch: 'main', url: 'https://github.com/Max-Medov/IT-Diagnostics-Management-Platform.git'
            }
        }

        // Checkout the repository containing Kubernetes YAML files
        stage('Checkout Kubernetes Configurations') {
            steps {
                dir('kubernetes-config') {
                    git branch: 'main', url: 'https://github.com/Max-Medov/IT-Diagnostics-Management-Platform-DEV-REPO.git'
                }
            }
        }

        // Build Docker images for each service
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

        // Push Docker images to Docker Hub
        stage('Push Images to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${REGISTRY_CREDENTIALS}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
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

        // Deploy Kubernetes configurations
        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: "kubeconfig-credentials-id", variable: 'KUBECONFIG')]) {
                    dir('kubernetes-config/kubernetes') {
                        sh """
                        kubectl apply -f namespace.yaml 
                        kubectl apply -f secrets-configmap.yaml 
                        kubectl apply -f postgres.yaml 
                        kubectl apply -f auth-service.yaml 
                        kubectl apply -f case-service.yaml 
                        kubectl apply -f diagnostic-service.yaml 
                        kubectl apply -f frontend.yaml 
                        kubectl apply -f ingress.yaml 
                        """
                    }
                }
            }
        }

        // Wait for all pods to become ready
        stage('Wait for Pods') {
            steps {
                script {
                    sh """
                    kubectl rollout status deployment/auth-service -n ${KUBE_NAMESPACE} --timeout=300s
                    kubectl rollout status deployment/case-service -n ${KUBE_NAMESPACE} --timeout=300s
                    kubectl rollout status deployment/diagnostic-service -n ${KUBE_NAMESPACE} --timeout=300s
                    kubectl rollout status deployment/frontend -n ${KUBE_NAMESPACE} --timeout=300s
                    """
                }
            }
        }

        // Perform integration tests
        stage('Integration Tests') {
            steps {
                script {
                    sh """
                    # Check if frontend is available
                    curl -f http://frontend.it-diagnostics.svc.cluster.local || (echo "Frontend not responding after deployment" && exit 1)

                    # Register a user
                    curl -f -X POST -H 'Content-Type: application/json' \
                        -d '{"username": "${TEST_USER}", "password": "${TEST_PASS}"}' \
                        http://auth-service.it-diagnostics.svc.cluster.local:5000/register || (echo "User registration failed" && exit 1)

                    # Login and get token
                    TOKEN=\$(curl -f -X POST -H 'Content-Type: application/json' \
                        -d '{"username": "${TEST_USER}", "password": "${TEST_PASS}"}' \
                        http://auth-service.it-diagnostics.svc.cluster.local:5000/login | jq -r '.access_token')

                    if [ -z "\$TOKEN" ] || [ "\$TOKEN" = "null" ]; then
                        echo "Login failed"
                        exit 1
                    fi
                    echo "TOKEN=\$TOKEN"

                    # Create a case
                    curl -f -X POST -H 'Content-Type: application/json' -H "Authorization: Bearer \$TOKEN" \
                        -d '{"description": "Integration Test Case", "platform": "Linux Machine"}' \
                        http://case-service.it-diagnostics.svc.cluster.local:5001/cases || (echo "Case creation failed" && exit 1)

                    # Verify the created case
                    CASES=\$(curl -f -H "Authorization: Bearer \$TOKEN" http://case-service.it-diagnostics.svc.cluster.local:5001/cases)
                    echo "Received cases: \$CASES"
                    echo "\$CASES" | jq 'map(select(.description == "Integration Test Case"))' | grep "Integration Test Case" || (echo "Created case not found in case list" && exit 1)
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
