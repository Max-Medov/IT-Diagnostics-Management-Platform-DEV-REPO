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

        // 1) Checkout main code
        stage('Checkout Application Code') {
            steps {
                git branch: 'main', url: 'https://github.com/Max-Medov/IT-Diagnostics-Management-Platform.git'
            }
        }

        // 2) Checkout dev repo for Kubernetes YAML
        stage('Checkout Kubernetes Configurations') {
            steps {
                dir('kubernetes-config') {
                    git branch: 'main', url: 'https://github.com/Max-Medov/IT-Diagnostics-Management-Platform-DEV-REPO.git'
                }
            }
        }

        // 3) Build Docker Images
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

        // 4) Pre-Push Minimal Sanity Tests
        stage('Pre-Push Minimal Sanity Tests') {
            steps {
                script {
                    def services = ['auth_service', 'case_service', 'diagnostic_service', 'frontend']
                    for (service in services) {
                        echo "Running sanity test for ${service}"
                        sh """
                          docker run -d --name test_${service} ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:${service}
                          sleep 10
                        """
                        def running = sh(script: """
                          docker ps --filter name=test_${service} --filter status=running | grep test_${service} || true
                        """, returnStatus: true)
                        if (running != 0) {
                            sh "docker rm -f test_${service}"
                            error("Pre-push sanity test failed for ${service}")
                        } else {
                            sh "docker rm -f test_${service}"
                        }
                    }
                }
            }
        }

        // 5) Push Docker Images
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

        // 6) Create Namespace
        stage('Create Namespace') {
            steps {
                withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIALS_ID}", variable: 'KUBECONFIG')]) {
                    dir('kubernetes-config/kubernetes') {
                        sh "kubectl apply -f namespace.yaml"
                    }
                }
            }
        }

        // 7) Deploy to Kubernetes (no create configmap --from-file)
        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIALS_ID}", variable: 'KUBECONFIG')]) {
                    dir('kubernetes-config/kubernetes') {
                        sh """
                          # Apply your secrets, Postgres, etc.
                          kubectl apply -f secrets-configmap.yaml
                          kubectl apply -f postgres.yaml

                          # Deploy all services
                          kubectl apply -f auth-service.yaml
                          kubectl apply -f case-service.yaml
                          kubectl apply -f diagnostic-service.yaml
                          kubectl apply -f frontend.yaml
                          kubectl apply -f ingress.yaml
                          kubectl apply -f prometheus-rbac.yaml
                          kubectl apply -f prometheus-k8s.yaml

                          # Now apply the Grafana config + service
                          kubectl apply -f grafana-dashboard-provider.yaml
                          kubectl apply -f grafana-dashboard-configmap.yaml
                          kubectl apply -f datasources.yaml
                          kubectl apply -f grafana.yaml
                        """
                    }
                }
            }
        }

        // 8) Wait for Pods
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

        // 9) Integration Tests
        stage('Integration Tests') {
            steps {
                script {
                    sh """
                      # Run port-forwards
                      kubectl port-forward svc/auth-service -n ${KUBE_NAMESPACE} 5000:5000 > auth-pf.log 2>&1 &
                      AUTH_PF_PID=\$!
                      kubectl port-forward svc/case-service -n ${KUBE_NAMESPACE} 5001:5001 > case-pf.log 2>&1 &
                      CASE_PF_PID=\$!
                      kubectl port-forward svc/diagnostic-service -n ${KUBE_NAMESPACE} 5002:5002 > diag-pf.log 2>&1 &
                      DIAG_PF_PID=\$!

                      sleep 10

                      # Test auth-service
                      REGISTER_RESPONSE=\$(curl -s -o /dev/null -w "%{http_code}" -X POST -H 'Content-Type: application/json' \\
                        -d '{"username": "${TEST_USER}", "password": "${TEST_PASS}"}' http://localhost:5000/register)
                      if [ "\$REGISTER_RESPONSE" = "409" ]; then
                        echo "User exists, ok"
                      elif [ "\$REGISTER_RESPONSE" = "201" ]; then
                        echo "User registered"
                      else
                        echo "Registration failed with code \$REGISTER_RESPONSE"
                        exit 1
                      fi

                      TOKEN=\$(curl -s -f -X POST -H 'Content-Type: application/json' \\
                        -d '{"username": "${TEST_USER}", "password": "${TEST_PASS}"}' http://localhost:5000/login | jq -r '.access_token')
                      if [ -z "\$TOKEN" ] || [ "\$TOKEN" = "null" ]; then
                        echo "Login failed"
                        exit 1
                      fi

                      # Quick test case-service
                      curl -f -X POST -H 'Content-Type: application/json' -H "Authorization: Bearer \$TOKEN" \\
                        -d '{"description": "Integration Test Case", "platform": "Linux Machine"}' \\
                        http://localhost:5001/cases || exit 1

                      # Quick test diagnostic-service
                      curl -f -H "Authorization: Bearer \$TOKEN" http://localhost:5002/download_script/1 || exit 1

                      kill \$AUTH_PF_PID || true
                      kill \$CASE_PF_PID || true
                      kill \$DIAG_PF_PID || true
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

