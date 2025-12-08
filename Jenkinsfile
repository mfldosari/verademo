pipeline {
    agent any
    // This pipeline will require the following Jenkins credentials:
    // 1. registry-creds: Username and Password for Registry
    // 2. registry-hostname: Hostname of the Registry
    // 3. dockerfile-name: Name of the Dockerfile to use for building the image
    // Make sure to set these up in Jenkins before running the pipeline.
    // Also, make sure that the deployment YAML file (verademo-deployment.yaml) is present in the repository.
    // you can customize the deployment YAML file or add your own as per your application's requirements but include the file in the script deployment step to take effect.
    // For security, Jenkins deployment should have access to create secrets and configmaps in the jenkins namespace and enough permissions on target namespaces to deploy applications.
    // So create appropriate RBAC roles and role bindings in your Kubernetes cluster for Jenkins service account and link them to the namespaces where deployments will occur.

    parameters {
        string(name: 'GIT_BRANCH_NAME', defaultValue: 'main', description: 'Git branch name to build from (leave empty to use SCM configuration)')
    }
    
    environment {
        // Application name
        APPLICATION_NAME = 'verademo'

        // Node Port for the service
        NODE_PORT = '30100'

        // Docker Registry credentials from Jenkins
        REGISTRY_CREDS = credentials('registry-creds')
        USERNAME = "${REGISTRY_CREDS_USR}"
        PASSWORD = "${REGISTRY_CREDS_PSW}"
        
        //Hostname from Jenkins credentials
        HOSTNAME = credentials('registry-hostname')
        
        // Dockerfile name from Jenkins credentials
        DOCKERFILE = credentials('dockerfile-name')
        
        IMAGE = "${HOSTNAME}/${APPLICATION_NAME}:${BUILD_NUMBER}"
        _LATEST = "${HOSTNAME}/${APPLICATION_NAME}:latest"
        
        // Convert any Git URL (GitHub, GitLab, Bitbucket, etc.) to git:// protocol
        GIT_REPO = "${env.GIT_URL.replaceAll('https://', 'git://').replaceAll('http://', 'git://')}"
        // Use parameter if provided, otherwise extract from SCM branch
        GIT_BRANCH = "${params.GIT_BRANCH_NAME ?: (env.GIT_BRANCH ? env.GIT_BRANCH.replaceAll('origin/', '') : 'main')}"
    }
    
    stages {
        // Stage one - Checkout code from GitHub
        stage('Checkout') {
            steps {
                echo 'Checking out code from GitHub...'
                checkout scm
                echo 'Code checkout completed'
            }
        }
        // Stage two - Build Docker image using Kaniko
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
        // Stage three - Admin or DevOps Approval for Deployment
        stage('Pre-Deployment Approval') {
            steps {
                script {
                    env.DEPLOY_ENV = input message: 'Approve deployment to production?', 
                          ok: 'Deploy',
                          submitter: 'admin,devops',
                          parameters: [
                              choice(name: 'DEPLOY_ENVIRONMENT', 
                                     choices: ['Production', 'Development', 'Staging'], 
                                     description: 'Select environment to deploy')
                          ]
                }
                echo "${APPLICATION_NAME} Deployment approved to ${env.DEPLOY_ENV}"
            }
        }
        
        // Stage four A - Deploy to Kubernetes cluster
        stage('Production Deployment') {
            when {
                expression { env.DEPLOY_ENV == 'Production' }
            }
            steps {
                echo "Deploying application to ${env.DEPLOY_ENV} namespace..."
                script {
                    sh """
                        cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: ${env.DEPLOY_ENV.toLowerCase()}-${APPLICATION_NAME.toLowerCase()}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APPLICATION_NAME.toLowerCase()}
  namespace: ${env.DEPLOY_ENV.toLowerCase()}-${APPLICATION_NAME.toLowerCase()}
  labels:
    app: ${APPLICATION_NAME.toLowerCase()}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ${APPLICATION_NAME.toLowerCase()}
  template:
    metadata:
      labels:
        app: ${APPLICATION_NAME.toLowerCase()}
    spec:
      containers:
      - name: ${APPLICATION_NAME.toLowerCase()}
        image: ${IMAGE}
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: ${APPLICATION_NAME.toLowerCase()}-service
  namespace: ${env.DEPLOY_ENV.toLowerCase()}-${APPLICATION_NAME.toLowerCase()}
spec:
  type: NodePort
  selector:
    app: ${APPLICATION_NAME.toLowerCase()}
  ports:
  - name: http
    port: 80
    targetPort: 8080
    nodePort: ${NODE_PORT}
EOF
                    """
                }
                echo "Application ${APPLICATION_NAME} deployed successfully to ${env.DEPLOY_ENV}"
            }
        }
        
        // Stage four B - Deploy to development environment
        stage('Development Deployment') {
            when {
                expression { env.DEPLOY_ENV == 'Development'}
            }
            steps {
                echo "Deploying ${APPLICATION_NAME} to ${env.DEPLOY_ENV} namespace..."
                echo "${APPLICATION_NAME} deployed successfully to ${env.DEPLOY_ENV}"
            }
        }
        
        // Stage four C - Deploy to staging environment
        stage('Staging Deployment') {
            when {
                expression { env.DEPLOY_ENV == 'Staging'}
            }
            steps {
                echo "Deploying ${APPLICATION_NAME} to ${env.DEPLOY_ENV} namespace..."
                echo "${APPLICATION_NAME} deployed successfully to ${env.DEPLOY_ENV}"
            }
        }
    }
    // Post actions
    post {
        // Notify on success or failure
        success {
            script {
                echo "Build completed successfully!"
                echo "Docker image: ${IMAGE}"
                echo "Docker Hub image: ${_LATEST}"
                echo "=================================================="
                echo "Deployment Summary for ${env.DEPLOY_ENV} Environment"
                echo "=================================================="
                
                // Get namespace based on deployment
                def namespace = "${env.DEPLOY_ENV.toLowerCase()}-${APPLICATION_NAME.toLowerCase()}"
                
                sh """
                    echo "\n=== NAMESPACE INFO ==="
                    kubectl get namespace ${namespace}
                    
                    echo "\n=== DEPLOYMENTS ==="
                    kubectl get deployments -n ${namespace}
                    
                    echo "\n=== PODS ==="
                    kubectl get pods -n ${namespace} -o wide
                    
                    echo "\n=== SERVICES ==="
                    kubectl get services -n ${namespace}
                    
                    echo "\n=== REPLICASETS ==="
                    kubectl get replicasets -n ${namespace}
                    
                    echo "\n=== ENDPOINTS ==="
                    kubectl get endpoints -n ${namespace}
                    
                    echo "\n=== POD DETAILS ==="
                    kubectl describe pods -n ${namespace}
                    
                    echo "\n=== SERVICE DETAILS ==="
                    kubectl describe services -n ${namespace}
                    
                    echo "\n=== DEPLOYMENT ROLLOUT STATUS ==="
                    kubectl rollout status deployment/${APPLICATION_NAME.toLowerCase()} -n ${namespace}
                """
                
                echo "=================================================="
                echo "Access your application at: http://<node-ip>:${NODE_PORT}"
                echo "=================================================="
            }
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
