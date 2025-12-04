pipeline {
    agent any
    
    parameters {
        password(name: 'NEXUS_PASSWORD', defaultValue: '1234', description: 'Nexus repository password')
    }
    
    environment {
        DOCKER_IMAGE = "verademo:${BUILD_NUMBER}"
        DOCKER_LATEST = "verademo:latest"
        NEXUS_REGISTRY = "10.43.158.166:8082"
        NEXUS_IMAGE = "${NEXUS_REGISTRY}/verademo:latest"
        NEXUS_PASSWORD = "${params.NEXUS_PASSWORD}"
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
                    sh "docker build -f Dockerfile.simple -t ${DOCKER_IMAGE} ."
                    sh "docker tag ${DOCKER_IMAGE} ${DOCKER_LATEST}"
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
        
        stage('Push to Nexus') {
            steps {
                echo 'Pushing image to Nexus repository...'
                script {
                    sh '''
                        echo "${NEXUS_PASSWORD}" | docker login ${NEXUS_REGISTRY} -u admin --password-stdin
                        docker tag ${DOCKER_LATEST} ${NEXUS_IMAGE}
                        docker push ${NEXUS_IMAGE}
                    '''
                }
                echo 'Image pushed to Nexus successfully'
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
            echo "CD is deployed!"
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
