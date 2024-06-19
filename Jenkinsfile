pipeline {
    agent any

    environment {
        REGISTRY = 'akash4ocpl'
        REPOSITORY = 'backend'
        IMAGE_NAME = "${env.REGISTRY}/${env.REPOSITORY}"
        DOCKER_CREDENTIALS_ID = 'docker-hub-credentials' // ID used in Jenkins
        GITHUB_CREDENTIALS_ID = '1234567890987654321' // GitHub credentials ID
        KUBECONFIG_CREDENTIALS_ID = 'kubeconfig-credentials' // ID used in Jenkins
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    git branch: 'main', credentialsId: env.GITHUB_CREDENTIALS_ID, url: 'https://github.com/akash4ocpl/backend.git'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def commitHash = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    def previousVersion = sh(script: "docker images ${env.IMAGE_NAME} --format '{{.Tag}}' | grep -E '^version-' | sort -r | head -1 | awk -F '-' '{print \$2}'", returnStdout: true).trim()
                    def newVersion = (previousVersion.isInteger() ? previousVersion.toInteger() + 1 : 1)

                    // Build Docker image with 'latest' tag and version tag
                    sh "docker build -t ${env.IMAGE_NAME}:latest -t ${env.IMAGE_NAME}:version-${newVersion} -t ${env.IMAGE_NAME}:${commitHash} ."
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry([credentialsId: env.DOCKER_CREDENTIALS_ID, url: '']) {
                        sh "docker push ${env.IMAGE_NAME}:latest"
                        sh "docker push ${env.IMAGE_NAME}:version-${newVersion}"
                        sh "docker push ${env.IMAGE_NAME}:${commitHash}"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    withCredentials([file(credentialsId: env.KUBECONFIG_CREDENTIALS_ID, variable: 'KUBECONFIG')]) {
                        sh """
                        kubectl apply -f deployment.yml
                        kubectl apply -f service.yml
                        """
                    }
                }
            }
        }

        stage('Remove Latest Tag from Previous Image') {
            steps {
                script {
                    def previousLatestDigest = sh(script: "docker images --filter=reference=${env.IMAGE_NAME}:latest --format '{{.ID}}'", returnStdout: true).trim()
                    if (previousLatestDigest) {
                        sh "docker rmi ${env.IMAGE_NAME}:latest"
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                // Safely remove images if they exist
                sh "docker rmi ${env.IMAGE_NAME}:latest || true"
                sh "docker rmi ${env.IMAGE_NAME}:${commitHash} || true"
                sh "docker rmi ${env.IMAGE_NAME}:version-${newVersion} || true"
            }
        }
    }
}
