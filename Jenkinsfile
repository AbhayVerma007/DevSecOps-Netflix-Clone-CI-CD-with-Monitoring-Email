// ============================================================
// Netflix Clone - DevSecOps CI/CD Pipeline
// Target: Docker Deployment (Pre-Kubernetes)
// Version: 2.1.0
// Fixes applied:
//   1. env.BUILD_NUMBER → BUILD_NUMBER (reliable across all Jenkins versions)
//   2. waitForQualityGate wrapped in timeout(2 min) to prevent hanging
//   3. NVD key passed via temp properties file (no inline secret in args)
//   4. jdk21 tool name — must match Jenkins Global Tool Configuration exactly
// ============================================================

pipeline {
    agent any

    tools {
        jdk 'jdk21'       // Must match name in: Manage Jenkins → Global Tool Configuration → JDK
        nodejs 'node18'   // Must match name in: Manage Jenkins → Global Tool Configuration → NodeJS
    }

    environment {
        SCANNER_HOME      = tool 'sonar-scanner'
        DOCKER_IMAGE      = "abhayverma007/netflix"
        BUILD_VERSION     = "2.0.${BUILD_NUMBER}"        // FIX 1: was env.BUILD_NUMBER
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
                git branch: 'main', url: 'https://github.com/Aj7Ay/Netflix-clone.git'
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
        // FIX 2: wrapped in timeout — prevents pipeline hanging
        //         indefinitely if SonarQube never responds
        // ----------------------------------------------------------
        stage('Quality Gate') {
            steps {
                script {
                    timeout(time: 2, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                    }
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
        // FIX 3: NVD key written to a temp properties file instead of
        //         passing inline — avoids secret leaking in debug logs
        // ----------------------------------------------------------
        stage('OWASP FS Scan') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_KEY')]) {
                        sh 'echo "nvd.api.key=${NVD_KEY}" > /tmp/nvd.properties'
                        dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit --propertyfile /tmp/nvd.properties',
                                        odcInstallation: 'DP-Check'
                        sh 'rm -f /tmp/nvd.properties'
                    }
                }
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
                to: 'dhaliwalakshit@gmail.com',
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
            )
        }
    }
}
