pipeline {

    agent any

    tools {
        maven 'Maven'
    }

    environment {
        IMAGE_NAME = "vinothkumaraws/owasp-app"
        IMAGE_TAG = "${BUILD_NUMBER}"
        APP_SERVER = "13.50.236.28"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar'
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                script {

                    def dcHome = tool 'DependencyCheck'

                    withCredentials([
                        string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')
                    ]) {

                        sh '''
                            mkdir -p dependency-check-report
                        '''

                        sh """
                            ${dcHome}/bin/dependency-check.sh \
                            --project "OWASP-Jenkins" \
                            --scan . \
                            --out dependency-check-report \
                            --format HTML \
                            --format XML \
                            --nvdApiKey \$NVD_API_KEY
                        """
                    }
                }
            }
        }

        stage('Publish OWASP Report') {
            steps {
                dependencyCheckPublisher(
                    pattern: 'dependency-check-report/dependency-check-report.xml'
                )
            }
        }

        stage('Docker Build') {
            steps {
                sh """
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
                """
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                    '''
                }
            }
        }

        stage('Docker Push') {
            steps {
                sh """
                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    docker push ${IMAGE_NAME}:latest
                """
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent(['ec2-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${APP_SERVER} << EOF

                        docker pull ${IMAGE_NAME}:latest

                        docker stop java-app || true

                        docker rm java-app || true

                        docker run -d \
                          --name java-app \
                          -p 8080:8080 \
                          ${IMAGE_NAME}:latest

                        EOF
                    """
                }
            }
        }
    }

    post {

        success {
            echo "===================================="
            echo "PIPELINE COMPLETED SUCCESSFULLY"
            echo "===================================="
        }

        failure {
            echo "===================================="
            echo "PIPELINE FAILED"
            echo "===================================="
        }

        always {
            archiveArtifacts artifacts: 'dependency-check-report/**/*', fingerprint: true
            cleanWs()
        }
    }
}
