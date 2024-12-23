pipeline {
    agent any

    environment {
        // Docker/Registry info
        REGISTRY = "docker.io"
        REGISTRY_CREDENTIALS = "dockerhub-credentials-id"
        DOCKER_ORG = "maxmedov"
        IMAGE_PREFIX = "it-diagnostics-management-platform"

        // Kubernetes info
        KUBE_NAMESPACE = "it-diagnostics"
        KUBECONFIG_CREDENTIALS_ID = "kubeconfig-credentials-id"

        // Test data for integration tests
        TEST_USER = "testuser"
        TEST_PASS = "testpass"
    }

    stages {
        // 1. Checkout main application repo
        stage('Checkout Application Code') {
            steps {
                git branch: 'main', url: 'https://github.com/Max-Medov/IT-Diagnostics-Management-Platform.git'
            }
        }

        // 2. Checkout DEV repo containing K8s manifests
        stage('Checkout Kubernetes Configurations') {
            steps {
                dir('kubernetes-config') {
                    git branch: 'main', url: 'https://github.com/Max-Medov/IT-Diagnostics-Management-Platform-DEV-REPO.git'
                }
            }
        }

        // 3. Build Docker images for each service
        stage('Build Docker Images') {
            steps {
                script {
                    // Build auth_service
                    sh """
                        docker build -t ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:auth_service \
                        -f backend/auth_service/Dockerfile \
                        backend/auth_service
                    """

                    // Build case_service
                    sh """
                        docker build -t ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:case_service \
                        -f backend/case_service/Dockerfile \
                        backend/case_service
                    """

                    // Build diagnostic_service
                    sh """
                        docker build -t ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:diagnostic_service \
                        -f backend/diagnostic_service/Dockerfile \
                        backend/diagnostic_service
                    """

                    // Build frontend
                    sh """
                        docker build -t ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:frontend \
                        -f frontend/Dockerfile \
                        frontend
                    """
                }
            }
        }

        // 4. Minimal sanity tests
        stage('Pre-Push Minimal Sanity Tests') {
            steps {
                script {
                    def services = ['auth_service', 'case_service', 'diagnostic_service', 'frontend']
                    for (service in services) {
                        echo "Running sanity test for ${service}"

                        // Run the container in the background
                        sh """
                            docker run -d --name test_${service} \
                            ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:${service}
                            sleep 10
                        """

                        // Check if container is running
                        def running = sh(
                            script: """
                                docker ps --filter name=test_${service} --filter status=running | grep test_${service} || true
                            """,
                            returnStatus: true
                        )

                        if (running != 0) {
                            echo "${service} container not running after 10s, sanity test failed"
                            sh "docker rm -f test_${service}"
                            error("Pre-push sanity test failed for ${service}")
                        } else {
                            echo "${service} container is running, sanity test passed"
                            sh "docker rm -f test_${service}"
                        }
                    }
                }
            }
        }

        // 5. Push Docker images to Docker Hub
        stage('Push Images to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${REGISTRY_CREDENTIALS}",
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
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

        // 6. Create the Kubernetes namespace
        stage('Create Namespace') {
            steps {
                withCredentials([file(
                    credentialsId: "${KUBECONFIG_CREDENTIALS_ID}",
                    variable: 'KUBECONFIG'
                )]) {
                    dir('kubernetes-config/kubernetes') {
                        sh "kubectl apply -f namespace.yaml"
                    }
                }
            }
        }

        // 7. Create Prometheus & Grafana ConfigMaps
        stage('Create ConfigMaps') {
            steps {
                dir('kubernetes-config') {
                    script {
                        // Make sure the files exist in kubernetes-config/kubernetes
                        sh """
                            # Create or update the prometheus.yml config
                            kubectl create configmap prometheus-config \\
                              --from-file=prometheus.yml=kubernetes/prometheus.yml \\
                              -n ${KUBE_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

                            # Create or update Grafana datasources
                            kubectl create configmap grafana-datasources \\
                              --from-file=datasources.yaml=kubernetes/datasources.yaml \\
                              -n ${KUBE_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

                            # Create or update Grafana dashboard (auth-service-dashboard.json)
                            kubectl create configmap grafana-dashboard \\
                              --from-file=auth-service-dashboard.json=kubernetes/auth-service-dashboard.json \\
                              -n ${KUBE_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

                            # Create or update Grafana dashboard provider
                            kubectl create configmap grafana-dashboard-provider \\
                              --from-file=dashboard-provider.yaml=kubernetes/grafana-dashboard-provider.yaml \\
                              -n ${KUBE_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                        """
                    }
                }
            }
        }

        // 8. Deploy all Kubernetes manifests
        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(
                    credentialsId: "${KUBECONFIG_CREDENTIALS_ID}",
                    variable: 'KUBECONFIG'
                )]) {
                    dir('kubernetes-config/kubernetes') {
                        sh """
                            # Apply your secrets/configmaps if you have them
                            kubectl apply -f secrets-configmap.yaml
                            kubectl apply -f postgres.yaml

                            # Deploy each microservice
                            kubectl apply -f auth-service.yaml
                            kubectl apply -f case-service.yaml
                            kubectl apply -f diagnostic-service.yaml
                            kubectl apply -f frontend.yaml
                            kubectl apply -f ingress.yaml

                            # Deploy Prometheus & Grafana
                            kubectl apply -f prometheus-rbac.yaml
                            kubectl apply -f prometheus-k8s.yaml
                            kubectl apply -f grafana-dashboard-configmap.yaml
                            kubectl apply -f grafana-dashboard-provider.yaml
                            kubectl apply -f grafana.yaml
                        """
                    }
                }
            }
        }

        // 9. Wait for pods
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

        // 10. Integration tests
        stage('Integration Tests') {
            steps {
                script {
                    sh """
                        # Run port-forward in the background for each service
                        kubectl port-forward svc/auth-service -n ${KUBE_NAMESPACE} 5000:5000 > auth-pf.log 2>&1 &
                        AUTH_PF_PID=\$!
                        kubectl port-forward svc/case-service -n ${KUBE_NAMESPACE} 5001:5001 > case-pf.log 2>&1 &
                        CASE_PF_PID=\$!
                        kubectl port-forward svc/diagnostic-service -n ${KUBE_NAMESPACE} 5002:5002 > diagnostic-pf.log 2>&1 &
                        DIAGNOSTIC_PF_PID=\$!

                        # Allow time for port-forwarding
                        sleep 10

                        #################################################
                        # 1) Test auth-service: register or detect user
                        #################################################
                        REGISTER_RESPONSE=\$(curl -s -o /dev/null -w "%{http_code}" -X POST -H 'Content-Type: application/json' \\
                            -d '{"username": "${TEST_USER}", "password": "${TEST_PASS}"}' \\
                            http://localhost:5000/register)

                        if [ "\$REGISTER_RESPONSE" = "409" ]; then
                            echo "User already exists. Proceeding with login..."
                        elif [ "\$REGISTER_RESPONSE" = "201" ]; then
                            echo "User registered successfully."
                        else
                            echo "Unexpected error during user registration. HTTP Status: \$REGISTER_RESPONSE"
                            exit 1
                        fi

                        #################################################
                        # 2) Login to get JWT
                        #################################################
                        TOKEN=\$(curl -s -X POST -H 'Content-Type: application/json' \\
                            -d '{"username": "${TEST_USER}", "password": "${TEST_PASS}"}' \\
                            http://localhost:5000/login | jq -r '.access_token')

                        if [ -z "\$TOKEN" ] || [ "\$TOKEN" = "null" ]; then
                            echo "Login failed"
                            exit 1
                        fi
                        echo "TOKEN=\$TOKEN"

                        #################################################
                        # 3) Test case-service: create a case
                        #################################################
                        curl -f -X POST -H 'Content-Type: application/json' -H "Authorization: Bearer \$TOKEN" \\
                            -d '{"description": "Integration Test Case", "platform": "Linux Machine"}' \\
                            http://localhost:5001/cases || (echo "Case creation failed" && exit 1)

                        #################################################
                        # 4) Test case-service: retrieve cases
                        #################################################
                        CASES=\$(curl -sf -H "Authorization: Bearer \$TOKEN" http://localhost:5001/cases)
                        echo "Received cases: \$CASES"
                        echo "\$CASES" | jq 'map(select(.description == "Integration Test Case"))' | grep "Integration Test Case" || (echo "Created case not found in list" && exit 1)

                        #################################################
                        # 5) Test diagnostic-service
                        #################################################
                        curl -sf -H "Authorization: Bearer \$TOKEN" http://localhost:5002/download_script/1 || (echo "Diagnostic service not responding" && exit 1)

                        #################################################
                        # Cleanup: kill port-forward
                        #################################################
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

