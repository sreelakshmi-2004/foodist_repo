
pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS_ID = 'docker-cred'
        DOCKER_IMAGE          = 'sreelakshmins2004/foodfinda'
        SONAR_SERVER_NAME     = 'sonarqube-env'
        SONAR_SCANNER_TOOL    = 'sonar-scanner'
        GIT_CREDENTIALS_ID    = 'github-cred'

        // Update this URL if your actual manifest repository
        // has a different name.
        MANIFEST_REPO         = 'https://github.com/sreelakshmi-2004/foodfinda_pipeline.git'
        MANIFEST_BRANCH       = 'main'
    }

    stages {

        // Stage 1: Checkout source code
        stage('Checkout Source Code') {
            steps {
                checkout scm
                echo 'Foodfinda source code checked out successfully!'
            }
        }

        // Stage 2: SonarQube Analysis
        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool "${SONAR_SCANNER_TOOL}"

                    withSonarQubeEnv("${SONAR_SERVER_NAME}") {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=foodfinda-key \
                            -Dsonar.projectName='foodfinda-project' \
                            -Dsonar.sources=.
                        """
                    }
                }
            }
        }

        // Stage 3: Build and Push Docker Image
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
                            echo "\$DOCKER_PASS" | docker login \
                            -u "\$DOCKER_USER" --password-stdin

                            docker build \
                            -t ${DOCKER_IMAGE}:${env.BUILD_NUMBER} \
                            -t ${DOCKER_IMAGE}:latest \
                            -f ./Dockerfile .

                            docker push ${DOCKER_IMAGE}:${env.BUILD_NUMBER}
                            docker push ${DOCKER_IMAGE}:latest

                            docker logout
                        """
                    }
                }
            }
        }

        // Stage 4: Update Kubernetes Manifest Repository
        stage('Update Manifest Repo') {
            steps {
                dir('manifest-workspace') {

                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/${MANIFEST_BRANCH}"]],
                        userRemoteConfigs: [[
                            url: "${MANIFEST_REPO}",
                            credentialsId: "${GIT_CREDENTIALS_ID}"
                        ]]
                    ])

                    sh """
                        sed -i '/name: foodfinda/{n;s|image:.*|image: ${DOCKER_IMAGE}:${env.BUILD_NUMBER}|;}' manifests/deployment.yaml

                        git config user.name "sreelakshmi-2004"
                        git config user.email "sreelakshmins873@gmail.com"

                        git add manifests/deployment.yaml

                        git commit -m "chore(manifest): update foodfinda image tag to ${env.BUILD_NUMBER} [skip ci]"

                        git push origin ${MANIFEST_BRANCH}
                    """
                }
            }
        }
    }

    post {
        success {
            echo 'Foodfinda pipeline executed successfully! Manifest repository updated.'
        }

        failure {
            echo 'Foodfinda pipeline failed. Please check the stage logs.'
        }
    }
}
