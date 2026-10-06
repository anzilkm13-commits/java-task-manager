pipeline {

    agent any

    environment {
        APP_NAME = 'java-task-manager'
        APP_PORT = '8081'

        NEXUS_URL = 'http://172.31.13.19:8081'
        NEXUS_REPOSITORY = 'maven-snapshots'

        GROUP_ID = 'com.example'
        ARTIFACT_ID = 'spring-boot-todo-applicationx'
        APP_VERSION = '0.0.1-SNAPSHOT'
    }

    stages {

        // =========================================================
        // 1. CHECKOUT
        // =========================================================
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'

                checkout scm
            }
        }


        // =========================================================
        // 2. VERIFY TOOLS
        // =========================================================
        stage('Verify Tools') {
            steps {

                sh '''
                    echo "======================================"
                    echo "JAVA VERSION"
                    echo "======================================"
                    java --version

                    echo "======================================"
                    echo "MAVEN VERSION"
                    echo "======================================"
                    ./mvnw --version

                    echo "======================================"
                    echo "GIT VERSION"
                    echo "======================================"
                    git --version

                    echo "======================================"
                    echo "DOCKER VERSION"
                    echo "======================================"
                    docker --version

                    echo "======================================"
                    echo "TRIVY VERSION"
                    echo "======================================"
                    trivy --version
                '''
            }
        }


        // =========================================================
        // 3. CLEAN
        // =========================================================
        stage('Clean') {
            steps {

                echo 'Cleaning previous Maven build...'

                sh '''
                    ./mvnw clean
                '''
            }
        }


        // =========================================================
        // 4. COMPILE
        // =========================================================
        stage('Compile') {
            steps {

                echo 'Compiling Java application...'

                sh '''
                    ./mvnw compile
                '''
            }
        }


        // =========================================================
        // 5. TEST
        // =========================================================
        stage('Test') {
            steps {

                echo 'Running unit tests...'

                sh '''
                    ./mvnw test
                '''
            }
        }


        // =========================================================
        // 6. TRIVY FILESYSTEM SCAN
        // =========================================================
        stage('Trivy Filesystem Scan') {
            steps {

                echo 'Scanning project files with Trivy...'

                sh '''
                    trivy fs \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        .
                '''
            }
        }


        // =========================================================
        // 7. SONARQUBE ANALYSIS
        // =========================================================
        stage('SonarQube Analysis') {
            steps {

                echo 'Running SonarQube analysis...'

                withSonarQubeEnv('SonarQube') {

                    sh '''
                        ./mvnw \
                            org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                            -Dsonar.projectKey=java-task-manager \
                            -Dsonar.projectName=java-task-manager
                    '''
                }
            }
        }


        // =========================================================
        // 8. SONARQUBE QUALITY GATE
        // =========================================================
        stage('SonarQube Quality Gate') {
            steps {

                echo 'Waiting for SonarQube Quality Gate...'

                timeout(
                    time: 5,
                    unit: 'MINUTES'
                ) {

                    waitForQualityGate(
                        abortPipeline: true
                    )
                }
            }
        }


        // =========================================================
        // 9. PACKAGE
        // =========================================================
        stage('Package') {
            steps {

                echo 'Packaging Spring Boot application...'

                sh '''
                    ./mvnw package -DskipTests
                '''
            }
        }


        // =========================================================
        // 10. TEST NEXUS AUTHENTICATION
        // =========================================================
        stage('Test Nexus Authentication') {
            steps {

                echo 'Testing Jenkins -> Nexus authentication...'

                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexus-credentials',
                        usernameVariable: 'NEXUS_USERNAME',
                        passwordVariable: 'NEXUS_PASSWORD'
                    )
                ]) {

                    sh '''
                        HTTP_CODE=$(curl -s \
                            -o /dev/null \
                            -w "%{http_code}" \
                            -u "${NEXUS_USERNAME}:${NEXUS_PASSWORD}" \
                            "${NEXUS_URL}/service/rest/v1/status")

                        echo "Nexus authentication HTTP status: ${HTTP_CODE}"

                        if [ "$HTTP_CODE" != "200" ]; then
                            echo "Nexus authentication failed."
                            exit 1
                        fi

                        echo "Nexus authentication successful."
                    '''
                }
            }
        }


        // =========================================================
        // 11. PUBLISH JAR TO NEXUS
        // =========================================================
        stage('Publish to Nexus') {
            steps {

                echo 'Publishing Maven artifact to Nexus...'

                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexus-credentials',
                        usernameVariable: 'NEXUS_USERNAME',
                        passwordVariable: 'NEXUS_PASSWORD'
                    )
                ]) {

                    sh '''
                        set -e

                        echo "Preparing Maven repository path..."

                        GROUP_PATH=$(echo "${GROUP_ID}" | tr '.' '/')

                        ARTIFACT_FILE="target/${ARTIFACT_ID}-${APP_VERSION}.jar"

                        ARTIFACT_URL="${NEXUS_URL}/repository/${NEXUS_REPOSITORY}/${GROUP_PATH}/${ARTIFACT_ID}/${APP_VERSION}/${ARTIFACT_ID}-${APP_VERSION}.jar"

                        echo "Artifact:"
                        echo "${ARTIFACT_FILE}"

                        echo "Repository path:"
                        echo "${GROUP_PATH}/${ARTIFACT_ID}/${APP_VERSION}/"

                        echo "Uploading JAR to Nexus..."

                        curl -f \
                            -u "${NEXUS_USERNAME}:${NEXUS_PASSWORD}" \
                            --upload-file "${ARTIFACT_FILE}" \
                            "${ARTIFACT_URL}"

                        echo ""
                        echo "======================================"
                        echo "NEXUS JAR UPLOAD SUCCESSFUL"
                        echo "======================================"
                    '''
                }
            }
        }


        // =========================================================
        // 12. DOCKER BUILD
        // =========================================================
        stage('Docker Build') {
            steps {

                echo 'Building Docker image...'

                sh '''
                    docker build \
                        -t ${APP_NAME}:build-${BUILD_NUMBER} .
                '''
            }
        }


        // =========================================================
        // 13. TRIVY DOCKER IMAGE SCAN
        // =========================================================
        stage('Trivy Image Scan') {
            steps {

                echo 'Scanning Docker image for vulnerabilities...'

                sh '''
                    trivy image \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        ${APP_NAME}:build-${BUILD_NUMBER}
                '''
            }
        }
    }


    // =============================================================
    // POST ACTIONS
    // =============================================================
    post {

        always {

            echo 'Cleaning temporary Docker image...'

            sh '''
                docker image rm \
                    ${APP_NAME}:build-${BUILD_NUMBER} \
                    2>/dev/null || true
            '''
        }


        success {

            echo '''
            ======================================
                 PIPELINE SUCCESSFUL
            ======================================
            '''
        }


        failure {

            echo '''
            ======================================
                 PIPELINE FAILED
            ======================================
            '''
        }
    }
}
