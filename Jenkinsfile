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
          script {
    def scanConfig = readJSON file: env.scan_config_json
    env.SCAN_API_URL = scanConfig.SCAN_API_URL
    env.ACCESS_TOKEN = scanConfig.ACCESS_TOKEN

    // 1. Add -v to curl for more info, and capture the output
    echo "Sending request to ${env.SCAN_API_URL}/api/v1/orchestrator/execute"
    
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
              "git_branch": "${env.GIT_BRANCH}",
              "git_username": "${env.githubUsername}",
              "git_password": "${env.githubToken}"
          }
        }
        EOF
    """, returnStdout: true).trim()

    // 2. DEBUG: Print the raw response to the Jenkins console
    echo "RAW API RESPONSE: ${response}"

    if (response && response.startsWith("{")) {
        def props = readJSON text: response
        
        // 3. Check if scan_id exists or if it's named something else
        if (props.scan_id) {
            env.SCAN_ID = props.scan_id
            echo "Scan initiated successfully. ID: ${env.SCAN_ID}"
        } else {
            error "API responded but no scan_id found. Response was: ${response}"
        }
    } else {
        error "Failed to get a valid JSON response from the API. Check connectivity or credentials."
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