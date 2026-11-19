pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS_ID = 'docker-cred'
        DOCKER_IMAGE          = 'adhil6407/friday-chatbot'

        SONAR_SERVER_NAME     = 'sonarqube_env'
        SONAR_SCANNER_TOOL    = 'sonar-scanner'

        GIT_CREDENTIALS_ID    = 'github-cred'
        MANIFEST_REPO         = 'https://github.com/adhil6407/friday-chatbot.git'
        MANIFEST_BRANCH       = 'main'
    }

    stages {

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool "${SONAR_SCANNER_TOOL}"

                    withSonarQubeEnv("${SONAR_SERVER_NAME}") {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                                -Dsonar.projectKey=friday-chatbot-key \
                                -Dsonar.projectName=friday-chatbot-project \
                                -Dsonar.sources=. \
                                -Dsonar.exclusions=**/venv/**,**/.venv/**,**/__pycache__/**,**/.pytest_cache/**,**/static/** \
                                -Dsonar.python.version=3.11
                        """
                    }
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: "${DOCKER_CREDENTIALS_ID}",
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )
                    ]) {

                        sh '''
                            echo "$DOCKER_PASS" | docker login \
                                -u "$DOCKER_USER" \
                                --password-stdin

                            echo "Checking required files..."
                            ls -la

                            test -f Dockerfile
                            test -f requirements.txt
                            test -f main.py

                            docker build \
                                -t ${DOCKER_IMAGE}:${BUILD_NUMBER} \
                                -t ${DOCKER_IMAGE}:latest .

                            docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                            docker push ${DOCKER_IMAGE}:latest

                            docker logout
                        '''
                    }
                }
            }
        }

        stage('Update Manifest Repo') {
            steps {
                script {

                    dir('manifest-workspace') {

                        deleteDir()

                        withCredentials([
                            usernamePassword(
                                credentialsId: "${GIT_CREDENTIALS_ID}",
                                usernameVariable: 'GH_USER',
                                passwordVariable: 'GH_TOKEN'
                            )
                        ]) {

                            sh '''
                                git clone \
                                    -b "$MANIFEST_BRANCH" \
                                    "https://$GH_USER:$GH_TOKEN@github.com/adhil6407/friday-chatbot.git" \
                                    .

                                git config user.name "adhilzaed"
                                git config user.email "adhilzaed@gmail.com"

                                echo "Updating Kubernetes image..."

                                sed -i "/name: friday-chatbot/{n;s|image:.*|image: ${DOCKER_IMAGE}:${BUILD_NUMBER}|}" \
                                    k8s/Deployment.yaml

                                git add k8s/Deployment.yaml

                                git diff --cached --quiet || \
                                    git commit -m "chore(manifest): update friday-chatbot image tag to ${BUILD_NUMBER} [skip ci]"

                                git push origin "$MANIFEST_BRANCH"
                            '''
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo "========================================"
            echo "Pipeline completed successfully!"
            echo "Docker Image: ${DOCKER_IMAGE}:${BUILD_NUMBER}"
            echo "========================================"
        }

        failure {
            echo "========================================"
            echo "Pipeline FAILED!"
            echo "Check the stage above for the error."
            echo "========================================"
        }

        always {
            echo "Jenkins build: ${BUILD_NUMBER}"
        }
    }
}
