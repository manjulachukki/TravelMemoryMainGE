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
            parallel {
                stage('Backend') {
                    steps {
                        dir('backend') {
                            sh """
                                aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                                docker build -t ${ECR_REGISTRY}/${BACKEND_REPO}:${IMAGE_TAG} .
                                docker push ${ECR_REGISTRY}/${BACKEND_REPO}:${IMAGE_TAG}
                                docker tag ${ECR_REGISTRY}/${BACKEND_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${BACKEND_REPO}:latest
                                docker push ${ECR_REGISTRY}/${BACKEND_REPO}:latest
                            """
                        }
                    }
                }
                stage('Frontend') {
                    steps {
                        dir('frontend') {
                            sh """
                                aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                                docker build -t ${ECR_REGISTRY}/${FRONTEND_REPO}:${IMAGE_TAG} .
                                docker push ${ECR_REGISTRY}/${FRONTEND_REPO}:${IMAGE_TAG}
                                docker tag ${ECR_REGISTRY}/${FRONTEND_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${FRONTEND_REPO}:latest
                                docker push ${ECR_REGISTRY}/${FRONTEND_REPO}:latest
                            """
                        }
                    }
                }
            }
        }

        stage('Terraform - Provision Infrastructure') {
            steps {
                dir('terraform') {
                    sh """
                        terraform init
                        terraform plan -out=tfplan
                        terraform apply -auto-approve tfplan
                    """
                }
            }
        }

        stage('Ansible - Configure Infrastructure') {
            steps {
                dir('ansible') {
                    sh """
                        ansible-galaxy install -r requirements.yml
                        ansible-playbook playbooks/site.yml
                    """
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
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

        stage('Deploy Monitoring') {
            steps {
                sh """
                    kubectl apply -f monitoring/namespace.yml
                    kubectl apply -f monitoring/prometheus-config.yml
                    kubectl apply -f monitoring/prometheus-deployment.yml
                    kubectl apply -f monitoring/grafana-deployment.yml
                """
            }
        }

        stage('Verify Deployment') {
            steps {
                sh """
                    kubectl rollout status deployment/backend -n travelmemory --timeout=120s
                    kubectl rollout status deployment/frontend -n travelmemory --timeout=120s
                    kubectl get pods -n travelmemory
                    kubectl get svc -n travelmemory
                """
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
