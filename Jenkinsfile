pipeline {
    agent any

    stages {
        stage('Verify GitHub Checkout') {
            steps {
                echo 'StreamingApp Jenkins CI/CD pipeline started'
                sh 'git --version'
                sh 'git rev-parse --short HEAD'
            }
        }

        stage('Check Build Tools') {
            steps {
                sh '''
                    echo "Checking Docker..."
                    if command -v docker >/dev/null 2>&1; then
                        docker --version
                        docker info >/dev/null 2>&1 && echo "Docker daemon accessible" || echo "Docker daemon NOT accessible"
                    else
                        echo "Docker NOT installed"
                    fi

                    echo "Checking AWS CLI..."
                    if command -v aws >/dev/null 2>&1; then
                        aws --version
                    else
                        echo "AWS CLI NOT installed"
                    fi
                '''
            }
        }
    }
}
