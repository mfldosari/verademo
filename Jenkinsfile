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
        // githubCred = credentials('github-credentials')
        // githubUsername = "${env.githubCred_USR}"
        // githubToken = "${env.githubCred_PSW}"

        IMAGE = "${HOSTNAME}/${APPLICATION_NAME}:v${BUILD_NUMBER}"
        _LATEST = "${HOSTNAME}/${APPLICATION_NAME}:latest"
        
        GIT_REPO = "${env.GIT_URL.replaceAll('https://', 'git://').replaceAll('http://', 'git://')}"
        GIT_BRANCH = "${params.GIT_BRANCH_NAME ?: (env.GIT_BRANCH ? env.GIT_BRANCH.replaceAll('origin/', '') : 'main')}"
    }

    stages {
        stage('Scan Codebase') {
            steps {
                // Binding 'string' for Secret Text and 'file' for Secret File
                withCredentials([
                    string(credentialsId: "${env.GH_TOKEN_ID}", variable: 'GH_TOKEN'),
                    file(credentialsId: "${env.SCAN_CONFIG_ID}", variable: 'SCAN_CONFIG_FILE')
                ]) {
                    script {
                        // 1. Load config from the path
                        def scanConfig = readJSON file: env.SCAN_CONFIG_FILE
                        def scanApiUrl = scanConfig.SCAN_API_URL
                        def accessToken = scanConfig.ACCESS_TOKEN

                        // 2. Build Payload
                        def payloadMap = [
                            action_type: "scan_only",
                            mock_mode: false,
                            issue_tracker: [
                                tracker_type: "github",
                                server_url: "https://github.com",
                                username: "mfldosari",
                                password: GH_TOKEN,
                                project_key: "mfldosari/${env.APPLICATION_NAME}"
                            ],
                            scanner: [
                                name: "Security Scan - veracode",
                                git_url: "https://github.com/mfldosari/verademo",
                                git_branch: env.GIT_BRANCH,
                                git_credentials: GH_TOKEN,
                                git_username: "mfldosari",
                                git_password: GH_TOKEN,
                                llm_endpoint_url: "http://172.20.30.92:11434/v1",
                                llm_model_name: "SimonPu/qwen3-coder:30B-Instruct_Q4_K_XL",
                                llm_api_key: "ollama",
                                llm_max_context_tokens: 262144,
                                include_file_extensions: [".java",".jsp",".js",".py"],
                                exclude_file_dirs: [".git", "node_modules", "venv", "__pycache__", "docs", "target"],
                                include_vulnerability_types: ["SQL Injection", "Cross-Site Scripting (XSS)", "Insecure Authentication"],
                                max_vulnerabilities: 10
                            ]
                        ]

                        def jsonPayload = groovy.json.JsonOutput.toJson(payloadMap)

                        // 3. Trigger Scan
                        // Note: Using single quotes for the -d payload to avoid shell interference
                        def response = sh(script: """
                            curl -s -k -X POST \
                            -H "Content-Type: application/json" \
                            -H "accept: application/json" \
                            -b "access_token=${accessToken}" \
                            "${scanApiUrl}/api/v1/orchestrator/execute" \
                            -d '${jsonPayload}'
                        """, returnStdout: true).trim()

                        echo "API RESPONSE: ${response}"
                        def props = readJSON text: response
                        
                        if (props.scan_id) {
                            env.SCAN_ID = props.scan_id
                        } else {
                            error "Scan failed to start: ${response}"
                        }

                        // 4. Polling Loop
                        def status = "running"
                        def statusUrl = "${scanApiUrl}/api/v1/scans/${env.SCAN_ID}"
                        
                        while (status == "running" || status == "queued" || status == "in_progress") {
                            echo "Current Status: ${status}. Waiting 2 minutes..."
                            sleep(120) 
                            def pollResponse = sh(script: "curl -s -X GET '${statusUrl}' -H 'accept: application/json'", returnStdout: true).trim()
                            def pollJson = readJSON text: pollResponse
                            status = pollJson.status
                        }

                        // 5. Security Gate
                        if (status == "completed") {
                            def finalResult = sh(script: "curl -s -X GET '${statusUrl}' -H 'accept: application/json'", returnStdout: true).trim()
                            def data = readJSON text: finalResult
                            
                            int critical = data.critical_count ?: 0
                            int high = data.high_count ?: 0

                            echo "Final Counts - Critical: ${critical}, High: ${high}"

                            if (critical > 0 || high > 0) {
                                error "GATE FAILED: High-risk vulnerabilities found."
                            } else {
                                echo "Gate Passed."
                            }
                        }
                    }
                }
            }
        }
    }
}
        
      
            
        
        

        // stage('Build & Push') {
        //     steps {
        //         echo "Security gate passed. Building Docker image..."
        //         sh """
        //             docker build -t ${IMAGE} -f ${DOCKERFILE} .
        //             docker tag ${IMAGE} ${_LATEST}
        //             echo ${PASSWORD} | docker login ${HOSTNAME} -u ${USERNAME} --password-stdin
        //             docker push ${IMAGE}
        //             docker push ${_LATEST}
        //         """
        //     }
        // }
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
//     }
// }