pipeline {
    agent any

    environment {
        // ==============================
        // APPLICATION
        // ==============================
        APP_NAME = 'java-task-manager'
        APP_VERSION = '0.0.1-SNAPSHOT'

        // ==============================
        // DOCKER HUB
        // ==============================
        DOCKER_IMAGE = 'anzilkm/java-task-manager'
        DOCKER_TAG = '1.0'

        // ==============================
        // NEXUS
        // ==============================
        NEXUS_URL = 'http://172.31.25.150:8081'
        NEXUS_REPOSITORY = 'maven-snapshots'

        // ==============================
        // SONARQUBE
        // ==============================
        SONAR_URL = 'http://172.31.25.150:9000'
    }

    stages {

        // ==========================================
        // 1. CHECKOUT
        // ==========================================
        stage('Checkout') {
            steps {
                echo '======================================'
                echo 'CHECKING OUT SOURCE CODE'
                echo '======================================'

                checkout scm
            }
        }

        // ==========================================
        // 2. VERIFY TOOLS
        // ==========================================
        stage('Verify Java, Maven & Docker') {
            steps {
                sh '''
                    echo "===== JAVA VERSION ====="
                    java -version

                    echo ""
                    echo "===== MAVEN VERSION ====="
                    mvn -version

                    echo ""
                    echo "===== DOCKER VERSION ====="
                    docker --version

                    echo ""
                    echo "===== GIT VERSION ====="
                    git --version

                    echo ""
                    echo "===== TRIVY VERSION ====="
                    trivy --version
                '''
            }
        }

        // ==========================================
        // 3. CLEAN
        // ==========================================
        stage('Clean') {
            steps {
                echo 'Cleaning previous Maven build...'

                sh '''
                    chmod +x mvnw
                    ./mvnw clean
                '''
            }
        }

        // ==========================================
        // 4. BUILD & TEST
        // ==========================================
        stage('Build & Test') {
            steps {
                echo 'Building application and running tests...'

                sh '''
                    chmod +x mvnw
                    ./mvnw test
                '''
            }
        }

        // ==========================================
        // 5. SONARQUBE
        // ==========================================
        stage('SonarQube Analysis') {
            steps {

                echo 'Running SonarQube code quality analysis...'

                withCredentials([
                    string(
                        credentialsId: 'sonarqube-token-1',
                        variable: 'SONAR_TOKEN'
                    )
                ]) {

                    sh '''
                        chmod +x mvnw

                        ./mvnw sonar:sonar \
                          -Dsonar.host.url=${SONAR_URL} \
                          -Dsonar.token=${SONAR_TOKEN} \
                          -Dsonar.projectKey=java-task-manager \
                          -Dsonar.projectName=java-task-manager
                    '''
                }
            }
        }

        // ==========================================
        // 6. PACKAGE
        // ==========================================
        stage('Package') {
            steps {

                echo 'Packaging Spring Boot application...'

                sh '''
                    chmod +x mvnw

                    ./mvnw package -DskipTests
                '''
            }
        }

        // ==========================================
        // 7. UPLOAD JAR TO NEXUS
        // ==========================================
        stage('Upload JAR to Nexus') {
            steps {

                echo 'Uploading Maven artifact to Nexus...'

                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexus-credentials-1',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASSWORD'
                    )
                ]) {

                    sh '''
                        cat > nexus-settings.xml <<EOF
<settings>
    <servers>
        <server>
            <id>nexus</id>
            <username>${NEXUS_USER}</username>
            <password>${NEXUS_PASSWORD}</password>
        </server>
    </servers>
</settings>
EOF

                        ./mvnw deploy \
                          -DskipTests \
                          -s nexus-settings.xml \
                          -DaltDeploymentRepository=nexus::default::${NEXUS_URL}/repository/${NEXUS_REPOSITORY}/
                    '''
                }
            }
        }

        // ==========================================
        // 8. DOCKER BUILD
        // ==========================================
        stage('Docker Build') {
            steps {

                echo 'Building Docker image...'

                sh '''
                    docker build \
                      -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                '''

                sh '''
                    echo "===== DOCKER IMAGE ====="
                    docker images ${DOCKER_IMAGE}
                '''
            }
        }

        // ==========================================
        // 9. DOCKER TEST
        // ==========================================
        stage('Docker Test') {
            steps {

                echo 'Starting temporary container for application test...'

                sh '''
                    docker rm -f ${APP_NAME}-test 2>/dev/null || true

                    docker run -d \
                      --name ${APP_NAME}-test \
                      -p 8081:8081 \
                      ${DOCKER_IMAGE}:${DOCKER_TAG}

                    echo "Waiting for Spring Boot application..."
                    sleep 20

                    echo "Testing application..."

                    curl --fail \
                      --retry 5 \
                      --retry-delay 3 \
                      http://localhost:8081

                    echo ""
                    echo "Application test successful!"

                    docker rm -f ${APP_NAME}-test
                '''
            }
        }

        // ==========================================
        // 10. TRIVY SECURITY SCAN
        // ==========================================
        stage('Trivy Security Scan') {
            steps {

                echo 'Scanning Docker image for HIGH and CRITICAL vulnerabilities...'

                sh '''
                    trivy image \
                      --severity HIGH,CRITICAL \
                      --exit-code 1 \
                      ${DOCKER_IMAGE}:${DOCKER_TAG}
                '''
            }
        }

        // ==========================================
        // 11. DOCKER HUB PUSH
        // ==========================================
        stage('Docker Hub Push') {
            steps {

                echo 'Logging into Docker Hub and pushing image...'

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials-1',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {

                    sh '''
                        echo "${DOCKER_PASSWORD}" | docker login \
                          --username "${DOCKER_USER}" \
                          --password-stdin

                        docker push ${DOCKER_IMAGE}:${DOCKER_TAG}

                        docker logout
                    '''
                }
            }
        }
    }

    // ==========================================
    // POST ACTIONS
    // ==========================================
    post {

        always {

            echo 'Cleaning temporary resources...'

            sh '''
                docker rm -f ${APP_NAME}-test 2>/dev/null || true

                rm -f nexus-settings.xml
            '''
        }

        success {

            echo '''
========================================
       PIPELINE SUCCESSFUL
========================================

Application:
java-task-manager

Docker Image:
anzilkm/java-task-manager:1.0

Pipeline completed:
GitHub
   ↓
Maven Build & Test
   ↓
SonarQube
   ↓
Maven Package
   ↓
Nexus
   ↓
Docker Build
   ↓
Docker Test
   ↓
Trivy Security Scan
   ↓
Docker Hub Push

========================================
'''
        }

        failure {

            echo '''
========================================
        PIPELINE FAILED
========================================

Check the failed stage above.

========================================
'''
        }
    }
}
