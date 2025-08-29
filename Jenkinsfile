pipeline {
    agent any

    options {
        timeout(time: 1, unit: 'HOURS')
    }
    
    environment {
        AWS_ECR_LOGIN = 'OMITTED'
        DOCKER_IMAGE = 'wordsmyth-w2-prod'
        ECR_REPO = 'OMITTED'
        DEPLOYMENT_NAME = 'wordsmyth-w2-prod-deployment'
        NAMESPACE = 'prod'
        COMMIT_HASH = "${env.GIT_COMMIT}"
        DISCORD_WEBHOOK_URL = 'OMITTED'
        SUCCESS_COLOR = '3066993'  // Green
        FAILURE_COLOR = '15158332' // Red
        
        // SSH Configuration
        REMOTE_SERVER = 'ubuntu@ip-11-0-1-168'
        REMOTE_DIR = '/home/ubuntu/repos'
        SSH_CREDENTIALS = 'ssh-credentials-id'  // Add this credential in Jenkins
    }

    stages {
        stage('Clone Repository') {
            steps {
                script {
                    // Checkout the repository
                    checkout scm
                    
                    // Set commit information using environment variables
                    env.COMMIT_AUTHOR = sh(
                        script: 'git show -s --pretty=%an',
                        returnStdout: true
                    ).trim()
                    
                    env.COMMIT_MESSAGE = sh(
                        script: 'git show -s --pretty=%B',
                        returnStdout: true
                    ).trim()
                    
                    // Print for debugging
                    echo "Commit Author: ${env.COMMIT_AUTHOR}"
                    echo "Commit Message: ${env.COMMIT_MESSAGE}"
                }
            }
        }

        stage('Building Docker Image') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}:${COMMIT_HASH}")
                }
            }
        }
        
        stage('Tag Docker Image') {
            steps {
                script {
                    sh "docker tag ${DOCKER_IMAGE}:${COMMIT_HASH} ${ECR_REPO}:${COMMIT_HASH}"
                }
            }
        }

        stage('Login to AWS ECR') {
            steps {
                sh "${AWS_ECR_LOGIN}"
            }
        }

        stage('Pushing Docker Image to AWS ECR') {
            steps {
                script {
                    sh "docker push ${ECR_REPO}:${COMMIT_HASH}"
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sshagent([env.SSH_CREDENTIALS]) {
                    script {
                        // SSH into the remote server and update the deployment file
                        sh """
                            ssh -o StrictHostKeyChecking=no ${REMOTE_SERVER} '
                                cd ${REMOTE_DIR}
                                # Update the image in the deployment YAML using sed
                                sed -i "s|image: ${ECR_REPO}:[[:alnum:]]*|image: ${ECR_REPO}:${COMMIT_HASH}|g" frontend-wordsmyth.yaml
                                # Apply the updated deployment
                                kubectl apply -f frontend-wordsmyth.yaml
                            '
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                def message = "Hello Wordsmyth Team!"
                def description = """Deployment of WordsMyth-Frontendv1 through Jenkins Pipeline ಯಶಸ್ವಿಯಾಗಿದೆ! 🚀

**Committed by:** ${env.COMMIT_AUTHOR}
**Commit Message:** ${env.COMMIT_MESSAGE}
**Commit Hash:** ${env.COMMIT_HASH.take(7)}"""

                notifyDiscord(message, description, env.SUCCESS_COLOR)
            }
        }
        
        failure {
            script {
                def message = "⚠️ Attention Wordsmyth Team!"
                def description = """Deployment of WordsMyth-Frontendv1 ವಿಫಲವಾಗಿದೆ! ❌

**Committed by:** ${env.COMMIT_AUTHOR}
**Commit Message:** ${env.COMMIT_MESSAGE}
**Commit Hash:** ${env.COMMIT_HASH.take(7)}
**Failed Stage:** ${currentBuild.previousBuild?.result == 'FAILURE' ? 'Multiple stages' : currentBuild.currentResult}
**Error Details:** ${currentBuild.description ?: 'See Jenkins logs for details'}"""

                notifyDiscord(message, description, env.FAILURE_COLOR)
            }
        }
        
        always {
            cleanWs()
        }
    }

    triggers {
        githubPush()
    }
}

// Helper function to send Discord notifications
def notifyDiscord(message, description, color) {
    def timestamp = new Date().format("yyyy-MM-dd'T'HH:mm:ss.SSS'Z'", TimeZone.getTimeZone('UTC'))
    
    // Escape special characters in description
    description = description.replaceAll('"', '\\\\"').replaceAll('\n', '\\\\n')
    
    sh """
        curl -H "Content-Type: application/json" -X POST -d '{
            "content": "${message}",
            "embeds": [{
                "title": "WordsMyth-Frontendv1 Deployment ಮಾಹಿತಿ:",
                "description": "${description}",
                "color": ${color},
                "footer": {
                    "text": "Jenkins Build #${env.BUILD_NUMBER}"
                },
                "timestamp": "${timestamp}"
            }]
        }' ${DISCORD_WEBHOOK_URL}
    """
}
