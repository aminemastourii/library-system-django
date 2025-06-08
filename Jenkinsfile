pipeline {
    agent none  // No global agent, each stage will define its own
    environment {
        DOCKER_CONFIG = '/tmp/.docker'  // Set to a directory with write access
        repoUri = "615299749113.dkr.ecr.us-east-1.amazonaws.com/aminemastouri"
        repoRegistryUrl = "https://615299749113.dkr.ecr.us-east-1.amazonaws.com"
        registryCreds = 'ecr:us-east-1:awscreds'
        cluster = "aminemastouri"
        service = "librarysys-task-service-caitjd4g"
        region = 'us-east-1'
    }

    stages {
        stage('Docker Test') {
            
            agent {
                docker {
                    image 'docker:latest'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'  // Mount Docker socket
                }
            }
            steps {
                script {

                    sh 'docker ps'
                    cleanWs()
                }
            }
        }
        stage('Run Unit Tests') {
                agent {
                    docker {
                        image 'python:3.10'
                    }
                }
                steps {
                    script {
                        echo "Installing dependencies and running tests..."
                        sh '''
                            python3 -m venv venv
                            . venv/bin/activate
                            pip install --upgrade pip
                            pip install -r requirements.txt
                            python manage.py test

                            deactivate
                            rm -rf venv
                        '''
                    }
                }
            }
        
        stage('Code Analysis SonarQube') {
            environment {
                scannerHome = tool 'sonnar-scanner' // or your configured scanner name
            }
            steps {
                withSonarQubeEnv('sonar-server') {
                sh '''${scannerHome}/bin/sonar-scanner \
                    -Dsonar.projectKey=librarysys \
                    -Dsonar.projectName=librarysys \
                    -Dsonar.projectVersion=1.0 \
                    -Dsonar.sources=. \
                    -Dsonar.language=python \
                    -Dsonar.sourceEncoding=UTF-8 \
                    -Dsonar.python.version=3.10 \
                    -Dsonar.organization=am-org'''
                }
            }
            }

        stage('Build Docker Image') {
            agent {

                docker {
                    image 'docker:latest'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'  // Mount Docker socket
                }
            }
            steps {
                script {
                    echo 'Building Docker Image from Dockerfile...'
                    sh 'mkdir -p /tmp/.docker'  // Ensure the directory exists
                    dockerImage = docker.build(repoUri + ":$BUILD_NUMBER")
                }
            }
        }

        stage('Push Docker Image to ECR') {
            agent {
                docker {
                    image 'docker:latest'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'  // Mount Docker socket
                }
            }
            steps {
                script {
                    echo "Pushing Docker Image to ECR..."
                    docker.withRegistry(repoRegistryUrl, registryCreds) {
                        dockerImage.push("$BUILD_NUMBER")
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Deploy to ECS') {
            agent {
                docker {
                    image 'amazon/aws-cli:latest'  // Use a pre-built AWS CLI Docker image for ECS deployment
                    args '-v /var/run/docker.sock:/var/run/docker.sock --entrypoint=""'  // Optional if needed by AWS CLI
                }
            }
            steps {
                script {
                    echo "Deploying Image to ECS..."
                    withAWS(credentials: 'awscreds', region: "${region}") {
                        sh 'aws ecs update-service --cluster ${cluster} --service ${service} --force-new-deployment'
                    }
                }
            }
        }
    }
}
