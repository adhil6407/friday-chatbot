pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS_ID = 'docker-cred'
        DOCKER_IMAGE          = 'adhil6407/fast-api-chatbot'
        SONAR_SERVER_NAME     = 'sonarqube_env'
        SONAR_SCANNER_TOOL    = 'sonar-scanner'
        GIT_CREDENTIALS_ID    = 'github-cred'
        MANIFEST_REPO         = 'https://github.com/adhil6407/api-chatbot.git'
        MANIFEST_BRANCH       = 'main'
    }

    stages {

        stage('Checkout Source Code') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool "${SONAR_SCANNER_TOOL}"

                    withSonarQubeEnv("${SONAR_SERVER_NAME}") {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                                -Dsonar.projectKey=fast-api-chatbot-key \
                                -Dsonar.projectName='fast-api-chatbot-project' \
                                -Dsonar.sources=. \
                                -Dsonar.exclusions=**/venv/**,**/.venv/**,**/__pycache__/**,**/.pytest_cache/**,**/static/**
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
                        sh """
                            echo "\$DOCKER_PASS" | docker login -u "\$DOCKER_USER" --password-stdin

                            docker build \
                                -t ${DOCKER_IMAGE}:${env.BUILD_NUMBER} \
                                -t ${DOCKER_IMAGE}:latest .

                            docker push ${DOCKER_IMAGE}:${env.BUILD_NUMBER}
                            docker push ${DOCKER_IMAGE}:latest
                        """
                    }
                }
            }
        }

        stage('Update Manifest Repo') {
            steps {
                script {
                    dir('manifest-workspace') {

                        sh """
                            git clone ${MANIFEST_REPO} .
                        """

                        withCredentials([
                            usernamePassword(
                                credentialsId: "${GIT_CREDENTIALS_ID}",
                                usernameVariable: 'GH_USER',
                                passwordVariable: 'GH_TOKEN'
                            )
                        ]) {
                            sh """
                                git remote set-url origin https://\$GH_USER:\$GH_TOKEN@github.com/adhil6407/api-chatbot.git

                                git pull --rebase origin ${MANIFEST_BRANCH}

                                sed -i '/name: fast-api-chatbot/{n;s|image:.*|image: ${DOCKER_IMAGE}:${env.BUILD_NUMBER}|}' k8s/Deployment.yaml

                                git config user.name "adhilzaed"
                                git config user.email "adhilzaed@gmail.com"

                                git add k8s/Deployment.yaml

                                git commit -m "chore(manifest): update fast-api-chatbot image tag to ${env.BUILD_NUMBER} [skip ci]" || true

                                git push origin ${MANIFEST_BRANCH}
                            """
                        }
                    }
                }
            }
        }
    }
}
