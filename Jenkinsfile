pipeline {
    agent any
    
    parameters {
        password(name: 'DOCKERHUB_PASSWORD', defaultValue: '', description: 'Docker Hub Password')
    }
    
    environment {
        DOCKERHUB_USERNAME = "mfaldosari"
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
        
        stage('Build Docker Image with Kaniko') {
            steps {
                echo 'Building Docker image with Kaniko...'
                echo "Building ${DOCKER_IMAGE}"
                script {
                    // Create Docker config secret for Kaniko
                    sh """
                        kubectl create secret generic docker-credentials \\
                          --from-literal=username=${DOCKERHUB_USERNAME} \\
                          --from-literal=password=${DOCKERHUB_PASSWORD} \\
                          --namespace=jenkins \\
                          --dry-run=client -o yaml | kubectl apply -f -
                        
                        # Create configmap with Docker credentials
                        echo '{"auths":{"https://index.docker.io/v1/":{"auth":"'"\$(echo -n ${DOCKERHUB_USERNAME}:${DOCKERHUB_PASSWORD} | base64)"'"}}}' > /tmp/config.json
                        kubectl create configmap kaniko-docker-config \\
                          --from-file=/tmp/config.json \\
                          --namespace=jenkins \\
                          --dry-run=client -o yaml | kubectl apply -f -
                        rm /tmp/config.json
                    """
                    
                    // Run Kaniko build
                    sh """
                        kubectl run kaniko-build-${BUILD_NUMBER} \\
                          --restart=Never \\
                          --image=gcr.io/kaniko-project/executor:latest \\
                          --namespace=jenkins \\
                          --overrides='{
                            "spec": {
                              "containers": [{
                                "name": "kaniko",
                                "image": "gcr.io/kaniko-project/executor:latest",
                                "args": [
                                  "--context=git://github.com/mfldosari/verademo.git#refs/heads/main",
                                  "--dockerfile=Dockerfile.simple",
                                  "--destination=${DOCKER_IMAGE}",
                                  "--destination=${DOCKER_LATEST}"
                                ],
                                "volumeMounts": [{
                                  "name": "docker-config",
                                  "mountPath": "/kaniko/.docker"
                                }]
                              }],
                              "volumes": [{
                                "name": "docker-config",
                                "configMap": {
                                  "name": "kaniko-docker-config"
                                }
                              }],
                              "restartPolicy": "Never"
                            }
                          }' \\
                          --attach=true \\
                          --rm
                    """
                }
                echo 'Docker image built and pushed successfully with Kaniko'
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
            // Cleanup Kaniko resources
            script {
                sh """
                    kubectl delete configmap kaniko-docker-config --namespace=jenkins --ignore-not-found=true
                    kubectl delete secret docker-credentials --namespace=jenkins --ignore-not-found=true
                """ 
            }
        }
    }
}
