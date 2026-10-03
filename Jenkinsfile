pipeline {
    agent any

    environment {
        APP_NAME = 'java-task-manager'
        APP_VERSION = '0.0.1-SNAPSHOT'

        DOCKER_IMAGE = 'anzilkm/java-task-manager'
        DOCKER_TAG = '1.0'

        NEXUS_URL = 'http://172.31.25.150:8081'
        NEXUS_REPOSITORY = 'maven-snapshots'

        SONAR_URL = 'http://172.31.25.150:9000'
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out source code...'

                checkout scm
            }
        }

        stage('Verify Java & Maven') {
            steps {
                sh '''
                    echo "===== JAVA ====="
                    java -version

                    echo "===== MAVEN ====="
                    mvn -version
                '''
            }
        }

        stage('Clean') {
            steps {
                sh './mvnw clean'
            }
        }

        stage('Build & Test') {
            steps {
                sh './mvnw test'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'sonarqube-token',
                        variable: 'SONAR_TOKEN'
                    )
                ]) {
                    sh '''
                        ./mvnw sonar:sonar \
                          -Dsonar.host.url=${SONAR_URL} \
                          -Dsonar.token=${SONAR_TOKEN} \
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

        stage('Upload JAR to Nexus') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexus-credentials',
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
                          -DaltDeploymentRepository=nexus::${NEXUS_URL}/repository/${NEXUS_REPOSITORY}/
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build \
                      -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                '''
            }
        }

        stage('Docker Test') {
            steps {
                sh '''
                    docker rm -f ${APP_NAME}-test 2>/dev/null || true

                    docker run -d \
                      --name ${APP_NAME}-test \
                      -p 8081:8081 \
                      ${DOCKER_IMAGE}:${DOCKER_TAG}

                    echo "Waiting for application..."
                    sleep 15

                    curl --fail http://localhost:8081

                    docker rm -f ${APP_NAME}-test
                '''
            }
        }

        stage('Trivy Security Scan') {
            steps {
                sh '''
                    trivy image \
                      --severity HIGH,CRITICAL \
                      --exit-code 1 \
                      ${DOCKER_IMAGE}:${DOCKER_TAG}
                '''
            }
        }

        stage('Docker Hub Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
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

    post {
        always {
            sh '''
                docker rm -f ${APP_NAME}-test 2>/dev/null || true
                rm -f nexus-settings.xml
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
            echo 'Check the failed stage'
            echo '======================================'
        }
    }
}
