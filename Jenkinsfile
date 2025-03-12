pipeline {
    agent any

    environment {
        DOCKER_IMAGE_NAME = "ezequielmirallas/simple-nodejs"
    }
    
    stages {
        stage('Build Docker Image') {
             steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', passwordVariable: 'DOCKER_HUB_PASSWORD', usernameVariable: 'DOCKER_HUB_USERNAME')]) {
                    sh "echo ${DOCKER_HUB_PASSWORD} | docker login -u ${DOCKER_HUB_USERNAME} --password-stdin"
                    sh "docker build -t ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER} ."
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    sh "docker run --rm ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER} npm test"
                }
            }
        }
    
        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', passwordVariable: 'DOCKER_HUB_PASSWORD', usernameVariable: 'DOCKER_HUB_USERNAME')]) {
                        sh "echo ${DOCKER_HUB_PASSWORD} | docker login -u ${DOCKER_HUB_USERNAME} --password-stdin"
                        sh "docker push ${DOCKER_IMAGE_NAME}:${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
                        
                        // Tag as latest only if on the main branch
                        if (env.BRANCH_NAME == 'main') {
                            sh "docker tag ${DOCKER_IMAGE_NAME}:${env.BRANCH_NAME}-${env.BUILD_NUMBER} ${DOCKER_IMAGE_NAME}:latest"
                            sh "docker push ${DOCKER_IMAGE_NAME}:latest"
                        }
                    }
                }
            }
        }
        
        stage('Deploy Approval') {
            when {
                branch 'main'
            }
            steps {
                script {
                    input message: 'Deploy to production?', ok: 'Yes, deploy'
                }
            }
        }
    
        stage('Deploy PROD') {    
            when { 
                branch 'main'
            }
            steps {
                script {
                    withCredentials([sshUserPrivateKey(credentialsId: 'ssh-user-aws', keyFileVariable: 'SSH_KEY')]) {
                        sh """
                            ssh -o StrictHostKeyChecking=no -i ${SSH_KEY} ${DEPLOY_USER}@${DEPLOY_SERVER} bash -c '
                                docker pull ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}
                                docker stop simple-nodejs || true
                                docker rm simple-nodejs || true
                                docker run -d --name simple-nodejs -p 3000:3000 ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}
                            '
                        """
                    }
                }
            }
        }
    }   

    post {
        always {
            script {
                try {
                    sh "docker rmi ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
                    sh "docker rmi ${DOCKER_IMAGE_NAME}:latest"
                } catch (Exception e) {
                    echo 'Failed to remove Docker image.'
                }
            }
        }
    }
}