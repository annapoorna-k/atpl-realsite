pipeline {
    agent any

    environment {
        DOCKERHUB_USER = 'yashuraj'

        BACKEND_IMAGE  = 'yashuraj/realsite-backend'
        FRONTEND_IMAGE = 'yashuraj/realsite-frontend'

        K8S_NAMESPACE = 'default'

        POSTGRES_PASSWORD = credentials('realsite-postgres-password')
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {

        // ============================================================
        // 1. CHECKOUT
        // ============================================================
        stage('Checkout') {
            steps {
                checkout scm

                script {
                    IMAGE_TAG = sh(
                        script: 'git rev-parse --short=12 HEAD',
                        returnStdout: true
                    ).trim()

                    echo "Building images with tag: ${IMAGE_TAG}"
                }
            }
        }


        // ============================================================
        // 2. BUILD + PUSH DOCKER IMAGES
        // ============================================================
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
                        echo "Logging in to Docker Hub"
                        echo "=========================================="

                        echo "$DOCKERHUB_TOKEN" | docker login \
                            -u "$DOCKERHUB_LOGIN_USER" \
                            --password-stdin

                        echo "Docker login successful"
                        echo "Docker user: $DOCKERHUB_LOGIN_USER"

                        echo "Docker server information:"
                        docker info --format 'Docker Server: {{.ServerVersion}}'

                        echo "Docker context:"
                        docker context show


                        echo "=========================================="
                        echo "Building backend image"
                        echo "=========================================="

                        docker build \
                            -t "$BACKEND_IMAGE:$IMAGE_TAG" \
                            ./backend


                        echo "=========================================="
                        echo "Building frontend image"
                        echo "=========================================="

                        docker build \
                            -t "$FRONTEND_IMAGE:$IMAGE_TAG" \
                            ./frontend


                        echo "=========================================="
                        echo "Pushing backend image"
                        echo "=========================================="

                        docker push "$BACKEND_IMAGE:$IMAGE_TAG"


                        echo "=========================================="
                        echo "Pushing frontend image"
                        echo "=========================================="

                        docker push "$FRONTEND_IMAGE:$IMAGE_TAG"


                        echo "=========================================="
                        echo "Docker images pushed successfully"
                        echo "=========================================="
                    '''
                }
            }
        }


        // ============================================================
        // 3. DEPLOY TO KUBERNETES
        // ============================================================
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

                        export KUBECONFIG="$KUBECONFIG_FILE"

                        echo "=========================================="
                        echo "Kubernetes cluster information"
                        echo "=========================================="

                        kubectl cluster-info

                        echo "Current Kubernetes context:"
                        kubectl config current-context


                        echo "=========================================="
                        echo "Applying backend configuration"
                        echo "=========================================="

                        kubectl apply \
                            -f backend/k8s/configmap.yaml \
                            -n "$K8S_NAMESPACE"

                        kubectl apply \
                            -f backend/k8s/storage.yaml \
                            -n "$K8S_NAMESPACE"

                        kubectl apply \
                            -f backend/k8s/postgres.yaml \
                            -n "$K8S_NAMESPACE"

                        kubectl apply \
                            -f backend/k8s/deployment.yaml \
                            -n "$K8S_NAMESPACE"

                        kubectl apply \
                            -f backend/k8s/service.yaml \
                            -n "$K8S_NAMESPACE"


                        echo "=========================================="
                        echo "Applying frontend configuration"
                        echo "=========================================="

                        kubectl apply \
                            -f frontend/k8s/deployment.yaml \
                            -n "$K8S_NAMESPACE"

                        kubectl apply \
                            -f frontend/k8s/service.yaml \
                            -n "$K8S_NAMESPACE"


                        echo "=========================================="
                        echo "Updating backend image"
                        echo "=========================================="

                        kubectl set image deployment/realsite-backend \
                            realsite-backend="$BACKEND_IMAGE:$IMAGE_TAG" \
                            -n "$K8S_NAMESPACE"


                        echo "=========================================="
                        echo "Updating frontend image"
                        echo "=========================================="

                        kubectl set image deployment/realsite-frontend \
                            realsite-frontend="$FRONTEND_IMAGE:$IMAGE_TAG" \
                            -n "$K8S_NAMESPACE"


                        echo "=========================================="
                        echo "Waiting for backend rollout"
                        echo "=========================================="

                        kubectl rollout status \
                            deployment/realsite-backend \
                            -n "$K8S_NAMESPACE" \
                            --timeout=180s


                        echo "=========================================="
                        echo "Waiting for frontend rollout"
                        echo "=========================================="

                        kubectl rollout status \
                            deployment/realsite-frontend \
                            -n "$K8S_NAMESPACE" \
                            --timeout=180s


                        echo "=========================================="
                        echo "Kubernetes pods"
                        echo "=========================================="

                        kubectl get pods \
                            -n "$K8S_NAMESPACE" \
                            -o wide


                        echo "=========================================="
                        echo "Kubernetes services"
                        echo "=========================================="

                        kubectl get services \
                            -n "$K8S_NAMESPACE"


                        echo "=========================================="
                        echo "Deployment completed successfully"
                        echo "=========================================="
                    '''
                }
            }
        }
    }


    // ================================================================
    // POST ACTIONS
    // ================================================================
    post {

        success {
            echo "=========================================="
            echo "SUCCESS: CI/CD pipeline completed"
            echo "Image tag: ${IMAGE_TAG}"
            echo "=========================================="
        }

        failure {
            echo "=========================================="
            echo "FAILED: Check Jenkins Console Output"
            echo "=========================================="
        }

        always {
            sh '''
                docker logout || true
            '''
        }
    }
}

