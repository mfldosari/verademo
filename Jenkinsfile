pipeline {
    agent any
    
    parameters {
        password(name: 'DOCKERHUB_PASSWORD', defaultValue: '', description: 'Docker Hub Password')
    }
    
    environment {
        DOCKERHUB_USERNAME = "mfldosari"
        DOCKER_IMAGE = "${DOCKERHUB_USERNAME}/verademo:${BUILD_NUMBER}"
        DOCKER_LATEST = "${DOCKERHUB_USERNAME}/verademo:latest"
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code from GitHub...'
                checkout scm
                echo 'Code checkout completed'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                echo "Building ${DOCKER_IMAGE}"
                script {
                    sh "docker build -f Dockerfile.simple -t ${DOCKER_IMAGE} -t ${DOCKER_LATEST} ."
                }
                echo 'Docker image built successfully'
            }
        }
        
        stage('List Images') {
            steps {
                echo 'Listing Docker images...'
                sh 'docker images | grep verademo'
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                echo 'Pushing image to Docker Hub...'
                script {
                    sh '''
                        echo "$DOCKERHUB_PASSWORD" | docker login -u "$DOCKERHUB_USERNAME" --password-stdin
                        docker push ${DOCKER_IMAGE}
                        docker push ${DOCKER_LATEST}
                        docker logout
                    '''
                }
                echo 'Image pushed successfully to Docker Hub'
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
                                     choices: ['Production', 'development', 'staging'], 
                                     description: 'Select environment to deploy')
                          ]
                }
                echo 'Deployment approved by admin'
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                echo 'Deploying application to prod-env namespace...'
                script {
                    sh '''
                        kubectl apply -f verademo-deployment.yaml
                        kubectl rollout status deployment/verademo -n prod-env --timeout=2m
                    '''
                }
                echo 'Application deployed successfully to Kubernetes'
            }
        }
    }
    
    post {
        success {
            echo "Build completed successfully!"
            echo "Docker image: ${DOCKER_IMAGE}"
            echo "Docker Hub image: ${DOCKER_LATEST}"
            echo "Application deployed to prod-env!"
            echo "Access the application at: http://<node-ip>:30100"
        }
        failure {
            echo 'Build failed!'
        }
        always {
            echo 'Pipeline execution completed.'
        }
    }
}
