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
        DEV_NODE_PORT = credentials('DEV_NODE_PORT') 
        PROD_NODE_PORT = credentials('PROD_NODE_PORT')
        
        // Use pre-built VeraDemo image from Docker Hub
        USE_PREBUILT_IMAGE = 'true'
        PREBUILT_IMAGE = 'antfie/verademo:latest'

        // Docker Registry credentials from Jenkins
        REGISTRY_CREDS = credentials('registry-creds')
        USERNAME = "${REGISTRY_CREDS_USR}"
        PASSWORD = "${REGISTRY_CREDS_PSW}"
        
        //Hostname from Jenkins credentials
        HOSTNAME = credentials('registry-hostname')
        
        // Dockerfile name from Jenkins credentials
        DOCKERFILE = credentials('dockerfile-name')
        
        IMAGE = "${HOSTNAME}/${APPLICATION_NAME}:v${BUILD_NUMBER}"
        _LATEST = "${HOSTNAME}/${APPLICATION_NAME}:latest"
        
        // Convert any Git URL (GitHub, GitLab, Bitbucket, etc.) to git:// protocol
        GIT_REPO = "${env.GIT_URL.replaceAll('https://', 'git://').replaceAll('http://', 'git://')}"
        // Use parameter if provided, otherwise extract from SCM branch
        GIT_BRANCH = "${params.GIT_BRANCH_NAME ?: (env.GIT_BRANCH ? env.GIT_BRANCH.replaceAll('origin/', '') : 'main')}"
    }
    
    stages {

        // // Stage one - Checkout code from GitHub
        // stage('Checkout') {
        //     steps {
        //         echo 'Checking out code from GitHub...'
        //         checkout scm
        //         echo 'Code checkout completed'
        //     }
        // }
        stage('registry credentials setup') {
            steps {
                sh '''
                  kubectl create secret docker-registry registry-config \
                    --docker-server="$HOSTNAME" \
                    --docker-username="$USERNAME" \
                    --docker-password="$PASSWORD" \
                    --namespace=jenkins \
                    --dry-run=client -o yaml | kubectl apply -f -
                '''
            }
        }
        // since docker is installed we can run docker build directly or use kaniko
        // // Stage two - Build Docker image using Kaniko
        // stage('Build Docker Image with Kaniko') {
        //     when {
        //         expression { USE_PREBUILT_IMAGE != 'true' }
        //     }
        //     steps {
        //         echo 'Building Docker image with Kaniko...'
        //         echo "Building ${IMAGE}"
        //         script {
        //             // Run Kaniko build
        //             sh """
        //                 kubectl run kaniko-build-${BUILD_NUMBER} \\
        //                   --restart=Never \\
        //                   --image=gcr.io/kaniko-project/executor:latest \\
        //                   --namespace=jenkins \\
        //                   --overrides='{
        //                     "spec": {
        //                       "containers": [{
        //                         "name": "kaniko",
        //                         "image": "gcr.io/kaniko-project/executor:latest",
        //                         "args": [
        //                           "--context=${GIT_REPO}#refs/heads/${GIT_BRANCH}",
        //                           "--dockerfile=${DOCKERFILE}",
        //                           "--destination=${IMAGE}",
        //                           "--destination=${_LATEST}",
        //                           "--skip-tls-verify"
        //                         ],
        //                         "volumeMounts": [{
        //                           "name": "docker-config",
        //                           "mountPath": "/kaniko/.docker"
        //                         }]
        //                       }],
        //                       "volumes": [{
        //                         "name": "docker-config",
        //                         "secret": {
        //                           "secretName": "registry-config"
        //                         }
        //                       }],
        //                       "restartPolicy": "Never"
        //                     }
        //                   }' \\
        //                   --attach=true \\
        //                   --rm
        //             """
        //         }
        //         echo 'Docker image built and pushed successfully with Kaniko'
        //     }
        // }
        // Stage three - Admin or DevOps Approval for Deployment
        stage('Pre-Deployment Approval') {
            steps {
                script {
                    env.DEPLOY_ENV = input message: 'Approve deployment to production?', 
                          ok: 'Deploy',
                          submitter: 'admin,devops',
                          parameters: [
                              choice(name: 'DEPLOY_ENVIRONMENT', 
                                     choices: ['Prod [disabled]', 'Dev', 'Staging [disabled]', 'Cancel'], 
                                     description: 'Select environment to deploy')
                          ]
                }
                echo "${APPLICATION_NAME} Deployment approved to ${env.DEPLOY_ENV}"
            }
        }
        
        // Stage four A - Deploy to Kubernetes cluster
        stage('Production Deployment') {
            when {
                expression { env.DEPLOY_ENV == 'Prod' }
            }
            steps {
                echo "Deploying application to ${env.DEPLOY_ENV} namespace..."
            }
        }
        
        // Stage four B - Deploy to development environment
        stage('Development Deployment') {
            when {
                expression { env.DEPLOY_ENV == 'Dev'}
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
        image: ${USE_PREBUILT_IMAGE == 'true' ? PREBUILT_IMAGE : IMAGE}
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        env:
        - name: MYSQL_DATABASE
          value: "blab"
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
    port: 8080
    targetPort: 8080
    nodePort: ${DEV_NODE_PORT}
EOF
                    """
                }
                echo "Application ${APPLICATION_NAME} deployed successfully to ${env.DEPLOY_ENV}"
            }
        }
        
        // Stage four C - Deploy to staging environment
        stage('Staging Deployment') {
            when {
                expression { env.DEPLOY_ENV == 'Staging'}
            }
            steps {
                echo "Deploying application to ${env.DEPLOY_ENV} namespace..."
            }
        }
        stage('Cancel Deployment') {
            when {
                expression { env.DEPLOY_ENV == 'Cancel' }
            }
            steps {
                echo 'Deployment has been cancelled by the approver.'
            }
        }
    }
    // Post actions
    post {
        // Notify on success or failure
        success {
            script {
                echo "Build completed successfully!"
                echo "Deployed Image: ${USE_PREBUILT_IMAGE == 'true' ? PREBUILT_IMAGE : IMAGE} to ${env.DEPLOY_ENV} environment."
            }
        }
        failure {
            echo 'Build failed!'
        }
        always {
            echo 'Pipeline execution completed.'
            // Cleanup Kaniko resources
            // script {
            //     sh """
            //         kubectl delete configmap kaniko-registry-config --namespace=jenkins --ignore-not-found=true
            //     """ 
            // }
        }
    }

}
