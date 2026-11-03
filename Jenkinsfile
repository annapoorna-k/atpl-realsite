pipeline {
    agent any

    environment {
        // Docker Hub
        DOCKERHUB_USER = 'yashuraj'

        BACKEND_IMAGE  = 'yashuraj/realsite-backend'
        FRONTEND_IMAGE = 'yashuraj/realsite-frontend'

        // Kubernetes
        K8S_NAMESPACE = 'default'

        // PostgreSQL Jenkins credential
        // Credential type: Secret text
        POSTGRES_PASSWORD = credentials('postgress-credentials')
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {

        // =========================================================
        // 1. CHECKOUT
        // =========================================================
        stage('Checkout') {
            steps {
                checkout scm

                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short=12 HEAD',
                        returnStdout: true
                    ).trim()

                    echo "=========================================="
                    echo "Git checkout completed"
                    echo "Image tag: ${env.IMAGE_TAG}"
                    echo "=========================================="
                }
            }
        }

        // =========================================================
        // 2. BUILD AND PUSH DOCKER IMAGES
        // =========================================================
        stage('Build and push images') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKERHUB_LOGIN_USER',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    )
                ]) {

                    sh '''
                        set -e

                        echo "=========================================="
                        echo "Docker Hub Login"
                        echo "=========================================="

                        echo "$DOCKERHUB_TOKEN" | docker login \
                            -u "$DOCKERHUB_LOGIN_USER" \
                            --password-stdin

                        echo "Docker login successful"
                        echo "Docker user: $DOCKERHUB_LOGIN_USER"

                        echo "=========================================="
                        echo "Docker Information"
                        echo "=========================================="

                        docker info --format 'Docker Server: {{.ServerVersion}}'

                        echo "Docker context:"
                        docker context show

                        echo "=========================================="
                        echo "Building Backend Image"
                        echo "=========================================="

                        echo "Image: $BACKEND_IMAGE:$IMAGE_TAG"

                        docker build \
                            -t "$BACKEND_IMAGE:$IMAGE_TAG" \
                            ./backend

                        echo "Backend image build completed"

                        echo "=========================================="
                        echo "Building Frontend Image"
                        echo "=========================================="

                        echo "Image: $FRONTEND_IMAGE:$IMAGE_TAG"

                        docker build \
                            -t "$FRONTEND_IMAGE:$IMAGE_TAG" \
                            ./frontend

                        echo "Frontend image build completed"

                        echo "=========================================="
                        echo "Pushing Backend Image"
                        echo "=========================================="

                        docker push "$BACKEND_IMAGE:$IMAGE_TAG"

                        echo "Backend image pushed successfully"

                        echo "=========================================="
                        echo "Pushing Frontend Image"
                        echo "=========================================="

                        docker push "$FRONTEND_IMAGE:$IMAGE_TAG"

                        echo "Frontend image pushed successfully"

                        echo "=========================================="
                        echo "Docker images pushed successfully"
                        echo "=========================================="
                    '''
                }
            }
        }

        // =========================================================
        // 3. DEPLOY TO KUBERNETES
        // =========================================================
        stage('Deploy to Kubernetes') {
            steps {

                withCredentials([
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {

                    sh '''
                        set -e

                        echo "=========================================="
                        echo "Kubernetes Deployment"
                        echo "=========================================="

                        export KUBECONFIG="$KUBECONFIG_FILE"

                        echo "Kubernetes cluster:"
                        kubectl cluster-info

                        echo "Current Kubernetes context:"
                        kubectl config current-context

                        echo "=========================================="
                        echo "Applying Backend ConfigMap"
                        echo "=========================================="

                        kubectl apply \
                            -f backend/k8s/configmap.yaml \
                            -n "$K8S_NAMESPACE"

                        echo "=========================================="
                        echo "Applying Storage"
                        echo "=========================================="

                        kubectl apply \
                            -f backend/k8s/storage.yaml \
                            -n "$K8S_NAMESPACE"

                        echo "=========================================="
                        echo "Applying PostgreSQL"
                        echo "=========================================="

                        kubectl apply \
                            -f backend/k8s/postgres.yaml \
                            -n "$K8S_NAMESPACE"

                        echo "=========================================="
                        echo "Applying Backend Deployment"
                        echo "=========================================="

                        kubectl apply \
                            -f backend/k8s/deployment.yaml \
                            -n "$K8S_NAMESPACE"

                        echo "=========================================="
                        echo "Applying Backend Service"
                        echo "=========================================="

                        kubectl apply \
                            -f backend/k8s/service.yaml \
                            -n "$K8S_NAMESPACE"

                        echo "=========================================="
                        echo "Applying Frontend Deployment"
                        echo "=========================================="

                        kubectl apply \
                            -f frontend/k8s/deployment.yaml \
                            -n "$K8S_NAMESPACE"

                        echo "=========================================="
                        echo "Applying Frontend Service"
                        echo "=========================================="

                        kubectl apply \
                            -f frontend/k8s/service.yaml \
                            -n "$K8S_NAMESPACE"

                        echo "=========================================="
                        echo "Updating Backend Image"
                        echo "=========================================="

                        kubectl set image \
                            deployment/realsite-backend \
                            realsite-backend="$BACKEND_IMAGE:$IMAGE_TAG" \
                            -n "$K8S_NAMESPACE"

                        echo "=========================================="
                        echo "Updating Frontend Image"
                        echo "=========================================="

                        kubectl set image \
                            deployment/realsite-frontend \
                            realsite-frontend="$FRONTEND_IMAGE:$IMAGE_TAG" \
                            -n "$K8S_NAMESPACE"

                        echo "=========================================="
                        echo "Waiting for Backend Rollout"
                        echo "=========================================="

                        kubectl rollout status \
                            deployment/realsite-backend \
                            -n "$K8S_NAMESPACE" \
                            --timeout=180s

                        echo "Backend rollout successful"

                        echo "=========================================="
                        echo "Waiting for Frontend Rollout"
                        echo "=========================================="

                        kubectl rollout status \
                            deployment/realsite-frontend \
                            -n "$K8S_NAMESPACE" \
                            --timeout=180s

                        echo "Frontend rollout successful"

                        echo "=========================================="
                        echo "Kubernetes Pods"
                        echo "=========================================="

                        kubectl get pods \
                            -n "$K8S_NAMESPACE" \
                            -o wide

                        echo "=========================================="
                        echo "Kubernetes Services"
                        echo "=========================================="

                        kubectl get services \
                            -n "$K8S_NAMESPACE"

                        echo "=========================================="
                        echo "Deployment Completed"
                        echo "Image tag: $IMAGE_TAG"
                        echo "=========================================="
                    '''
                }
            }
        }
    }

    // =============================================================
    // POST ACTIONS
    // =============================================================
    post {

        success {
            echo "=========================================="
            echo "SUCCESS: CI/CD pipeline completed"
            echo "Image tag: ${env.IMAGE_TAG}"
            echo "=========================================="
        }

        failure {
            echo "=========================================="
            echo "FAILED: Check Jenkins Console Output"
            echo "=========================================="
        }

        always {
            echo "Pipeline finished."
        }
    }
}
