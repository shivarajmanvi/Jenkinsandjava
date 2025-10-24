pipeline {
    agent any

    environment {
        // 🔹 GitHub repository
        GIT_REPO = 'https://github.com/shivarajmanvi/Jenkinsandjava.git'
        BRANCH_NAME = 'main' // change to 'master' if your repo uses that

        // 🔹 AWS & ECR configuration
        AWS_REGION = 'ap-south-1'
        ECR_PUBLIC_REPO_URI = 'public.ecr.aws/s9l8a0f0/jenkinsecr'
        IMAGE_TAG = 'latest'
        IMAGE_URI = "${ECR_PUBLIC_REPO_URI}:${IMAGE_TAG}"

        // 🔹 EKS configuration
        EKS_CLUSTER_NAME = 'my-eks-cluster'
        K8S_DEPLOYMENT_NAME = 'jenkins-java-app'
        K8S_NAMESPACE = 'default'
    }

    stages {

        stage('Install AWS CLI and kubectl') {
            steps {
                sh '''
                    set -e
                    echo "Installing AWS CLI and kubectl..."
                    sudo apt update -y
                    sudo apt install -y unzip curl apt-transport-https gnupg

                    # Install AWS CLI v2
                    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
                    rm -rf aws
                    unzip -q awscliv2.zip
                    sudo ./aws/install --update
                    aws --version

                    # Install kubectl
                    curl -o kubectl https://s3.us-west-2.amazonaws.com/amazon-eks/1.28.4/2023-11-14/bin/linux/amd64/kubectl
                    chmod +x ./kubectl
                    sudo mv ./kubectl /usr/local/bin/
                    kubectl version --client
                '''
            }
        }

        stage('Configure AWS Credentials') {
            steps {
                // 🔒 Uses credentials stored securely in Jenkins
                withCredentials([aws(credentialsId: 'aws-jenkins-creds', region: "${AWS_REGION}")]) {
                    sh '''
                        echo "AWS credentials configured from Jenkins Credentials Store..."
                        aws sts get-caller-identity
                    '''
                }
            }
        }

        stage('Clone Repository') {
            steps {
                echo "Cloning source code from GitHub..."
                git url: "${GIT_REPO}", branch: "${BRANCH_NAME}"
            }
        }

        stage('Build Java Application') {
            steps {
                sh '''
                    echo "Building and packaging Java application..."
                    mvn clean package -DskipTests
                '''
            }
        }

        stage('Archive Build Artifact') {
            steps {
                echo "Archiving JAR file for reference..."
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        stage('Login to AWS ECR') {
            steps {
                sh '''
                    echo "Logging in to AWS ECR Public..."
                    aws ecr-public get-login-password --region us-east-1 | \
                    docker login --username AWS --password-stdin public.ecr.aws
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "Building Docker image..."
                    docker build -t ${IMAGE_URI} .
                '''
            }
        }

        stage('Push Docker Image to ECR') {
            steps {
                sh '''
                    echo "Pushing Docker image to ECR Public..."
                    docker push ${IMAGE_URI}
                '''
            }
        }

        stage('Deploy to Amazon EKS') {
            steps {
                withCredentials([aws(credentialsId: 'aws-jenkins-creds', region: "${AWS_REGION}")]) {
                    sh '''
                        echo "Configuring kubectl to connect to EKS cluster..."
                        aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER_NAME}

                        echo "Updating Kubernetes deployment with new Docker image..."
                        kubectl set image deployment/${K8S_DEPLOYMENT_NAME} \
                            ${K8S_DEPLOYMENT_NAME}=${IMAGE_URI} \
                            -n ${K8S_NAMESPACE} || \
                        echo "Deployment not found. Creating new deployment..."

                        # If deployment doesn’t exist, create one
                        kubectl get deployment ${K8S_DEPLOYMENT_NAME} -n ${K8S_NAMESPACE} || kubectl create deployment ${K8S_DEPLOYMENT_NAME} --image=${IMAGE_URI} -n ${K8S_NAMESPACE}

                        echo "Exposing service on port 8080..."
                        kubectl expose deployment ${K8S_DEPLOYMENT_NAME} --type=LoadBalancer --port=8080 -n ${K8S_NAMESPACE} || true

                        echo "✅ Application deployed successfully to EKS!"
                        kubectl get svc -n ${K8S_NAMESPACE}
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ Build, push, and deployment completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed. Please check the logs."
        }
    }
}

