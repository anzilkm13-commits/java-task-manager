pipeline {

    agent any

    environment {
        DOCKER_IMAGE = 'anzilkm/java-task-manager'
        IMAGE_TAG = "${BUILD_NUMBER}"

        NEXUS_URL = 'http://172.31.13.19:8081'
        NEXUS_REPOSITORY = 'maven-snapshots'

        GROUP_ID = 'com.example'
        ARTIFACT_ID = 'spring-boot-todo-applicationx'
        APP_VERSION = '0.0.1-SNAPSHOT'

        KUBECONFIG = '/var/lib/jenkins/.kube/config'
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
                    echo "Java:"
                    java -version

                    echo "Maven:"
                    ./mvnw -version

                    echo "Docker:"
                    docker --version

                    echo "Trivy:"
                    trivy --version

                    echo "Kubectl:"
                    kubectl version --client
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
                sh './mvnw package -DskipTests'
            }
        }

        stage('Publish Artifact to Nexus') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexus-credentials',
                        usernameVariable: 'NEXUS_USERNAME',
                        passwordVariable: 'NEXUS_PASSWORD'
                    )
                ]) {

                    sh '''
                        set -e

                        GROUP_PATH=$(echo "${GROUP_ID}" | tr '.' '/')

                        ARTIFACT_FILE="target/${ARTIFACT_ID}-${APP_VERSION}.jar"

                        ARTIFACT_URL="${NEXUS_URL}/repository/${NEXUS_REPOSITORY}/${GROUP_PATH}/${ARTIFACT_ID}/${APP_VERSION}/${ARTIFACT_ID}-${APP_VERSION}.jar"

                        echo "Uploading artifact to Nexus..."

                        curl -f \
                            -u "${NEXUS_USERNAME}:${NEXUS_PASSWORD}" \
                            --upload-file "${ARTIFACT_FILE}" \
                            "${ARTIFACT_URL}"

                        echo "Nexus upload successful."
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    echo "Building Docker image..."

                    docker build \
                        -t "${DOCKER_IMAGE}:${IMAGE_TAG}" \
                        .

                    echo "Docker image created:"
                    docker images "${DOCKER_IMAGE}"
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                    echo "Scanning Docker image..."

                    trivy image \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        "${DOCKER_IMAGE}:${IMAGE_TAG}"
                '''
            }
        }

        stage('Push Docker Image') {
            steps {

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

                        echo "Pushing image:"
                        echo "${DOCKER_IMAGE}:${IMAGE_TAG}"

                        docker push "${DOCKER_IMAGE}:${IMAGE_TAG}"

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    echo "Deploying application to Kubernetes..."

                    echo "Kubernetes cluster:"
                    kubectl --kubeconfig="${KUBECONFIG}" get nodes

                    echo "Updating deployment image..."

                    kubectl \
                        --kubeconfig="${KUBECONFIG}" \
                        set image deployment/java-task-manager \
                        java-task-manager="${DOCKER_IMAGE}:${IMAGE_TAG}"

                    echo "Waiting for rollout..."

                    kubectl \
                        --kubeconfig="${KUBECONFIG}" \
                        rollout status deployment/java-task-manager \
                        --timeout=180s

                    echo "Deployment successful."

                    echo "Current deployment:"
                    kubectl \
                        --kubeconfig="${KUBECONFIG}" \
                        get deployment java-task-manager

                    echo "Current pods:"
                    kubectl \
                        --kubeconfig="${KUBECONFIG}" \
                        get pods -o wide
                '''
            }
        }
    }

    post {

        success {
            echo """
            ==========================================
            PIPELINE SUCCESSFUL
            ==========================================

            Docker Image:
            ${DOCKER_IMAGE}:${IMAGE_TAG}

            Kubernetes Deployment:
            java-task-manager

            ==========================================
            """
        }

        failure {
            echo """
            ==========================================
            PIPELINE FAILED
            ==========================================

            Check the failed stage in the Jenkins console.

            ==========================================
            """
        }

        always {
            echo "Pipeline execution completed."
        }
    }
}
