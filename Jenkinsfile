pipeline {
    agent any
    tools {
        jdk 'jdk'
        nodejs 'node17'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/TranPio/devsecops-prime-video.git'
            }
        }
        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=devsecops-prime-video \
                    -Dsonar.projectKey=devsecops-prime-video '''
                }
            }
        }
        stage("quality gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build & Push") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        // Đăng nhập vào Docker Hub sử dụng token (PAT)
                        sh "echo \$DOCKER_PASSWORD | docker login -u \$DOCKER_USERNAME --password-stdin"
                        
                        // Xây dựng và đẩy Docker image
                        sh "docker build -t devsecops-prime-video ."
                        sh "docker tag devsecops-prime-video piotran/devsecops-prime-video:latest"
                        sh "docker push piotran/devsecops-prime-video:latest"
                    }
                }
            }
        }
        stage('Docker Scout Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh 'docker-scout quickview piotran/devsecops-prime-video:latest'
                        sh 'docker-scout cves piotran/devsecops-prime-video:latest'
                        sh 'docker-scout recommendations piotran/devsecops-prime-video:latest'
                    }
                }
            }
        }
        stage("TRIVY-docker-images") {
            steps {
                sh "trivy image piotran/devsecops-prime-video:latest > trivyimage.txt"
            }
        }
        stage('App Deploy to Docker container') {
            steps {
                sh 'docker run -d --name devsecops-prime-video -p 3000:3000 piotran/devsecops-prime-video:latest'
            }
        }
    }
    post {
        always {
            script {
                def buildStatus = currentBuild.currentResult
                def buildUser = currentBuild.getBuildCauses('hudson.model.Cause$UserIdCause')[0]?.userId ?: 'Github User'
                
                emailext (
                    subject: "Pipeline ${buildStatus}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: """
                        <p>This is a Jenkins DevSecOps-Prime-Video CICD pipeline status.</p>
                        <p>Project: ${env.JOB_NAME}</p>
                        <p>Build Number: ${env.BUILD_NUMBER}</p>
                        <p>Build Status: ${buildStatus}</p>
                        <p>Started by: ${buildUser}</p>
                        <p>Build URL: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    """,
                    to: 'tranhoaiphu04@gmail.com',
                    from: 'tranhoaiphu04@gmail.com',
                    replyTo: 'tranhoaiphu04@gmail.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
                )
            }
        }
    }
}
