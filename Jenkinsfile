pipeline {
    agent any

    environment {
        AWS_REGION       = 'ap-south-1'
        AWS_ACCOUNT_ID   = '975050024946'
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
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials-groupe']]) {
                    sh """
                        set -e

                        # Isolate Docker config to avoid shared Jenkins auth conflicts
                        export DOCKER_CONFIG=\$(mktemp -d)
                        trap 'rm -rf \$DOCKER_CONFIG' EXIT

                        # Verify correct IAM identity
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
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials-groupe']]) {
                    sh """
                        aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION}

                        # Update image tags to use this build number
                        sed -i 's|travelmemory-backend:latest|travelmemory-backend:${IMAGE_TAG}|g' k8s/backend-deployment.yml
                        sed -i 's|travelmemory-frontend:latest|travelmemory-frontend:${IMAGE_TAG}|g' k8s/frontend-deployment.yml

                        # Deploy backend first (service + deployment)
                        kubectl apply -f k8s/namespace.yml
                        kubectl apply -f k8s/backend-deployment.yml
                        kubectl apply -f k8s/backend-service.yml

                        # Wait for backend LoadBalancer to get an external hostname
                        echo "Waiting for backend LoadBalancer..."
                        for i in \$(seq 1 30); do
                            BACKEND_HOST=\$(kubectl get svc backend-service -n travelmemory -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null)
                            if [ -n "\$BACKEND_HOST" ]; then
                                echo "Backend LB hostname: \$BACKEND_HOST"
                                break
                            fi
                            sleep 10
                        done

                        # Update frontend deployment with the real backend URL
                        if [ -n "\$BACKEND_HOST" ]; then
                            sed -i "s|http://backend-service:3001|http://\${BACKEND_HOST}:3001|g" k8s/frontend-deployment.yml
                        fi

                        kubectl apply -f k8s/frontend-deployment.yml
                        kubectl apply -f k8s/frontend-service.yml
                        kubectl apply -f k8s/hpa.yml
                    """
                }
            }
        }

        stage('Deploy Monitoring') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials-groupe']]) {
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
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials-groupe']]) {
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
