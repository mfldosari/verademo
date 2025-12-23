pipeline {
    agent any

    parameters {
        string(name: 'GIT_BRANCH_NAME', defaultValue: 'main', description: 'Git branch name to build from')
        string(name: 'GIT_TARGET_SCANNER_BRANCH_NAME', defaultValue: 'test', description: 'Git branch name to scan from')
    }

    environment {
        APPLICATION_NAME = 'verademo'
        DEV_NODE_PORT = credentials('DEV_NODE_PORT') 
        PROD_NODE_PORT = credentials('PROD_NODE_PORT')
        REGISTRY_CREDS = credentials('registry-creds')
        USERNAME = "${REGISTRY_CREDS_USR}"
        PASSWORD = "${REGISTRY_CREDS_PSW}"
        HOSTNAME = credentials('registry-hostname')
        DOCKERFILE = 'Dockerfile.dev'
        scan_config_json = credentials('scan-config-json')
        IMAGE = "${HOSTNAME}/${APPLICATION_NAME}:v${BUILD_NUMBER}"
        _LATEST = "${HOSTNAME}/${APPLICATION_NAME}:latest"
        GIT_REPO = "${env.GIT_URL.replaceAll('https://', 'git://').replaceAll('http://', 'git://')}"
        GIT_BRANCH = "${params.GIT_BRANCH_NAME ?: (env.GIT_BRANCH ? env.GIT_BRANCH.replaceAll('origin/', '') : 'main')}"
        GIT_TARGET_SCANNER_BRANCH_NAME = "${params.GIT_TARGET_SCANNER_BRANCH_NAME ?: (env.GIT_BRANCH ? env.GIT_BRANCH.replaceAll('origin/', '') : 'test')}"
    }

    stages {
      stage('Scan Codebase & Security Gate') {
        steps {
            echo "Scanning codebase for vulnerabilities..."
            script {
                def scanConfig = readJSON file: env.scan_config_json
                env.SCAN_API_URL = scanConfig.SCAN_API_URL 
                env.ACCESS_TOKEN = scanConfig.ACCESS_TOKEN
                env.git_token = scanConfig.git_token  

                // 2. Trigger the Scan (Fixed formatting for the JSON payload)
                def response = sh(script: """
                    curl -s -k -X POST \
                    -H "Content-Type: application/json" \
                    -H "accept: application/json" \
                    -b "access_token=${env.ACCESS_TOKEN}" \
                    "${env.SCAN_API_URL}/api/v1/orchestrator/execute" \
                    --data-binary @- <<EOF
{
    "action_type": "scan_only",
    "mock_mode": false,
    "issue_tracker": {
        "tracker_type": "github",
        "server_url": "https://github.com",
        "username": "mfldosari",
        "password": "${env.git_token}",
        "project_key": "mfldosari/${env.APPLICATION_NAME}"
    },
    "scanner": {
        "name": "Security Scan - veracode",
        "git_url": "https://github.com/mfldosari/verademo",
        "git_branch": "${env.GIT_TARGET_SCANNER_BRANCH_NAME}",
        "git_credentials": "${env.git_token}",
        "git_username": "mfldosari",
        "git_password": "${env.git_token}",
        "llm_endpoint_url": "http://172.20.30.92:11434/v1",
        "llm_model_name": "SimonPu/qwen3-coder:30B-Instruct_Q4_K_XL",
        "llm_api_key": "ollama",
        "llm_max_context_tokens": 262144,
        "include_file_extensions": [".java",".jsp",".js",".py"],
        "exclude_file_dirs": [".git", "node_modules", "venv", "__pycache__", "docs", "target"],
        "include_vulnerability_types": ["SQL Injection", "Cross-Site Scripting (XSS)", "Insecure Authentication"],
        "max_vulnerabilities": 10
    }
}
EOF
                """, returnStdout: true).trim()

                def props = readJSON text: response
                env.SCAN_ID = props.scan_id
                echo "Scan initiated. ID: ${env.SCAN_ID}"

                // 3. Polling Loop
                def status = "running"
                def statusUrl = "${env.SCAN_API_URL}/api/v1/scans/${env.SCAN_ID}"
                
                while (status == "running") {
                    echo "Scan still in progress... waiting 2 minutes."
                    sleep(120)
                    
                    def pollResponse = sh(script: "curl -s -X GET '${statusUrl}' -H 'accept: application/json'", returnStdout: true).trim()
                    def pollJson = readJSON text: pollResponse
                    status = pollJson.status
                    echo "Current Status: ${status}"
                    
                    if (status == "failed") {
                        error "Scanner system failure. Aborting build."
                    }
                }

                // 4. Final Security Gate
                if (status == "completed") {
                    def finalResult = sh(script: "curl -s -X GET '${statusUrl}' -H 'accept: application/json'", returnStdout: true).trim()
                    def data = readJSON text: finalResult
                    
                    int total = data.total_vulnerabilities_found ?: 0
                    int critical = data.critical_count ?: 0
                    int high = data.high_count ?: 0
                    int medium = data.medium_count ?: 0

                    echo "Report: Total: ${total}, Critical: ${critical}, High: ${high}, Medium: ${medium}"

                    if (total > 0 && (critical > 0 || high > 0 || medium > 0)) {
                        error "SECURITY GATE FAILED: High-risk vulnerabilities found. Blocking deployment."
                    } else {
                        echo "Security Gate Passed. Moving to Build stage."
                    }
                }
            }
        }
      }
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
    stage('approve deployment') {
        steps {
            script {
                def userInput = input(
                    id: 'userInput', 
                    message: 'Approve Deployment?', 
                    parameters: [
                        choice(choices: ['Prod (DISABLED)', 'Staging (DISABLED)', 'Dev', 'Cancel'], 
                        description: 'Select environment:', 
                        name: 'DEPLOY_ENV')
                    ]
                )
                env.DEPLOY_ENV = userInput
                echo "Deployment environment selected: ${env.DEPLOY_ENV}"
            }
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
        image: ${IMAGE}
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

}

}
  

//         stage('registry credentials setup') {
//             steps {
//                 sh """
//                 kubectl create secret docker-registry registry-config \
//                   --docker-server="${HOSTNAME}" \
//                   --docker-username="${USERNAME}" \
//                   --docker-password="${PASSWORD}" \
//                   --namespace=jenkins \
//                   --dry-run=client -o yaml | kubectl apply -f -
//                 """
//             }
//         }

//         stage('Build Docker Image with Kaniko') {
//             when {
//                 expression { USE_PREBUILT_IMAGE != 'true' }
//             }
//             steps {
//                 echo "Building ${IMAGE} with Kaniko..."
//                 script {
//                     sh """
//                     kubectl run kaniko-build-${BUILD_NUMBER} \\
//                       --restart=Never \\
//                       --image=gcr.io/kaniko-project/executor:latest \\
//                       --namespace=jenkins \\
//                       --overrides='{
//                         "spec": {
//                           "containers": [{
//                             "name": "kaniko",
//                             "image": "gcr.io/kaniko-project/executor:latest",
//                             "args": [
//                               "--context=${GIT_REPO}#refs/heads/${GIT_BRANCH}",
//                               "--dockerfile=${DOCKERFILE}",
//                               "--destination=${IMAGE}",
//                               "--destination=${_LATEST}",
//                               "--skip-tls-verify"
//                             ],
//                             "volumeMounts": [{
//                               "name": "docker-config",
//                               "mountPath": "/kaniko/.docker"
//                             }]
//                           }],
//                           "volumes": [{
//                             "name": "docker-config",
//                             "secret": {
//                               "secretName": "registry-config",
//                               "items": [{
//                                 "key": ".dockerconfigjson",
//                                 "path": "config.json"
//                               }]
//                             }
//                           }],
//                           "restartPolicy": "Never"
//                         }
//                       }' \\
//                       --attach=true \\
//                       --rm
//                     """
//                 }
//             }
//         }

//         stage("approve deployment") {
//             steps {
//                 script {
//                     def userInput = input(
//                         id: 'userInput', 
//                         message: 'Approve Deployment?', 
//                         parameters: [
//                             choice(choices: ['Prod (DISABLED)', 'Staging (DISABLED)', 'Dev', 'Cancel'], 
//                             description: 'Select environment:', 
//                             name: 'DEPLOY_ENV')
//                         ]
//                     )
//                     env.DEPLOY_ENV = userInput
//                     echo "Deployment environment selected: ${env.DEPLOY_ENV}"
//                 }
//             }
//         }

//         stage('Production Deployment') {
//             when {
//                 expression { env.DEPLOY_ENV == 'Prod' }
//             }
//             steps {
//                 echo "Deploying application to Production namespace..."
                
//             }
//         }

//         stage('Development Deployment') {
//             when {
//                 expression { env.DEPLOY_ENV == 'Dev'}
//             }
//             steps {
//                 echo "Deploying application to Development..."
//                 script {
//                     sh """
//                         cat <<EOF | kubectl apply -f -
// apiVersion: v1
// kind: Namespace
// metadata:
//   name: ${env.DEPLOY_ENV.toLowerCase()}-${APPLICATION_NAME.toLowerCase()}
// ---
// apiVersion: apps/v1
// kind: Deployment
// metadata:
//   name: ${APPLICATION_NAME.toLowerCase()}
//   namespace: ${env.DEPLOY_ENV.toLowerCase()}-${APPLICATION_NAME.toLowerCase()}
//   labels:
//     app: ${APPLICATION_NAME.toLowerCase()}
// spec:
//   replicas: 1
//   selector:
//     matchLabels:
//       app: ${APPLICATION_NAME.toLowerCase()}
//   template:
//     metadata:
//       labels:
//         app: ${APPLICATION_NAME.toLowerCase()}
//     spec:
//       containers:
//       - name: ${APPLICATION_NAME.toLowerCase()}
//         image: ${USE_PREBUILT_IMAGE == 'true' ? PREBUILT_IMAGE : IMAGE}
//         imagePullPolicy: Always
//         ports:
//         - containerPort: 8080
//         env:
//         - name: MYSQL_DATABASE
//           value: "blab"
// ---
// apiVersion: v1
// kind: Service
// metadata:
//   name: ${APPLICATION_NAME.toLowerCase()}-service
//   namespace: ${env.DEPLOY_ENV.toLowerCase()}-${APPLICATION_NAME.toLowerCase()}
// spec:
//   type: NodePort
//   selector:
//     app: ${APPLICATION_NAME.toLowerCase()}
//   ports:
//   - name: http
//     port: 8080
//     targetPort: 8080
//     nodePort: ${DEV_NODE_PORT}
// EOF
//                     """
//                 }
//                 echo "Application ${APPLICATION_NAME} deployed successfully to ${env.DEPLOY_ENV}"
//             }
//         }

//         stage('Staging Deployment') {
//             when {
//                 expression { env.DEPLOY_ENV == 'Staging'}
//             }
//             steps {
//                 echo "Deploying application to Staging..."
//             }
//         }

//         stage('Cancel Deployment') {
//             when {
//                 expression { env.DEPLOY_ENV == 'Cancel' }
//             }
//             steps {
//                 echo 'Deployment has been cancelled.'
//             }
//         }
//     }

//     post {
//         success {
//             echo "Build completed successfully!"
//         }
//         failure {
//             echo 'Build failed!'
//         }
//         always {
//             echo 'Pipeline execution completed.'
//         }
