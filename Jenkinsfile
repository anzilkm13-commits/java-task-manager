pipeline {

    agent any

    environment {
        APP_NAME = 'java-task-manager'
        APP_PORT = '8081'

        // Nexus
        NEXUS_URL = 'http://172.31.13.19:8081'
        NEXUS_REPOSITORY = 'maven-snapshots'

        // Maven coordinates
        GROUP_ID = 'com.example'
        ARTIFACT_ID = 'spring-boot-todo-applicationx'
        APP_VERSION = '0.0.1-SNAPSHOT'

        // Docker Hub
        DOCKER_IMAGE = 'anzilkm/java-task-manager:1.0'
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }

        stage('Verify Tools') {
            steps {
                sh '''
                    echo "===== Java ====="
                    java -version

                    echo "===== Maven ====="
                    ./mvnw -version

                    echo "===== Git ====="
                    git --version

                    echo "===== Docker ====="
                    docker --version

                    echo "===== Trivy ====="
                    trivy --version
                '''
            }
        }

        stage('Clean') {
            steps {
                echo 'Cleaning previous build files...'

                sh '''
                    ./mvnw clean
                '''
            }
        }

        stage('Compile') {
            steps {
                echo 'Compiling application...'

                sh '''
                    ./mvnw compile
                '''
            }
        }

        stage('Test') {
            steps {
                echo 'Running unit tests...'

                sh '''
                    ./mvnw test
                '''
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                echo 'Running Trivy filesystem security scan...'

                sh '''
                    trivy fs \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        .
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo 'Running SonarQube analysis...'

                withSonarQubeEnv('SonarQube') {
                    sh '''
                        ./mvnw org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                            -Dsonar.projectKey=java-task-manager \
                            -Dsonar.projectName=java-task-manager
                    '''
                }
            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                echo 'Waiting for SonarQube Quality Gate...'

                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Package') {
            steps {
                echo 'Creating application JAR...'

                sh '''
                    ./mvnw package -DskipTests
                '''
            }
        }

        stage('Test Nexus Authentication') {
            steps {

                echo 'Testing Nexus authentication...'

                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexus-credentials',
                        usernameVariable: 'NEXUS_USERNAME',
                        passwordVariable: 'NEXUS_PASSWORD'
                    )
                ]) {

                    sh '''
                        HTTP_CODE=$(curl -s -o /dev/null \
                            -w "%{http_code}" \
                            -u "${NEXUS_USERNAME}:${NEXUS_PASSWORD}" \
                            "${NEXUS_URL}/service/rest/v1/status")

                        echo "Nexus HTTP status: ${HTTP_CODE}"

                        if [ "$HTTP_CODE" != "200" ]; then
                            echo "Nexus authentication failed!"
                            exit 1
                        fi

                        echo "NEXUS AUTHENTICATION SUCCESSFUL"
                    '''
                }
            }
        }

        stage('Publish to Nexus') {
            steps {

                echo 'Publishing JAR to Nexus...'

                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexus-credentials',
                        usernameVariable: 'NEXUS_USERNAME',
                        passwordVariable: 'NEXUS_PASSWORD'
                    )
                ]) {

                    sh '''
                        GROUP_PATH=$(echo "${GROUP_ID}" | tr '.' '/')

                        ARTIFACT_FILE="target/${ARTIFACT_ID}-${APP_VERSION}.jar"

                        ARTIFACT_URL="${NEXUS_URL}/repository/${NEXUS_REPOSITORY}/${GROUP_PATH}/${ARTIFACT_ID}/${APP_VERSION}/${ARTIFACT_ID}-${APP_VERSION}.jar"

                        echo "Uploading artifact to Nexus..."
                        echo "Artifact: ${ARTIFACT_FILE}"

                        curl -f \
                            -u "${NEXUS_USERNAME}:${NEXUS_PASSWORD}" \
                            --upload-file "${ARTIFACT_FILE}" \
                            "${ARTIFACT_URL}"

                        echo "JAR successfully uploaded to Nexus!"
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {

                echo 'Building Docker image...'

                sh '''
                    docker build \
                        -t "${DOCKER_IMAGE}" \
                        .
                '''

                echo 'Docker image built successfully.'
            }
        }

        stage('Trivy Image Scan') {
            steps {

                echo 'Scanning Docker image with Trivy...'

                sh '''
                    trivy image \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        "${DOCKER_IMAGE}"
                '''
            }
        }

        stage('Push Docker Image') {
            steps {

                echo 'Logging in to Docker Hub and pushing image...'

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {

                    sh '''
                        echo "${DOCKER_PASSWORD}" | \
                            docker login \
                            -u "${DOCKER_USERNAME}" \
                            --password-stdin

                        echo "Pushing Docker image..."

                        docker push "${DOCKER_IMAGE}"

                        echo "Docker image pushed successfully!"

                        docker logout
                    '''
                }
            }
        }
    }

    post {

        always {
            echo 'Pipeline completed.'

            sh '''
                docker image rm "${DOCKER_IMAGE}" || true
            '''
        }

        success {
            echo '=========================================='
            echo 'PIPELINE SUCCESSFUL'
            echo 'Docker image pushed to Docker Hub'
            echo '=========================================='
        }

        failure {
            echo '=========================================='
            echo 'PIPELINE FAILED'
            echo 'Check the Jenkins console output'
            echo '=========================================='
        }
    }
}
