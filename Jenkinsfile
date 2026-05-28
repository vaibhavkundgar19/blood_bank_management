pipeline {

    // MASTER-SLAVE Worker Selection
    agent {
        label "${env.BRANCH_NAME == 'main' ? 'worker-prod' : 'worker-staging'}"
    }

    tools {
        nodejs 'node21'
    }

    environment {
        DOCKERHUB_USERNAME = "biswajit7815"
        BACKEND_IMAGE      = "blood-backend"
        FRONTEND_IMAGE     = "blood-frontend"
        IMAGE_TAG          = "${BUILD_NUMBER}"

        SCANNER_HOME = tool 'sonar-scanner'

        // Branch ke hisaab se alag container names
        // main    --> blood-backend-main,    blood-frontend-main
        // staging --> blood-backend-staging, blood-frontend-staging
        BACKEND_CONTAINER  = "blood-backend-${env.BRANCH_NAME}"
        FRONTEND_CONTAINER = "blood-frontend-${env.BRANCH_NAME}"

        BACKEND_PORT = '5000'

        // EC2 Public IP
        EC2_PUBLIC_IP = "${env.BRANCH_NAME == 'main' ? '3.110.220.24' : '13.233.90.203'}"

        // SonarQube Project
        SONAR_PROJECT_KEY = "${env.BRANCH_NAME == 'main' ? 'blood-bank-prod' : 'blood-bank-staging'}"

        // Deploy Environment
        DEPLOY_ENV = "${env.BRANCH_NAME == 'main' ? 'PRODUCTION' : 'STAGING'}"
    }

    stages {

        // CLEANUP & CHECKOUT
        stage('cleanup & checkout') {
            steps {
                cleanWs()
                checkout scm
                echo "Code checkout complete - Build #${BUILD_NUMBER}"
                echo "Branch   : ${env.BRANCH_NAME}"
                echo "Worker   : ${env.NODE_NAME}"
                echo "Environ  : ${DEPLOY_ENV}"
            }
        }

        // INSTALL DEPENDENCIES
        stage('install dependencies') {
            parallel {

                stage('Backend install') {
                    steps {
                        dir('backend') {
                            sh 'npm install'
                        }
                    }
                }

                stage('Frontend install') {
                    steps {
                        dir('frontend') {
                            sh 'npm install'
                        }
                    }
                }
            }
        }

        // SECURITY SCAN
        stage('security scan') {
            parallel {

                // OWASP Dependency Check
                stage('OWASP Dependency Check') {
                    steps {

                        sh 'mkdir -p reports/owasp'

                        dependencyCheck(
                            additionalArguments: '''
                                --scan backend/
                                --scan frontend/
                                --format HTML
                                --format XML
                                --out reports/owasp/
                                --disableAssembly
                                --disableYarnAudit
                                --disableNodeAudit
                                --prettyPrint
                            ''',
                            odcInstallation: 'DP-Check'
                        )

                        dependencyCheckPublisher(
                            pattern: 'reports/owasp/dependency-check-report.xml',
                            failedTotalCritical: 20,
                            unstableTotalCritical: 20
                        )
                    }
                }

                // Trivy FS Scan
                stage('Trivy FS Scan') {
                    steps {

                        sh '''
                            mkdir -p reports/trivy

                            trivy fs . \
                                --exit-code 0 \
                                --severity HIGH,CRITICAL \
                                --format table \
                                -o reports/trivy/fs-scan.txt

                            cat reports/trivy/fs-scan.txt
                        '''
                    }
                }
            }
        }

        // SONARQUBE
        stage('SonarQube Analysis') {
            steps {

                withSonarQubeEnv('sonar-server') {

                    // SonarQube Scan
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                        -Dsonar.projectName=${SONAR_PROJECT_KEY} \
                        -Dsonar.sources=backend,frontend
                    """
                }
            }
        }

        // BUILD DOCKER IMAGES
        stage('Build Docker Images') {
            steps {

                echo "Building backend image..."

                // Docker Backend Build
                sh """
                    docker build \
                        -t ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:${IMAGE_TAG} \
                        -t ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:${env.BRANCH_NAME}-latest \
                        -f backend/Dockerfile \
                        ./backend
                """

                echo "Building frontend image..."

                // Docker Frontend Build
                sh """
                    docker build \
                        --build-arg VITE_API_PATH=http://${EC2_PUBLIC_IP}:${BACKEND_PORT} \
                        -t ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:${IMAGE_TAG} \
                        -t ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:${env.BRANCH_NAME}-latest \
                        -f frontend/Dockerfile \
                        ./frontend
                """
            }
        }

        // TRIVY IMAGE SCAN
        stage('Trivy Image Scan') {
            steps {

                sh """
                    trivy image \
                        --exit-code 0 \
                        --severity HIGH,CRITICAL \
                        --format table \
                        -o reports/trivy/backend-image-scan.txt \
                        ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:${IMAGE_TAG}

                    trivy image \
                        --exit-code 0 \
                        --severity HIGH,CRITICAL \
                        --format table \
                        -o reports/trivy/frontend-image-scan.txt \
                        ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:${IMAGE_TAG}
                """
            }
        }

        // PUSH TO DOCKER HUB
        stage('Push to Docker Hub') {
            steps {

                script {

                    withCredentials([
                        usernamePassword(
                            credentialsId: 'docker-hub-creds',
                            passwordVariable: 'DOCKER_PASS',
                            usernameVariable: 'DOCKER_USER'
                        )
                    ]) {

                        sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"

                        // Push Docker Images
                        sh "docker push ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:${IMAGE_TAG}"
                        sh "docker push ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:${IMAGE_TAG}"

                        sh "docker push ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:${env.BRANCH_NAME}-latest"
                        sh "docker push ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:${env.BRANCH_NAME}-latest"

                        sh "docker logout"
                    }
                }
            }
        }

        // DEPLOY
        stage('Deploy') {
            steps {

                script {

                    // MongoDB Credential Selection
                    def mongoCredId = (env.BRANCH_NAME == 'main')
                        ? 'MONGO_URI_PROD'
                        : 'MONGO_URI_STAGING'

                    withCredentials([
                        string(credentialsId: mongoCredId,  variable: 'MONGO_URI'),
                        string(credentialsId: 'JWT_SECRET', variable: 'JWT_SECRET')
                    ]) {

                        echo "Deploying to ${DEPLOY_ENV}..."

                        echo "Stopping old containers..."

                        sh "docker stop ${BACKEND_CONTAINER}  || true"
                        sh "docker stop ${FRONTEND_CONTAINER} || true"

                        sh "docker rm ${BACKEND_CONTAINER}  || true"
                        sh "docker rm ${FRONTEND_CONTAINER} || true"

                        echo "Creating Docker network..."

                        // Docker Network
                        sh "docker network create blood-network-${env.BRANCH_NAME} || true"

                        echo "Starting backend container..."

                        sh """
                            docker run -d \
                                --name ${BACKEND_CONTAINER} \
                                --network blood-network-${env.BRANCH_NAME} \
                                --restart unless-stopped \
                                -p ${BACKEND_PORT}:${BACKEND_PORT} \
                                -e MONGO_URI="${MONGO_URI}" \
                                -e JWT_SECRET="${JWT_SECRET}" \
                                -e NODE_ENV="${env.BRANCH_NAME == 'main' ? 'production' : 'staging'}" \
                                ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:${IMAGE_TAG}
                        """

                        echo "Starting frontend container..."

                        sh """
                            docker run -d \
                                --name ${FRONTEND_CONTAINER} \
                                --network blood-network-${env.BRANCH_NAME} \
                                --restart unless-stopped \
                                -p 80:80 \
                                ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:${IMAGE_TAG}
                        """

                        echo "Waiting for containers..."

                        sh "sleep 20"

                        echo "Container status..."

                        sh "docker ps --filter 'name=${BACKEND_CONTAINER}'"
                        sh "docker ps --filter 'name=${FRONTEND_CONTAINER}'"

                        echo "Backend health check..."

                        sh "curl -sf http://localhost:${BACKEND_PORT}/health && echo 'Backend Healthy'"

                        echo "Frontend health check..."

                        sh "curl -sf http://localhost && echo 'Frontend Healthy'"

                        echo "Application Live: http://${EC2_PUBLIC_IP}"
                    }
                }
            }
        }

        // CLEANUP OLD IMAGES
        stage('Cleanup Old Images') {
            steps {

                echo "Cleaning dangling images..."

                sh "docker image prune -f"

                echo "Removing old backend images..."

                // Cleanup Backend Images
                sh """
                    docker images ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE} --format "{{.Tag}}" \
                        | grep -v "${env.BRANCH_NAME}-latest" \
                        | grep -v "${IMAGE_TAG}" \
                        | xargs -r -I {} docker rmi ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:{} || true
                """

                echo "Removing old frontend images..."

                // Cleanup Frontend Images
                sh """
                    docker images ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE} --format "{{.Tag}}" \
                        | grep -v "${env.BRANCH_NAME}-latest" \
                        | grep -v "${IMAGE_TAG}" \
                        | xargs -r -I {} docker rmi ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:{} || true
                """
            }
        }
    }

    post {

        // ALWAYS RUN
        always {
            archiveArtifacts artifacts: 'reports/**/', allowEmptyArchive: true
            sh "docker logout || true"
        }

        // SUCCESS NOTIFICATION
        success {

            emailext(
                to:                 'biswajitbehera1868@gmail.com',
                subject:            "[${DEPLOY_ENV}] Build #${BUILD_NUMBER} - ${JOB_NAME} - SUCCESS",
                mimeType:           'text/html',
                attachmentsPattern: 'reports/trivy/*.txt',
                body:               """
                    <html>
                    <body style="font-family: Arial; padding: 20px; background-color: #f4f4f4;">

                        <div style="background-color: white; padding: 30px; border-radius: 8px; border-left: 5px solid green;">

                            <h2 style="color: green;">Build Successfully Deploy Ho Gaya!</h2>

                            <table border="1" cellpadding="10" cellspacing="0" width="100%">
                                <tr style="background-color: #f9f9f9;">
                                    <td><b>Environment</b></td>
                                    <td>${DEPLOY_ENV}</td>
                                </tr>
                                <tr>
                                    <td><b>Branch</b></td>
                                    <td>${env.BRANCH_NAME}</td>
                                </tr>
                                <tr style="background-color: #f9f9f9;">
                                    <td><b>Worker Node</b></td>
                                    <td>${env.NODE_NAME}</td>
                                </tr>
                                <tr>
                                    <td><b>Project</b></td>
                                    <td>${JOB_NAME}</td>
                                </tr>
                                <tr style="background-color: #f9f9f9;">
                                    <td><b>Build Number</b></td>
                                    <td>#${BUILD_NUMBER}</td>
                                </tr>
                                <tr>
                                    <td><b>Status</b></td>
                                    <td style="color: green;"><b>SUCCESS</b></td>
                                </tr>
                            </table>

                        </div>
                    </body>
                    </html>
                """
            )

//             slackSend(
//                 channel: '#jenkins-notifications',
//                 color:   'good',
//                 message: """
// [${DEPLOY_ENV}] BUILD SUCCESS

// Branch      : ${env.BRANCH_NAME}
// Worker Node : ${env.NODE_NAME}
// Project     : ${JOB_NAME}
// Build       : #${BUILD_NUMBER}
// Duration    : ${currentBuild.durationString}
// App URL     : http://${EC2_PUBLIC_IP}
// Build URL   : ${BUILD_URL}
//                 """
//             )
        }

        // FAILURE NOTIFICATION
        failure {

            emailext(
                to:                 'biswajitbehera1868@gmail.com',
                subject:            "[${DEPLOY_ENV}] Build #${BUILD_NUMBER} - ${JOB_NAME} - FAILED",
                mimeType:           'text/html',
                attachmentsPattern: 'reports/trivy/*.txt',
                body:               """
                    <html>
                    <body style="font-family: Arial; padding: 20px; background-color: #f4f4f4;">

                        <div style="background-color: white; padding: 30px; border-radius: 8px; border-left: 5px solid red;">

                            <h2 style="color: red;">Build Fail Ho Gaya!</h2>

                        </div>
                    </body>
                    </html>
                """
            )

//             slackSend(
//                 channel: '#jenkins-notifications',
//                 color:   'danger',
//                 message: """
// [${DEPLOY_ENV}] BUILD FAILED

// Branch      : ${env.BRANCH_NAME}
// Worker Node : ${env.NODE_NAME}
// Project     : ${JOB_NAME}
// Build       : #${BUILD_NUMBER}
// Duration    : ${currentBuild.durationString}
// Console     : ${BUILD_URL}console
//                 """
//             )
        }

        // cleanup
        cleanup {
            cleanWs(
                cleanWhenSuccess:  true,
                cleanWhenFailure:  false,
                cleanWhenAborted:  true
            )
        }
    }
}