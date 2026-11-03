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
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.IMAGE_TAG = env.GIT_COMMIT.take(12)
                }
            }
        }

        stage('Build and push images') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_TOKEN'
                )]) {
                    sh '''#!/usr/bin/env bash
                        set -euo pipefail
                        printf '%s' "$DOCKERHUB_TOKEN" | docker login \
                          --username "$DOCKERHUB_USER" --password-stdin
                        docker build -t "$DOCKERHUB_NAMESPACE/realsite-backend:$IMAGE_TAG" \
                          -f backend/Dockerfile backend
                        docker build -t "$DOCKERHUB_NAMESPACE/realsite-frontend:$IMAGE_TAG" \
                          -f frontend/Dockerfile frontend
                        docker push "$DOCKERHUB_NAMESPACE/realsite-backend:$IMAGE_TAG"
                        docker push "$DOCKERHUB_NAMESPACE/realsite-frontend:$IMAGE_TAG"
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG'),
                    string(credentialsId: 'realsite-postgres-password', variable: 'POSTGRES_PASSWORD')
                ]) {
                    sh '''#!/usr/bin/env bash
                        set -euo pipefail

                        kubectl create secret generic realsite-postgres-secret \
                          --namespace "$K8S_NAMESPACE" \
                          --from-literal=POSTGRES_USER=portal \
                          --from-literal=POSTGRES_PASSWORD="$POSTGRES_PASSWORD" \
                          --from-literal=POSTGRES_DB=portal \
                          --dry-run=client -o yaml | kubectl apply -f -

                        kubectl create secret generic realsite-backend-secret \
                          --namespace "$K8S_NAMESPACE" \
                          --from-literal="DATABASE_URL=postgresql+psycopg://portal:$POSTGRES_PASSWORD@postgres:5432/portal" \
                          --from-literal=SMTP_PASSWORD= \
                          --dry-run=client -o yaml | kubectl apply -f -

                        kubectl apply -n "$K8S_NAMESPACE" \
                          -f backend/k8s/configmap.yaml \
                          -f backend/k8s/storage.yaml \
                          -f backend/k8s/postgres.yaml \
                          -f backend/k8s/deployment.yaml \
                          -f backend/k8s/service.yaml \
                          -f frontend/k8s/deployment.yaml \
                          -f frontend/k8s/service.yaml

                        kubectl set image -n "$K8S_NAMESPACE" \
                          deployment/realsite-backend \
                          "realsite-backend=$DOCKERHUB_NAMESPACE/realsite-backend:$IMAGE_TAG"
                        kubectl set image -n "$K8S_NAMESPACE" \
                          deployment/realsite-frontend \
                          "realsite-frontend=$DOCKERHUB_NAMESPACE/realsite-frontend:$IMAGE_TAG"

                        kubectl rollout status -n "$K8S_NAMESPACE" \
                          deployment/realsite-postgres --timeout=180s
                        kubectl rollout status -n "$K8S_NAMESPACE" \
                          deployment/realsite-redis --timeout=180s
                        kubectl rollout status -n "$K8S_NAMESPACE" \
                          deployment/realsite-backend --timeout=240s
                        kubectl rollout status -n "$K8S_NAMESPACE" \
                          deployment/realsite-frontend --timeout=180s
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout >/dev/null 2>&1 || true'
        }
        success {
            echo 'Deployment completed. Use: kubectl port-forward service/realsite-frontend 8090:80'
        }
    }
}
