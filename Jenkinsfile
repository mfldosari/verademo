pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = "verademo:${BUILD_NUMBER}"
        DOCKER_LATEST = "verademo:latest"
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code from GitHub...'
                checkout scm
            }
        }
        
        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                script {
                    docker.build(DOCKER_IMAGE)
                    docker.build(DOCKER_LATEST)
                }
            }
        }
        
        stage('List Images') {
            steps {
                echo 'Listing Docker images...'
                sh 'docker images | grep verademo'
            }
        }
    }
    
    post {
        success {
            echo "Build completed successfully!"
            echo "Docker image: ${DOCKER_IMAGE}"
        }
        failure {
            echo 'Build failed!'
        }
        always {
            echo 'Pipeline execution completed.'
        }
    }
}
