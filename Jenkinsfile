pipeline {
    agent any

    environment {
        REGISTRY = 'akash4ocpl'
        REPOSITORY = 'backend'
        IMAGE_NAME = "${env.REGISTRY}/${env.REPOSITORY}"
        DOCKER_CREDENTIALS_ID = '1234567890987654321' // ID used in Jenkins
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
                    // Get the current commit hash
                    def commitHash = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    echo "Commit Hash: ${commitHash}"

                    // Get the previous version
                    def previousVersion = sh(script: """
                        docker images ${env.IMAGE_NAME} --format '{{.Tag}}' | grep -E '^version-' | sort -r | head -1 | awk -F '-' '{print \$2}'
                    """, returnStdout: true).trim()
                    echo "Previous Version: ${previousVersion}"

                    // Determine the new version
                    def newVersion
                    if (previousVersion.isInteger()) {
                        newVersion = previousVersion.toInteger() + 1
                    } else {
                        newVersion = 1
                    }
                    echo "New Version: ${newVersion}"

                    // List files to debug the Dockerfile location
                    sh "ls -la"

                    // Build Docker image with 'latest' tag and version tag
                    echo "Building Docker image with tags: latest, version-${newVersion}, ${commitHash}"
                    sh "docker build -t ${env.IMAGE_NAME}:latest -t ${env.IMAGE_NAME}:version-${newVersion} -t ${env.IMAGE_NAME}:${commitHash} ."
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    try {
                        // Login to DockerHub using credentials
                        withCredentials([usernamePassword(credentialsId: env.DOCKER_CREDENTIALS_ID, passwordVariable: 'DOCKERHUB_PASSWORD', usernameVariable: 'DOCKERHUB_USERNAME')]) {
                            sh 'echo $DOCKERHUB_PASSWORD | docker login -u $DOCKERHUB_USERNAME --password-stdin'
                        }
                        echo "Docker login successful"

                        // Push the Docker image to DockerHub
                        sh "docker push ${env.IMAGE_NAME}:latest"
                        sh "docker push ${env.IMAGE_NAME}:version-${newVersion}"
                        sh "docker push ${env.IMAGE_NAME}:${commitHash}"

                        echo "Docker images pushed successfully"
                    } catch (Exception e) {
                        echo "Error during Docker image push: ${e.message}"
                        throw e
                    }
                }
            }
        }


        stage('Deploy to Kubernetes') {
            steps {
                script {
                    withCredentials([file(credentialsId: env.KUBECONFIG_CREDENTIALS_ID, variable: 'KUBECONFIG')]) {
                        sh 'echo "KUBECONFIG contents:"'
                        sh 'cat $KUBECONFIG'
                        sh 'kubectl get nodes'
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
