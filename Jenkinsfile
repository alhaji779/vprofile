pipeline {
    agent any
    
    tools {
        maven 'MAVEN3.9'
        jdk 'JDK17'
    }

    environment {
        SONARQUBE_ENV = 'sonarscanner'
        SONAR_PROJECT_KEY = 'vprofile-project'
        SONAR_SOURCES = 'src/'
        ARTIFACT_ID = "vprofile-v2"
        DOCKER_IMAGE = "alhaji779/vpro:latest"
        DEPLOY_SERVER = '209.38.208.51'
        DEPLOY_USER = 'devops'
        SSH_CREDENTIALS_ID = '209-devops-login' // <-- new SSH key credentials ID
        CONTAINER_NAME = 'vprofile-app'
        DEPLOY_PORT = '8086'
    }

    triggers {
        githubPush() // Trigger build when GitHub webhook fires
    }

    stages {
        stage('Clean Workspace') {
            steps {
                deleteDir()
            }
        }

        stage('Access GitHub Code') {
            steps {
                git branch: 'devops', url: 'https://github.com/alhaji779/vprofile.git'
            }
        }

        stage('Run Unit Tests & Generate Coverage') {
            steps {
                sh 'mvn clean verify'
            }
        }
        
        stage('Checkstyle Analysis') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh 'mvn clean verify sonar:sonar -Dsonar.projectKey=vprofile'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Artifact') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Archive Artifact') {
            steps {
                archiveArtifacts artifacts: '**/*.war', fingerprint: true
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    def safeImageName = "${DOCKER_IMAGE}".toLowerCase()
                    dockerImage = docker.build(safeImageName,"-f Docker-files/app/multistage/Dockerfile .")
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-login') {
                        dockerImage.push()
                        dockerImage.push("latest")
                    }
                }
            }
        }
       
        stage('Deploy to Remote Server via SSH Key') {
            steps {
                script {
                    sshagent([SSH_CREDENTIALS_ID]) {
                        
                        // Pull Docker image
                        sh """
                            ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${DEPLOY_SERVER} \
                                'docker pull ${DOCKER_IMAGE}'
                        """
                        
                        // Stop & remove existing container
                        sh """
                            ssh ${DEPLOY_USER}@${DEPLOY_SERVER} \
                                'docker stop ${CONTAINER_NAME} >/dev/null 2>&1 ; \
                                 docker rm ${CONTAINER_NAME} >/dev/null 2>&1 '
                        """
                        
                        // Run new container
                        sh """
                            ssh ${DEPLOY_USER}@${DEPLOY_SERVER} \
                                'docker run -d \
                                    --name ${CONTAINER_NAME} \
                                    -p ${DEPLOY_PORT}:8080 \
                                    --restart=always \
                                    ${DOCKER_IMAGE}'
                        """
                    }
                }
            }
        }
    }
}
