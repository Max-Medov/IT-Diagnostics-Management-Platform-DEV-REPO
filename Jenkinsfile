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

        // Create Prometheus and Grafana ConfigMaps
        stage('Create ConfigMaps') {
            steps {
                script {
                    sh """
                    kubectl create configmap prometheus-config \
                      --from-file=prometheus.yml=prometheus.yml \
                      -n ${KUBE_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

                    kubectl create configmap grafana-datasources \
                      --from-file=datasources.yaml=datasources.yaml \
                      -n ${KUBE_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

                    kubectl create configmap grafana-dashboard \
                      --from-file=auth-service-dashboard.json=auth-service-dashboard.json \
                      -n ${KUBE_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                    """
                }
            }
        }

        // Deploy Kubernetes configurations
        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIALS_ID}", variable: 'KUBECONFIG')]) {
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
                        kubectl apply -f prometheus.yaml
                        kubectl apply -f grafana.yaml
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
                    kubectl rollout status deployment/prometheus -n ${KUBE_NAMESPACE} --timeout=300s
                    kubectl rollout status deployment/grafana -n ${KUBE_NAMESPACE} --timeout=300s
                    """
                }
            }
        }

        // Perform integration tests
        stage('Integration Tests') {
            steps {
                script {
                    sh """
                    # Run port-forwarding in the background
                    kubectl port-forward svc/auth-service -n ${KUBE_NAMESPACE} 5000:5000 > auth-pf.log 2>&1 &
                    AUTH_PF_PID=\$!
                    kubectl port-forward svc/case-service -n ${KUBE_NAMESPACE} 5001:5001 > case-pf.log 2>&1 &
                    CASE_PF_PID=\$!
                    kubectl port-forward svc/diagnostic-service -n ${KUBE_NAMESPACE} 5002:5002 > diagnostic-pf.log 2>&1 &
                    DIAGNOSTIC_PF_PID=\$!

                    # Ensure port-forwarding is running
                    sleep 10

                    # Test auth-service: check if user exists
                    REGISTER_RESPONSE=\$(curl -s -o /dev/null -w "%{http_code}" -X POST -H 'Content-Type: application/json' \
                        -d '{"username": "${TEST_USER}", "password": "${TEST_PASS}"}' \
                        http://localhost:5000/register)

                    if [ "\$REGISTER_RESPONSE" = "409" ]; then
                        echo "User already exists. Proceeding with login..."
                    elif [ "\$REGISTER_RESPONSE" = "201" ]; then
                        echo "User registered successfully."
                    else
                        echo "Unexpected error during user registration. HTTP Status: \$REGISTER_RESPONSE"
                        exit 1
                    fi

                    # Test auth-service: login and get token
                    TOKEN=\$(curl -f -X POST -H 'Content-Type: application/json' \
                        -d '{"username": "${TEST_USER}", "password": "${TEST_PASS}"}' \
                        http://localhost:5000/login | jq -r '.access_token')

                    if [ -z "\$TOKEN" ] || [ "\$TOKEN" = "null" ]; then
                        echo "Login failed"
                        exit 1
                    fi
                    echo "TOKEN=\$TOKEN"

                    # Test case-service: create a case
                    curl -f -X POST -H 'Content-Type: application/json' -H "Authorization: Bearer \$TOKEN" \
                        -d '{"description": "Integration Test Case", "platform": "Linux Machine"}' \
                        http://localhost:5001/cases || (echo "Case creation failed" && exit 1)

                    # Test case-service: retrieve cases
                    CASES=\$(curl -f -H "Authorization: Bearer \$TOKEN" http://localhost:5001/cases)
                    echo "Received cases: \$CASES"
                    echo "\$CASES" | jq 'map(select(.description == "Integration Test Case"))' | grep "Integration Test Case" || (echo "Created case not found in case list" && exit 1)

                    # Test diagnostic-service: check if script is available
                    curl -f -H "Authorization: Bearer \$TOKEN" http://localhost:5002/download_script/1 || (echo "Diagnostic service not responding" && exit 1)

                    # Kill port-forwarding processes
                    kill \$AUTH_PF_PID || true
                    kill \$CASE_PF_PID || true
                    kill \$DIAGNOSTIC_PF_PID || true
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

