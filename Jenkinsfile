
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    environment {
        DOCKERHUB_NAMESPACE = 'yashuraj'
        K8S_NAMESPACE = 'default'
    }

    stages {

        // 1. Checkout source code from GitHub
        stage('Checkout') {
            steps {
                checkout scm

                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short=12 HEAD',
                        returnStdout: true
                    ).trim()

                    if (!env.IMAGE_TAG) {
                        error('Could not determine Git commit for image tag.')
                    }

                    echo "Building images with tag: ${env.IMAGE_TAG}"
                }
            }
        }

        // 2. Build and push Docker images
        stage('Build and push images') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    )
                ]) {
                    sh '''#!/usr/bin/env bash
                        set -euo pipefail

                        echo "Logging in to Docker Hub..."

                        printf '%s' "$DOCKERHUB_TOKEN" | docker login \
                            --username "$DOCKERHUB_USER" \
                            --password-stdin

                        echo "Building backend image..."

                        docker build \
                            -t "$DOCKERHUB_NAMESPACE/realsite-backend:$IMAGE_TAG" \
                            -f backend/Dockerfile backend

                        echo "Building frontend image..."

                        docker build \
                            -t "$DOCKERHUB_NAMESPACE/realsite-frontend:$IMAGE_TAG" \
                            -f frontend/Dockerfile frontend

                        echo "Pushing backend image..."

                        docker push \
                            "$DOCKERHUB_NAMESPACE/realsite-backend:$IMAGE_TAG"

                        echo "Pushing frontend image..."

                        docker push \
                            "$DOCKERHUB_NAMESPACE/realsite-frontend:$IMAGE_TAG"

                        echo "Both Docker images pushed successfully."
                    '''
                }
            }
        }

        // 3. Deploy application to Kubernetes
        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG'
                    ),
                    string(
                        credentialsId: 'realsite-postgres-password',
                        variable: 'POSTGRES_PASSWORD'
                    )
                ]) {
                    sh '''#!/usr/bin/env bash
                        set -euo pipefail

                        echo "Checking Kubernetes connection..."

                        kubectl cluster-info
                        kubectl get nodes

                        echo "Creating PostgreSQL secret..."

                        kubectl create secret generic realsite-postgres-secret \
                            --namespace "$K8S_NAMESPACE" \
                            --from-literal=POSTGRES_USER=portal \
                            --from-literal=POSTGRES_PASSWORD="$POSTGRES_PASSWORD" \
                            --from-literal=POSTGRES_DB=portal \
                            --dry-run=client -o yaml | kubectl apply -f -

                        echo "Creating backend secret..."

                        kubectl create secret generic realsite-backend-secret \
                            --namespace "$K8S_NAMESPACE" \
                            --from-literal="DATABASE_URL=postgresql+psycopg://portal:$POSTGRES_PASSWORD@postgres:5432/portal" \
                            --from-literal=SMTP_PASSWORD= \
                            --dry-run=client -o yaml | kubectl apply -f -

                        echo "Applying Kubernetes manifests..."

                        kubectl apply -n "$K8S_NAMESPACE" \
                            -f backend/k8s/configmap.yaml \
                            -f backend/k8s/storage.yaml \
                            -f backend/k8s/postgres.yaml \
                            -f backend/k8s/deployment.yaml \
                            -f backend/k8s/service.yaml \
                            -f frontend/k8s/deployment.yaml \
                            -f frontend/k8s/service.yaml

                        echo "Updating backend image..."

                        kubectl set image -n "$K8S_NAMESPACE" \
                            deployment/realsite-backend \
                            "realsite-backend=$DOCKERHUB_NAMESPACE/realsite-backend:$IMAGE_TAG"

                        echo "Updating frontend image..."

                        kubectl set image -n "$K8S_NAMESPACE" \
                            deployment/realsite-frontend \
                            "realsite-frontend=$DOCKERHUB_NAMESPACE/realsite-frontend:$IMAGE_TAG"

                        echo "Waiting for PostgreSQL rollout..."

                        kubectl rollout status -n "$K8S_NAMESPACE" \
                            deployment/realsite-postgres \
                            --timeout=180s

                        echo "Waiting for Redis rollout..."

                        kubectl rollout status -n "$K8S_NAMESPACE" \
                            deployment/realsite-redis \
                            --timeout=180s

                        echo "Waiting for backend rollout..."

                        kubectl rollout status -n "$K8S_NAMESPACE" \
                            deployment/realsite-backend \
                            --timeout=240s

                        echo "Waiting for frontend rollout..."

                        kubectl rollout status -n "$K8S_NAMESPACE" \
                            deployment/realsite-frontend \
                            --timeout=180s

                        echo "Kubernetes deployment completed successfully."

                        kubectl get pods -n "$K8S_NAMESPACE"
                        kubectl get services -n "$K8S_NAMESPACE"
                    '''
                }
            }
        }
    }

    // 4. Post-build actions
    post {
        always {
            sh '''
                docker logout >/dev/null 2>&1 || true
            '''
        }

        success {
            echo 'SUCCESS: ATPL Realsite deployment completed!'
            echo 'Use: kubectl port-forward service/realsite-frontend 8090:80'
        }

        failure {
            echo 'FAILED: Check Jenkins Console Output for the failed stage.'
        }
    }
}

