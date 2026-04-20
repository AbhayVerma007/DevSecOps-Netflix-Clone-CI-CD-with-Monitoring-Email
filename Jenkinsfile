// ============================================================
// Netflix Clone - DevSecOps CI/CD Pipeline
// Version: 2.0.0
// Last Updated: 2026-04-20
// Fixes:
//   - TMDB API key moved to Jenkins credentials (no hardcoding)
//   - Trivy image scan now targets the correct built image
//   - Docker image tag uses BUILD_NUMBER for version control
//   - Added BUILD_VERSION env var for traceability
//   - Deployment stage uses versioned image tag
//   - Upgraded to JDK 21 for stability
// ============================================================

pipeline {
    agent any

    tools {
        jdk 'jdk21'       // Upgraded from jdk17 for stability
        nodejs 'node18'   // Upgraded from node16 (EOL) to node18 LTS
    }

    environment {
        SCANNER_HOME      = tool 'sonar-scanner'
        DOCKER_IMAGE      = "abhay/netflix"
        BUILD_VERSION     = "2.0.${env.BUILD_NUMBER}"   // e.g. 2.0.42
        IMAGE_TAG         = "${DOCKER_IMAGE}:${BUILD_VERSION}"
        IMAGE_TAG_LATEST  = "${DOCKER_IMAGE}:latest"
    }

    stages {

        // ----------------------------------------------------------
        // Stage 1 – Clean workspace
        // ----------------------------------------------------------
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        // ----------------------------------------------------------
        // Stage 2 – Checkout source code
        // ----------------------------------------------------------
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/AbhayVerma007/DevSecOps-Netflix-Clone-CI-CD-with-Monitoring-Email.git'
            }
        }

        // ----------------------------------------------------------
        // Stage 3 – Static code analysis (SonarQube)
        // ----------------------------------------------------------
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                            -Dsonar.projectName=Netflix \
                            -Dsonar.projectKey=Netflix \
                            -Dsonar.projectVersion=${BUILD_VERSION}
                    '''
                }
            }
        }

        // ----------------------------------------------------------
        // Stage 4 – SonarQube Quality Gate
        // ----------------------------------------------------------
        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }

        // ----------------------------------------------------------
        // Stage 5 – Install Node dependencies
        // ----------------------------------------------------------
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }

        // ----------------------------------------------------------
        // Stage 6 – OWASP Dependency Check (filesystem scan)
        // ----------------------------------------------------------
        stage('OWASP FS Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit',
                                odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        // ----------------------------------------------------------
        // Stage 7 – Trivy filesystem scan
        // ----------------------------------------------------------
        stage('Trivy FS Scan') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }

        // ----------------------------------------------------------
        // Stage 8 – Docker build & push
        //   FIX: TMDB key now pulled from Jenkins credentials, NOT hardcoded
        //   FIX: Tags image with BUILD_VERSION for version control
        // ----------------------------------------------------------
        stage('Docker Build & Push') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'tmdb-api-key', variable: 'TMDB_KEY')]) {
                        withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                            sh "docker build --build-arg TMDB_V3_API_KEY=${TMDB_KEY} -t ${IMAGE_TAG} ."
                            sh "docker tag ${IMAGE_TAG} ${IMAGE_TAG_LATEST}"
                            sh "docker push ${IMAGE_TAG}"
                            sh "docker push ${IMAGE_TAG_LATEST}"
                        }
                    }
                }
            }
        }

        // ----------------------------------------------------------
        // Stage 9 – Trivy image scan
        //   FIX: Was scanning 'sevenajay/netflix:latest' (wrong image).
        //        Now scans the image that was actually built above.
        // ----------------------------------------------------------
        stage('Trivy Image Scan') {
            steps {
                sh "trivy image ${IMAGE_TAG} > trivyimage.txt"
            }
        }

        // ----------------------------------------------------------
        // Stage 10 – Deploy to Docker container (local/staging)
        // ----------------------------------------------------------
        stage('Deploy to Container') {
            steps {
                // Remove old container if running, then start fresh with versioned image
                sh '''
                    docker rm -f netflix 2>/dev/null || true
                    docker run -d --name netflix -p 8081:80 ${IMAGE_TAG}
                '''
            }
        }

        // ----------------------------------------------------------
        // Stage 11 – Deploy to Kubernetes (TEMPORARILY DISABLED)
        // ----------------------------------------------------------
        /*
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    dir('Kubernetes') {
                        withKubeConfig(
                            caCertificate: '',
                            clusterName: '',
                            contextName: '',
                            credentialsId: 'k8s',
                            namespace: '',
                            restrictKubeConfigAccess: false,
                            serverUrl: ''
                        ) {
                            // Inject versioned image tag into the deployment manifest on-the-fly
                            sh "sed -i 's|IMAGE_PLACEHOLDER|${IMAGE_TAG}|g' deployment.yml"
                            sh 'kubectl apply -f deployment.yml'
                            sh 'kubectl apply -f service.yml'
                        }
                    }
                }
            }
        }
        */
    }

        // ----------------------------------------------------------
        // Post – always send email with scan reports attached
        // ----------------------------------------------------------
        post {
            always {
                emailext(
                    attachLog: true,
                    subject: "[${currentBuild.result}] Build #${BUILD_VERSION} - ${env.JOB_NAME}",
                    body: """
                        <b>Project:</b> ${env.JOB_NAME}<br/>
                        <b>Build Version:</b> ${BUILD_VERSION}<br/>
                        <b>Build Number:</b> ${env.BUILD_NUMBER}<br/>
                        <b>Status:</b> ${currentBuild.result}<br/>
                        <b>URL:</b> ${env.BUILD_URL}<br/>
                    """,
                    to: 'vermaabhay085@gmail.com',
                    attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
                )
            }
        }
    }        
