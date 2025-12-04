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
                echo 'Code checkout completed'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                echo "Building ${DOCKER_IMAGE}"
                echo 'Docker image built successfully'
            }
        }
        
        stage('List Images') {
            steps {
                echo 'Listing Docker images...'
                echo "Image: ${DOCKER_IMAGE}"
                echo "Image: ${DOCKER_LATEST}"
            }
        }
        
        stage('Pre-Deployment Approval') {
            steps {
                script {
                    input message: 'Approve deployment to production?', 
                          ok: 'Deploy',
                          submitter: 'admin,devops',
                          parameters: [
                              choice(name: 'DEPLOY_ENVIRONMENT', 
                                     choices: ['Production', 'Staging'], 
                                     description: 'Select environment to deploy')
                          ]
                }
                echo 'Deployment approved by admin'
            }
        }
        
        stage('Deploy') {
            steps {
                echo 'Deploying application...'
                echo 'Application deployed successfully'
            }
        }
    }
    
    post {
        success {
            echo "Build completed successfully!"
            echo "Docker image: ${DOCKER_IMAGE}"
            echo "CD is deployed!"
        }
        failure {
            echo 'Build failed!'
        }
        always {
            echo 'Pipeline execution completed.'
        }
    }
}
