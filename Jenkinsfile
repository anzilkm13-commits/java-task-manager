pipeline {
    agent any

    environment {
        APP_NAME = 'java-task-manager'
        APP_PORT = '8081'

        // Nexus
        NEXUS_URL = 'http://172.31.13.19:8081'
        NEXUS_REPOSITORY = 'maven-snapshots'
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
                    echo "===== JAVA ====="
                    java --version

                    echo "===== MAVEN ====="
                    ./mvnw --version

                    echo "===== GIT ====="
                    git --version

                    echo "===== DOCKER ====="
                    docker --version

                    echo "===== TRIVY ====="
                    trivy --version
                '''
            }
        }

        stage('Clean') {
            steps {
                echo 'Cleaning project...'

                sh './mvnw clean'
            }
        }

        stage('Compile') {
            steps {
                echo 'Compiling Java application...'

                sh './mvnw compile'
            }
        }

        stage('Test') {
            steps {
                echo 'Running unit tests...'

                sh './mvnw test'
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                echo 'Scanning project files for vulnerabilities...'

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
                echo 'Running SonarQube code quality analysis...'

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
                echo 'Packaging Spring Boot application...'

                sh './mvnw package -DskipTests'

                echo 'Generated JAR files:'

                sh '''
                    ls -lh target/*.jar
                '''
            }
        }

        stage('Publish to Nexus') {
            steps {
                echo 'Uploading JAR to Nexus Repository...'

                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexus-credentials',
                        usernameVariable: 'NEXUS_USERNAME',
                        passwordVariable: 'NEXUS_PASSWORD'
                    )
                ]) {

                    sh '''
                        JAR_FILE=$(find target -maxdepth 1 -name "*.jar" \
                            ! -name "*-sources.jar" \
                            ! -name "*-javadoc.jar" \
                            | head -n 1)

                        echo "JAR file: $JAR_FILE"

                        curl -f \
                            -u "$NEXUS_USERNAME:$NEXUS_PASSWORD" \
                            --upload-file "$JAR_FILE" \
                            "${NEXUS_URL}/repository/${NEXUS_REPOSITORY}/$(basename "$JAR_FILE")"
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                echo 'Building Docker image...'

                sh '''
                    docker build \
                        -t ${APP_NAME}:build-${BUILD_NUMBER} .
                '''

                echo 'Docker image created:'

                sh '''
                    docker images ${APP_NAME}
                '''
            }
        }

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
            echo '======================================'
            echo 'PIPELINE SUCCESSFUL'
            echo '======================================'
        }

        failure {
            echo '======================================'
            echo 'PIPELINE FAILED'
            echo '======================================'
        }
    }
}
