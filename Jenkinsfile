pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        AWS_REGION = 'ap-south-1'
        AWS_ACCOUNT_ID = '545931885961'
        ECR_REGISTRY = '545931885961.dkr.ecr.ap-south-1.amazonaws.com'
        IMAGE_TAG = '1.0.1'
    }

    stages {
        stage('Verify Source') {
            steps {
                sh 'git rev-parse --short HEAD'
                sh 'docker --version'
                sh 'aws --version'
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    docker build -t streaming-auth:${IMAGE_TAG} ./backend/authService
                    docker build -f ./backend/streamingService/Dockerfile -t streaming-stream:${IMAGE_TAG} ./backend
                    docker build -t streaming-admin:${IMAGE_TAG} ./backend/adminService
                    docker build -t streaming-chat:${IMAGE_TAG} ./backend/chatService
                    docker build \
                      --build-arg REACT_APP_AUTH_API_URL=/api/auth \
                      --build-arg REACT_APP_STREAMING_API_URL=/api/streaming \
                      --build-arg REACT_APP_ADMIN_API_URL=/api/admin \
                      --build-arg REACT_APP_CHAT_API_URL=/api/chat \
                      -t streaming-frontend:${IMAGE_TAG} ./frontend
                '''
            }
        }

        stage('Push Images to ECR') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'streamingapp-ecr-credentials',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''
                        set +x
                        export AWS_DEFAULT_REGION="${AWS_REGION}"

                        aws ecr get-login-password --region "${AWS_REGION}" \
                          | docker login --username AWS --password-stdin "${ECR_REGISTRY}"

                        for service in auth stream admin chat frontend; do
                            docker tag "streaming-${service}:${IMAGE_TAG}" \
                              "${ECR_REGISTRY}/streaming-${service}:${IMAGE_TAG}"

                            docker push "${ECR_REGISTRY}/streaming-${service}:${IMAGE_TAG}"
                        done

                        docker logout "${ECR_REGISTRY}"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'All five StreamingApp images pushed to AWS ECR successfully.'
        }
        failure {
            echo 'Pipeline failed. Review the failing stage in Console Output.'
        }
    }
}
