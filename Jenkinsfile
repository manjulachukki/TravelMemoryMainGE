pipeline {
    agent any

    environment {
        AWS_REGION       = 'ap-south-1'
        ECR_REGISTRY     = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        BACKEND_REPO     = 'travelmemory-backend'
        FRONTEND_REPO    = 'travelmemory-frontend'
        EKS_CLUSTER_NAME = 'travelmemory-eks'
        IMAGE_TAG        = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Push Docker Images') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                    sh """
                        set -e

                        # Isolate Docker config to avoid shared Jenkins auth conflicts
                        export DOCKER_CONFIG=\$(mktemp -d)
                        trap 'rm -rf \$DOCKER_CONFIG' EXIT

                        # Diagnostic: show which IAM identity Jenkins is using
                        aws sts get-caller-identity

                        # Login to ECR
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}

                        # Build and push backend
                        cd backend
                        docker build -t ${ECR_REGISTRY}/${BACKEND_REPO}:${IMAGE_TAG} .
                        docker push ${ECR_REGISTRY}/${BACKEND_REPO}:${IMAGE_TAG}
                        docker tag ${ECR_REGISTRY}/${BACKEND_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${BACKEND_REPO}:latest
                        docker push ${ECR_REGISTRY}/${BACKEND_REPO}:latest

                        # Build and push frontend
                        cd ../frontend
                        docker build -t ${ECR_REGISTRY}/${FRONTEND_REPO}:${IMAGE_TAG} .
                        docker push ${ECR_REGISTRY}/${FRONTEND_REPO}:${IMAGE_TAG}
                        docker tag ${ECR_REGISTRY}/${FRONTEND_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${FRONTEND_REPO}:latest
                        docker push ${ECR_REGISTRY}/${FRONTEND_REPO}:latest
                    """
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                    sh """
                        aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION}
                        kubectl apply -f k8s/namespace.yml
                        kubectl apply -f k8s/backend-deployment.yml
                        kubectl apply -f k8s/backend-service.yml
                        kubectl apply -f k8s/frontend-deployment.yml
                        kubectl apply -f k8s/frontend-service.yml
                        kubectl apply -f k8s/hpa.yml
                    """
                }
            }
        }

        stage('Deploy Monitoring') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                    sh """
                        kubectl apply -f monitoring/namespace.yml
                        kubectl apply -f monitoring/prometheus-config.yml
                        kubectl apply -f monitoring/prometheus-deployment.yml
                        kubectl apply -f monitoring/grafana-deployment.yml
                    """
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                    sh """
                        kubectl rollout status deployment/backend -n travelmemory --timeout=120s
                        kubectl rollout status deployment/frontend -n travelmemory --timeout=120s
                        kubectl get pods -n travelmemory
                        kubectl get svc -n travelmemory
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
            echo 'Pipeline failed. Check logs for details.'
        }
        always {
            cleanWs()
        }
    }
}
