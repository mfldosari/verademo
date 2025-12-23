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
        
        // Docker Registry credentials
        REGISTRY_CREDS = credentials('registry-creds')
        USERNAME = "${REGISTRY_CREDS_USR}"
        PASSWORD = "${REGISTRY_CREDS_PSW}"
        
        HOSTNAME = credentials('registry-hostname')
        DOCKERFILE = credentials('dockerfile-name')

        // Scan Codebase config file
        scan_config_json = credentials('scan-config-json')

        // GitHub credentials
        githubCred = credentials('github-credentials')
        githubUsername = "${githubCred_USR}"
        githubToken = "${githubCred_PSW}"

        IMAGE = "${HOSTNAME}/${APPLICATION_NAME}:v${BUILD_NUMBER}"
        _LATEST = "${HOSTNAME}/${APPLICATION_NAME}:latest"
        
        GIT_REPO = "${env.GIT_URL.replaceAll('https://', 'git://').replaceAll('http://', 'git://')}"
        GIT_BRANCH = "${params.GIT_BRANCH_NAME ?: (env.GIT_BRANCH ? env.GIT_BRANCH.replaceAll('origin/', '') : 'main')}"
    }

    stages {
        stage('Scan Codebase') {
            steps {
                echo "Scanning codebase for vulnerabilities..."
                script {
                    // 1. Load configuration from JSON file
                    def scanConfig = readJSON file: env.scan_config_json
                    env.SCAN_API_URL = scanConfig.SCAN_API_URL
                    env.ACCESS_TOKEN = scanConfig.ACCESS_TOKEN

                    echo "Sending request to ${env.SCAN_API_URL}/api/v1/orchestrator/execute"

                    // 2. TRIGGER SCAN (POST)
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
                              "username": "${env.githubUsername}",
                              "password": "${env.githubToken}",
                              "project_key": "mfldosari/${env.APPLICATION_NAME}"
                          },
                          "scanner": {
                              "name": "Security Scan - veracode",
                              "git_url": "https://github.com/mfldosari/verademo",
                              "git_branch": "test",
                              "git_credentials": "${env.githubToken}",
                              "git_username": "${env.githubUsername}",
                              "git_password": "${env.githubToken}",
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

                    echo "RAW API RESPONSE: ${response}"
                    def props = readJSON text: response
                    env.SCAN_ID = props.scan_id

                    if (!env.SCAN_ID || env.SCAN_ID == "null") {
                        error "Failed to initiate scan. Response was: ${response}"
                    }

                    // 3. POLLING LOOP (Wait for "completed" or "failed")
                    def status = "running"
                    def statusUrl = "${env.SCAN_API_URL}/api/v1/scans/${env.SCAN_ID}"
                    
                    while (status == "running" || status == "queued" || status == "in_progress") {
                        echo "Current Status: ${status}. Waiting 2 minutes before next check..."
                        sleep(120) 
                        
                        def pollResponse = sh(script: "curl -s -X GET '${statusUrl}' -H 'accept: application/json'", returnStdout: true).trim()
                        def pollJson = readJSON text: pollResponse
                        status = pollJson.status
                        
                        if (status == "failed") {
                            error "Scanner encountered a system failure. Aborting build."
                        }
                    }

                    // 4. QUALITY GATE (Check total and specific counts)
                    if (status == "completed") {
                        def finalResult = sh(script: "curl -s -X GET '${statusUrl}' -H 'accept: application/json'", returnStdout: true).trim()
                        def data = readJSON text: finalResult
                        
                        int total    = data.total_vulnerabilities_found ?: 0
                        int critical = data.critical_count ?: 0
                        int high     = data.high_count ?: 0
                        int medium   = data.medium_count ?: 0

                        echo "Vulnerability Summary -> Total: ${total}, Critical: ${critical}, High: ${high}, Medium: ${medium}"

                        if (total == 0) {
                            echo "Zero vulnerabilities found. Proceeding with deployment."
                        } else if (critical > 0 || high > 0 || medium > 0) {
                            // This stops the pipeline and marks the build as FAILED
                            error "SECURITY GATE FAILED: Found ${critical} Critical, ${high} High, and ${medium} Medium vulnerabilities."
                        } else {
                            echo "Only Low vulnerabilities found. Continuing build."
                        }
                    }
                }
            }
        }

        stage('Build & Push Image') {
            // This stage only runs if the 'Scan Codebase' stage finishes successfully
            steps {
                echo "Security check passed. Starting Build..."
                sh """
                    docker build -t ${IMAGE} -f ${DOCKERFILE} .
                    docker tag ${IMAGE} ${_LATEST}
                    echo ${PASSWORD} | docker login ${HOSTNAME} -u ${USERNAME} --password-stdin
                    docker push ${IMAGE}
                    docker push ${_LATEST}
                """
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
    }
}