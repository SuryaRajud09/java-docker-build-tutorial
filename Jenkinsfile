pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1' 
        ECR_REPO_NAME = 'java-app' 
        IMAGE_TAG = "v${env.BUILD_NUMBER}"
        EKS_CLUSTER_NAME = 'java-cluster' 
        KUBERNETES_NAMESPACE = 'default' 
    }

    stages {
        stage('Clone Repository') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def imageName = "${ECR_REPO_NAME}:${IMAGE_TAG}"
                    sh "docker build -t ${imageName} ."
                }
            }
        }

        stage('Login to AWS ECR') {
            steps {
                script {
                    sh """
                        aws ecr-public get-login-password --region us-east-1 | docker login --username AWS --password-stdin public.ecr.aws/i2z0j0p5
                    """
                }
            }
        }

        stage('Tag and Push to ECR') {
            steps {
                script {
                    def ecrUri = "393354520949.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}"
                    def fullImageUri = "${ecrUri}:${IMAGE_TAG}"

                    sh """
                        docker tag java-app:latest public.ecr.aws/i2z0j0p5/java-app:${IMAGE_TAG}
                        docker push public.ecr.aws/i2z0j0p5/java-app:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Configure kubectl for EKS') {
            steps {
                script {
                    sh """
                        aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER_NAME}
                    """
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                script {
                    def ecrUri = "393354520949.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}"
                    def fullImageUri = "${ecrUri}:${IMAGE_TAG}"

                    // Update the image in the Kubernetes manifest
                    sh """
                        sed -i 's|image: .*|image: ${fullImageUri}|' deployment.yaml
                    """

                    // Apply the manifest to deploy to EKS
                    sh """
                        kubectl apply -f deployment.yaml --namespace ${KUBERNETES_NAMESPACE}
                    """
                }
            }
        }
    }

    post {
        always {
            script {
                sh 'docker system prune -f' // Clean up unused Docker objects
            }
        }
        success {
            echo "Application deployed to EKS successfully."
        }
        failure {
            echo "Build, push, or deployment failed."
        }
    }
}
