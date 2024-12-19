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
        
        # Set a dummy DB connection if needed, or ensure auth_service can run without DB
        # Adjust these ENV vars to what auth_service needs to run locally.
        AUTH_DB_URI = "postgresql://postgres:yourpassword@db:5432/itdiagnostics"
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

        stage('Pre-Push Sanity Tests') {
            steps {
                script {
                    // If auth_service needs a DB, run a DB container first. Adjust if your DB is different.
                    // If not needed, remove these lines.
                    sh """
                    docker run -d --name test_db -e POSTGRES_PASSWORD=yourpassword -e POSTGRES_DB=itdiagnostics -p 5432:5432 postgres:15
                    sleep 15
                    """

                    // Run auth_service with ENV vars for DB if needed
                    sh """
                    docker run -d --name test_auth -e DATABASE_URI=${AUTH_DB_URI} -p 9000:5000 ${REGISTRY}/${DOCKER_ORG}/${IMAGE_PREFIX}:auth_service
                    """

                    // Wait longer for auth_service to come up, e.g., 30 seconds
                    sh "sleep 30"

                    // Try registering a user locally
                    sh """
                    curl -f -X POST -H 'Content-Type: application/json' \
                      -d '{"username":"${TEST_USER}","password":"${TEST_PASS}"}' \
                      http://localhost:9000/register || (echo "User registration failed locally" && exit 1)
                    """

                    // Try logging in
                    sh """
                    TOKEN=\$(curl -f -X POST -H 'Content-Type: application/json' \
                      -d '{"username":"${TEST_USER}","password":"${TEST_PASS}"}' \
                      http://localhost:9000/login | jq -r '.access_token')

                    if [ -z "\$TOKEN" ] || [ "\$TOKEN" = "null" ]; then
                      echo "Login failed locally"
                      exit 1
                    fi
                    echo "Local auth_service sanity test passed with user register/login."
                    """

                    // Cleanup containers
                    sh "docker rm -f test_auth"
                    sh "docker rm -f test_db"
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
                    // Check frontend after deploy
                    sh 'curl -f http://frontend.local || (echo "Frontend not responding after deploy" && exit 1)'

                    // Register user in the deployed environment
                    sh """
                    curl -f -X POST -H 'Content-Type: application/json' \
                      -d '{"username":"${TEST_USER}","password":"${TEST_PASS}"}' \
                      http://auth.local/register || (echo "User registration failed after deploy" && exit 1)
                    """

                    // Login after deploy
                    sh """
                    TOKEN=\$(curl -f -X POST -H 'Content-Type: application/json' -d '{"username":"${TEST_USER}","password":"${TEST_PASS}"}' http://auth.local/login | jq -r '.access_token')
                    if [ -z "\$TOKEN" ] || [ "\$TOKEN" = "null" ]; then
                      echo "Login failed after deploy"
                      exit 1
                    fi
                    echo "TOKEN=\$TOKEN" > token_env.sh
                    """

                    sh '. ./token_env.sh'

                    // Create a case after deploy
                    sh """
                    curl -f -X POST -H 'Content-Type: application/json' -H "Authorization: Bearer \$TOKEN" \
                        -d '{"description": "Integration Test Case", "platform": "Linux Machine"}' \
                        http://case.local/cases || (echo "Case creation failed after deploy" && exit 1)
                    """

                    // Verify the case is listed
                    sh """
                    CASES=\$(curl -f -H "Authorization: Bearer \$TOKEN" http://case.local/cases)
                    echo "Received cases: \$CASES"
                    echo "\$CASES" | jq 'map(select(.description == "Integration Test Case"))' | grep "Integration Test Case" || (echo "Created case not found after deploy" && exit 1)
                    """

                    // Check diagnostic service endpoint after deploy
                    sh 'curl -f http://diagnostic.local/download_script/1 || (echo "Diagnostic service not responding after deploy" && exit 1)'
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully with local register/login test before push and full integration tests after deploy!'
        }
        failure {
            echo 'Some stage failed. Check logs.'
        }
        always {
            cleanWs()
        }
    }
}
