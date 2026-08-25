pipeline {
    agent any

    environment {
        DOCKERHUB_CREDS     = credentials('Dockerhub')
        DOCKERHUB_USERNAME  = "${DOCKERHUB_CREDS_USR}"
        BACKEND_URL         = "http://13.201.117.17:8000"
        IMAGE_TAG           = "latest"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Ankit-Mori1626/K8S-Project.git'
            }
        }

        stage('Docker Login') {
            steps {
                sh 'echo $DOCKERHUB_CREDS_PSW | docker login -u $DOCKERHUB_CREDS_USR --password-stdin'
            }
        }

        stage('Build Backend') {
            steps {
                sh "docker build -t ${DOCKERHUB_USERNAME}/myapp-backend:${IMAGE_TAG} ./backend"
            }
        }

        stage('Build Frontend') {
            steps {
                sh """
                docker build --no-cache -t ${DOCKERHUB_USERNAME}/myapp-frontend:${IMAGE_TAG} \
                  --build-arg REACT_APP_API_URL=${BACKEND_URL} \
                  ./frontend
                """
            }
        }

        stage('Push Images') {
            steps {
                sh """
                docker push ${DOCKERHUB_USERNAME}/myapp-backend:${IMAGE_TAG}
                docker push ${DOCKERHUB_USERNAME}/myapp-frontend:${IMAGE_TAG}
                """
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-cred', variable: 'KUBECONFIG_FILE')]) {
                    sh """
                    export KUBECONFIG=\$KUBECONFIG_FILE
                    sed -i "s/DOCKERHUB_USERNAME/${DOCKERHUB_USERNAME}/g" k8s/*.yaml

                    kubectl apply -f k8s/namespace.yaml
                    kubectl apply -f k8s/secrets.yaml
                    kubectl apply -f k8s/configmap.yaml
                    kubectl apply -f k8s/backend-deployment.yaml
                    kubectl apply -f k8s/backend-service.yaml
                    kubectl apply -f k8s/frontend-deployment.yaml
                    kubectl apply -f k8s/frontend-service.yaml

                    kubectl rollout restart deployment backend -n myapp
                    kubectl rollout restart deployment frontend -n myapp
                    """
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout'
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
