pipeline {
    agent any

    environment {
        APP_NAME = 'java-task-manager'
        APP_PORT = '8081'
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
                sh './mvnw clean'
            }
        }

        stage('Compile') {
            steps {
                sh './mvnw compile'
            }
        }

        stage('Test') {
            steps {
                sh './mvnw test'
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
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
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        ./mvnw sonar:sonar \
                            -Dsonar.projectKey=java-task-manager \
                            -Dsonar.projectName=java-task-manager
                    '''
                }
            }
        }

        stage('Package') {
            steps {
                sh './mvnw package -DskipTests'
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build \
                        -t ${APP_NAME}:build-${BUILD_NUMBER} .
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
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
