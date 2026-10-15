pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'saathwik63/student-placement-portal'
        DOCKER_CREDENTIALS_ID = 'docker-hub-credentials'
        GIT_CREDENTIALS_ID = 'github-credentials'
        K8S_CREDENTIALS_ID = 'k8s-kubeconfig'
        DEPLOYMENT_NAME = 'placement-portal-deployment'
        NAMESPACE = 'default'
        
        DOCKER_BIN = 'C:\\Users\\sanik\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo 'Checking out code from Git repository...'
                checkout([$class: 'GitSCM', 
                    branches: [[name: '*/main']], 
                    userRemoteConfigs: [[url: 'https://github.com/mairalsanika/SANIKAA.git', credentialsId: "${env.GIT_CREDENTIALS_ID}"]]
                ])
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                script {
                    bat "\"${env.DOCKER_BIN}\" build -t ${env.DOCKER_IMAGE}:${env.BUILD_NUMBER} --no-cache ."
                    bat "\"${env.DOCKER_BIN}\" tag ${env.DOCKER_IMAGE}:${env.BUILD_NUMBER} ${env.DOCKER_IMAGE}:latest"
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                echo 'Pushing Docker image to Docker Hub...'
                script {
                    // Use Jenkins native docker plugin registry helper with absolute path injected
                    docker.withRegistry('https://index.docker.io/v1/', "${env.DOCKER_CREDENTIALS_ID}") {
                        bat "\"${env.DOCKER_BIN}\" push ${env.DOCKER_IMAGE}:${env.BUILD_NUMBER}"
                        bat "\"${env.DOCKER_BIN}\" push ${env.DOCKER_IMAGE}:latest"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                echo 'Deploying application to Kubernetes cluster...'
                script {
                    bat "powershell -Command \"(Get-Content deployment.yaml) -replace 'image:.*', 'image: ${env.DOCKER_IMAGE}:${env.BUILD_NUMBER}' | Set-Content deployment.yaml\""
                    
                    withKubeConfig(credentialsId: "${env.K8S_CREDENTIALS_ID}", serverUrl: '') {
                        bat "kubectl apply -f deployment.yaml --namespace=${env.NAMESPACE}"
                    }
                }
            }
        }

        stage('Verify and Automated Rollback') {
            steps {
                echo 'Verifying Kubernetes rollout status...'
                script {
                    try {
                        withKubeConfig(credentialsId: "${env.K8S_CREDENTIALS_ID}", serverUrl: '') {
                            bat "kubectl rollout status deployment/${env.DEPLOYMENT_NAME} --namespace=${env.NAMESPACE} --timeout=120s"
                        }
                        echo 'Deployment verified successfully!'
                    } catch (Exception e) {
                        echo 'Deployment failed! Initiating automatic rollback...'
                        withKubeConfig(credentialsId: "${env.K8S_CREDENTIALS_ID}", serverUrl: '') {
                            bat "kubectl rollout undo deployment/${env.DEPLOYMENT_NAME} --namespace=${env.NAMESPACE}"
                        }
                        currentBuild.result = 'FAILURE'
                        error('Deployment failed and was automatically rolled back.')
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning up local workspace docker images...'
            bat "\"${env.DOCKER_BIN}\" rmi ${env.DOCKER_IMAGE}:${env.BUILD_NUMBER} || exit 0"
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check error logs above.'
        }
    }
}
