pipeline {
    agent any

    parameters {
        string(name: 'GIT_BRANCH_NAME', defaultValue: 'main', description: 'Git branch name to build from')
    }

    environment {
        APPLICATION_NAME = 'verademo'
        
        // Node Port for the service
        DEV_NODE_PORT = credentials('DEV_NODE_PORT') 
        PROD_NODE_PORT = credentials('PROD_NODE_PORT')
        
        USE_PREBUILT_IMAGE = 'false'
        PREBUILT_IMAGE = 'antfie/verademo:latest'

        // Docker Registry credentials
        REGISTRY_CREDS = credentials('registry-creds')
        USERNAME = "${REGISTRY_CREDS_USR}"
        PASSWORD = "${REGISTRY_CREDS_PSW}"
        
        HOSTNAME = credentials('registry-hostname')
        DOCKERFILE = credentials('dockerfile-name')
        
        IMAGE = "${HOSTNAME}/${APPLICATION_NAME}:v${BUILD_NUMBER}"
        _LATEST = "${HOSTNAME}/${APPLICATION_NAME}:latest"
        
        GIT_REPO = "${env.GIT_URL.replaceAll('https://', 'git://').replaceAll('http://', 'git://')}"
        GIT_BRANCH = "${params.GIT_BRANCH_NAME ?: (env.GIT_BRANCH ? env.GIT_BRANCH.replaceAll('origin/', '') : 'main')}"
    }

    stages {
        stage('registry credentials setup') {
            steps {
                sh """
                kubectl create secret docker-registry registry-config \
                  --docker-server="${HOSTNAME}" \
                  --docker-username="${USERNAME}" \
                  --docker-password="${PASSWORD}" \
                  --namespace=jenkins \
                  --dry-run=client -o yaml | kubectl apply -f -
                """
            }
        }

        stage('Build Docker Image with Kaniko') {
            when {
                expression { USE_PREBUILT_IMAGE != 'true' }
            }
            steps {
                echo "Building ${IMAGE} with Kaniko..."
                script {
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
                            "secret": {
                              "secretName": "registry-config",
                              "items": [{
                                "key": ".dockerconfigjson",
                                "path": "config.json"
                              }]
                            }
                          }],
                          "restartPolicy": "Never"
                        }
                      }' \\
                      --attach=true \\
                      --rm
                    """
                }
            }
        }

        stage("approve deployment") {
            steps {
                script {
                    def userInput = input(
                        id: 'userInput', 
                        message: 'Approve Deployment?', 
                        parameters: [
                            choice(choices: ['Prod', 'Staging', 'Dev', 'Cancel'], 
                            description: 'Select environment:', 
                            name: 'DEPLOY_ENV')
                        ]
                    )
                    env.DEPLOY_ENV = userInput
                    echo "Deployment environment selected: ${env.DEPLOY_ENV}"
                }
            }
        }

        stage('Production Deployment') {
            when {
                expression { env.DEPLOY_ENV == 'Prod' }
            }
            steps {
                echo "Deploying application to Production namespace..."
                // Add your production kubectl logic here
            }
        }

        stage('Development Deployment') {
            when {
                expression { env.DEPLOY_ENV == 'Dev'}
            }
            steps {
                echo "Deploying application to Development..."
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

        stage('Staging Deployment') {
            when {
                expression { env.DEPLOY_ENV == 'Staging'}
            }
            steps {
                echo "Deploying application to Staging..."
            }
        }

        stage('Cancel Deployment') {
            when {
                expression { env.DEPLOY_ENV == 'Cancel' }
            }
            steps {
                echo 'Deployment has been cancelled.'
            }
        }
    }

    post {
        success {
            echo "Build completed successfully!"
        }
        failure {
            echo 'Build failed!'
        }
        always {
            echo 'Pipeline execution completed.'
        }
    }
}