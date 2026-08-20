pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = '042134552922'
        AWS_REGION     = 'ap-southeast-1'
        ECR_REPO       = '042134552922.dkr.ecr.ap-southeast-1.amazonaws.com/viettech-app'
        IMAGE_TAG      = "${BUILD_NUMBER}"
        KUBECONFIG     = '/var/lib/jenkins/.kube/config'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Test') {
            steps {
                sh '''
                    test -f index.html
                    test -f Dockerfile
                    test -f css/style.css
                    test -f js/main.js

                    echo "Application files OK"
                '''
            }
        }

        stage('Login ECR') {
            steps {
                sh '''
                    aws ecr get-login-password --region ${AWS_REGION} | \
                    docker login \
                    --username AWS \
                    --password-stdin \
                    ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                    -t ${ECR_REPO}:${IMAGE_TAG} \
                    -t ${ECR_REPO}:latest \
                    .
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                sh '''
                    docker push ${ECR_REPO}:${IMAGE_TAG}
                    docker push ${ECR_REPO}:latest
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    kubectl apply -f k8s/service.yaml

                    if kubectl get deployment viettech-app >/dev/null 2>&1; then
                        kubectl set image deployment/viettech-app \
                          viettech-app=${ECR_REPO}:${IMAGE_TAG}
                    else
                        sed "s|:latest|:${IMAGE_TAG}|g" \
                          k8s/deployment.yaml | kubectl apply -f -
                    fi

                    kubectl rollout status deployment/viettech-app \
                      --timeout=180s
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    kubectl get pods -o wide
                    kubectl get svc viettech-app-service
                '''
            }
        }
    }

    post {
        success {
            echo "CI/CD completed successfully - image tag: ${IMAGE_TAG}"
        }

        failure {
            echo 'CI/CD pipeline failed. Check the stage logs.'
        }

        always {
            sh 'docker image prune -f || true'
        }
    }
}