pipeline {
    agent any

    environment {
        NAMESPACE = "chat-app"

        BACKEND_IMAGE = "dp04/chatapp-backend"
        FRONTEND_IMAGE = "dp04/chatapp-frontend"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Dharmik6096/chat-app.git'
            }
        }

        stage('Build Images') {
            steps {
                sh """
                    docker build -t ${BACKEND_IMAGE}:${IMAGE_TAG} ./backend
                    docker build -t ${FRONTEND_IMAGE}:${IMAGE_TAG} ./frontend
                """
            }
        }

        stage('Push Images') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                        docker push ${BACKEND_IMAGE}:${IMAGE_TAG}
                        docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Deploy Base Kubernetes Files') {
            steps {
                sh """
                    kubectl apply -f kubernetes/namespace.yml

                    kubectl apply -f kubernetes/mongodb/secrets.yml
                    kubectl apply -f kubernetes/mongodb/mongodb-pv.yml
                    kubectl apply -f kubernetes/mongodb/mongodb-pvc.yml
                    kubectl apply -f kubernetes/mongodb/mongodb-deployment.yml
                    kubectl apply -f kubernetes/mongodb/mongodb-service.yml

                    kubectl apply -f kubernetes/backend/backend-secret.yml
                    kubectl apply -f kubernetes/backend/backend-service.yml
                    kubectl apply -f kubernetes/backend/backend-deployment.yml

                    kubectl apply -f kubernetes/frontend/frontend-service.yml
                    kubectl apply -f kubernetes/frontend/frontend-deployment.yml

                    kubectl apply -f kubernetes/ingress.yml
                """
            }
        }

        stage('Update Images') {
            steps {
                sh """
                    kubectl set image deployment/chatapp-backend-deployment backend=${BACKEND_IMAGE}:${IMAGE_TAG} -n ${NAMESPACE}
                    kubectl set image deployment/frontend-chatapp-deployment frontend=${FRONTEND_IMAGE}:${IMAGE_TAG} -n ${NAMESPACE}
                """
            }
        }

        stage('Verify Deployment') {
            steps {
                sh """
                    kubectl rollout status deployment/chatapp-mongodb-deployment -n ${NAMESPACE} --timeout=180s
                    kubectl rollout status deployment/chatapp-backend-deployment -n ${NAMESPACE} --timeout=180s
                    kubectl rollout status deployment/frontend-chatapp-deployment -n ${NAMESPACE} --timeout=180s
                """
            }
        }

        stage('Health Check') {
            steps {
                sh """
                    kubectl get pods -n ${NAMESPACE}
                    kubectl get svc -n ${NAMESPACE}
                    kubectl get ingress -n ${NAMESPACE}
                    
                # As a local terminal use it     
               #    nohup kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 82:80 &
                """
            }
        }
    }

    post {
        success {
            echo "CI/CD completed successfully."
        }

        failure {
            echo "CI/CD failed. Rolling back backend and frontend..."
            sh """
                kubectl rollout undo deployment/chatapp-backend-deployment -n ${NAMESPACE} || true
                kubectl rollout undo deployment/frontend-chatapp-deployment -n ${NAMESPACE} || true
            """
        }
    }
}
