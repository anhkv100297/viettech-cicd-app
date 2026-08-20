pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
    }

    environment {
        AWS_ACCOUNT_ID = '042134552922'
        AWS_REGION     = 'ap-southeast-1'
        ECR_REGISTRY   = '042134552922.dkr.ecr.ap-southeast-1.amazonaws.com'
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
                    test -f k8s/deployment.yaml
                    test -f k8s/service.yaml

                    echo "Application files OK"
                '''
            }
        }

        stage('Login ECR') {
            steps {
                sh '''
                    aws ecr get-login-password \
                      --region ${AWS_REGION} | \
                    docker login \
                      --username AWS \
                      --password-stdin \
                      ${ECR_REGISTRY}
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

        stage('Refresh ECR Pull Secret') {
            steps {
                sh '''
                    ECR_PASSWORD=$(aws ecr get-login-password \
                      --region ${AWS_REGION})

                    kubectl create secret docker-registry ecr-secret \
                      --docker-server=${ECR_REGISTRY} \
                      --docker-username=AWS \
                      --docker-password="${ECR_PASSWORD}" \
                      --dry-run=client \
                      -o yaml | kubectl apply -f -
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    kubectl apply -f k8s/service.yaml

                    if kubectl get deployment viettech-app \
                      >/dev/null 2>&1; then

                        echo "Deployment already exists"
                        echo "Updating image to ${ECR_REPO}:${IMAGE_TAG}"

                        kubectl set image \
                          deployment/viettech-app \
                          viettech-app=${ECR_REPO}:${IMAGE_TAG}

                    else

                        echo "First deployment"

                        sed "s|${ECR_REPO}:latest|${ECR_REPO}:${IMAGE_TAG}|g" \
                          k8s/deployment.yaml | kubectl apply -f -

                    fi

                    kubectl rollout status \
                      deployment/viettech-app \
                      --timeout=180s
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "===== NODES ====="
                    kubectl get nodes -o wide

                    echo "===== PODS ====="
                    kubectl get pods -o wide

                    echo "===== SERVICE ====="
                    kubectl get svc viettech-app-service
                '''
            }
        }
    }

    post {

        success {
            echo "CI/CD completed successfully"
            echo "Docker image: ${ECR_REPO}:${IMAGE_TAG}"
        }

        failure {
            echo "CI/CD pipeline failed. Check the failed stage."
        }

        always {
            sh '''
                docker image rm ${ECR_REPO}:${IMAGE_TAG} || true
                docker image rm ${ECR_REPO}:latest || true
                docker image prune -f || true
            '''
        }
    }
}