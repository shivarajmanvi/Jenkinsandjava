pipeline {
    agent any 
    
    environment {
        GIT_REPO = "https://github.com/shivarajmanvi/Jenkinsandjava.git"
        AWS_REGION = 'ap-south-1'
        AWS_ACCESS_KEY_ID = "genearate"
        AWS_SECRET_ACCESS_KEY = "generate"
        ECR_REPO_NAME = 'jenkinsecr1'
        ECR_PRIVATE_REPO_URI = '009988249876.dkr.ecr.ap-south-1.amazonaws.com/jenkinsecr1'
        IMAGE_TAG = 'latest'
        AWS_ACCOUNT_ID = '009988249876'
        IMAGE_URI = "${ECR_PRIVATE_REPO_URI}:${IMAGE_TAG}"
    }
    stages {
        stage ('Install AWS CLI') {
            steps {
                script {
                    sh '''
                    set -e
                    echo "Installing AWS CLI..."
                    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
                    rm -rf aws
                    unzip -oq awscliv2.zip
                    ./aws/install -i ~/aws-cli -b ~/bin --update
                    export PATH=~/bin:$PATH
                    aws --version
                '''
                }
            }
        }
        stage('configure AWS Credentials') {
            steps {
                script {
                    sh '''
                    echo "setting up aws credentials for jenkins..."
                    mkdir -p /var/lib/jenkins/.aws
                    echo "[default]" > /var/lib/jenkins/.aws/credentials
                    aws_access_key_id=${AWS_ACCESS_KEY_ID}
                    aws_secret_access_key=${AWS_SECRET_ACCESS_KEY}
                    chown -R jenkins:jenkins /var/lib/jenkins/.aws
                '''
                }
            }
        }
        stage('Clone Repository') {
            steps {
                git url: "${GIT_REPO}", branch: 'main'
            }
        }
        stage('Build') {
            steps {
                script {
                    sh '''
                        echo "Building Java application..."
                        mvn clean -B -Denforcer.skip=true package
                    '''
                }
            }
        }

        stage('Login to AWS ECR') {
            steps {
                script {
                    sh '''
                        echo "Logging into AWS ECR..."
                        export PATH=~/bin:$PATH
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_PRIVATE_REPO_URI}
                    '''
                }   
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh '''
                        echo "Building Docker image..."
                        docker build -t ${IMAGE_URI} .
                    '''
                }
            }
        }

        stage('Push Docker Image to ECR') {
            steps {
                script {
                    sh '''
                        echo "Pushing Docker image to ECR..."
                        docker push ${IMAGE_URI}
                    '''
                }
            }
        }
    }
}
