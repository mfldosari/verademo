pipeline {
    agent any
    
    parameters {
        string(name: 'REGISTRY_URL', defaultValue: 'https://index.docker.io/v1/', description: 'Registry URL (e.g., https://my-registry.com)')
        string(name: 'USERNAME', defaultValue: 'mfaldosari', description: 'Registry Username')
        string(name: 'HOSTNAME', defaultValue: '', description: 'Registry Image Hostname')
        password(name: 'PASSWORD', defaultValue: '', description: 'Registry Password')
        string(name: 'DOCKERFILE', defaultValue: 'Dockerfile.simple', description: 'Dockerfile name (e.g., Dockerfile, Dockerfile.simple)')
        string(name: 'GIT_BRANCH_NAME', defaultValue: '', description: 'Git branch name to build from (leave empty to use SCM configuration)')
    }
    
    environment {
        USERNAME = "${USERNAME}"
        IMAGE = "${HOSTNAME}/verademo:${BUILD_NUMBER}"
        _LATEST = "${HOSTNAME}/verademo:latest"
        // Convert any Git URL (GitHub, GitLab, Bitbucket, etc.) to git:// protocol
        GIT_REPO = "${env.GIT_URL.replaceAll('https://', 'git://').replaceAll('http://', 'git://')}"
        // Use parameter if provided, otherwise extract from SCM branch
        GIT_BRANCH = "${params.GIT_BRANCH_NAME ?: (env.GIT_BRANCH ? env.GIT_BRANCH.replaceAll('origin/', '') : 'main')}"
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
                echo "Building ${IMAGE}"
                script {
                    // Create config secret for Kaniko with custom registry
                    sh """
                        kubectl create secret generic registry-credentials \\
                          --from-literal=username=${USERNAME} \\
                          --from-literal=password=${PASSWORD} \\
                          --namespace=jenkins \\
                          --dry-run=client -o yaml | kubectl apply -f -
                        
                        # Create configmap with registry credentials for custom registry
                        echo '{"auths":{"${HOSTNAME}":{"auth":"'"\$(echo -n ${USERNAME}:${PASSWORD} | base64)"'"}}}' > /tmp/config.json
                        kubectl create configmap kaniko-registry-config \\
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
                                  "--context=${GIT_REPO}#refs/heads/${GIT_BRANCH}",
                                  "--dockerfile=${DOCKERFILE}",
                                  "--destination=${IMAGE}",
                                  "--destination=${_LATEST}",
                                  "--skip-tls-verify"
                                ],
                                "volumeMounts": [{
                                  "name": "docker-config",
                                  "mountPath": "/kaniko/.docker"
                                }]
                              }],
                              "volumes": [{
                                "name": "docker-config",
                                "configMap": {
                                  "name": "kaniko-registry-config"
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
                    env.DEPLOY_ENV = input message: 'Approve deployment to production?', 
                          ok: 'Deploy',
                          submitter: 'admin,devops',
                          parameters: [
                              choice(name: 'DEPLOY_ENVIRONMENT', 
                                     choices: ['Production', 'development', 'staging'], 
                                     description: 'Select environment to deploy')
                          ]
                }
                echo "Deployment approved for ${env.DEPLOY_ENV}"
            }
        }
        
        stage('Deploy to Kubernetes') {
            when {
                expression { env.DEPLOY_ENV == 'Production' }
            }
            steps {
                echo 'Deploying application to prod-env namespace...'
                script {
                    sh """
                        kubectl apply -f verademo-deployment.yaml
                        kubectl set image deployment/verademo verademo=${IMAGE} -n prod-env
                    """
                }
                echo 'Application deployed successfully to Kubernetes'
            }
        }
    }
    
    post {
        success {
            echo "Build completed successfully!"
            echo "Docker image: ${IMAGE}"
            echo "Docker Hub image: ${_LATEST}"
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
                    kubectl delete configmap kaniko-registry-config --namespace=jenkins --ignore-not-found=true
                    kubectl delete secret registry-credentials --namespace=jenkins --ignore-not-found=true
                """ 
            }
        }
    }
}