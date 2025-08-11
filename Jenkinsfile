pipeline {
    agent any

    environment {
        GIT_REPO = 'git@github.com:your-org/your-repo.git'
        BRANCH = 'main'
        IMAGE_NAME = 'your-app'
        AWS_REGION = 'us-east-1'
        ECR_REPO = 'your-aws-account.dkr.ecr.us-east-1.amazonaws.com/your-app'
        S3_BUCKET = 'your-artifact-bucket'
        SONARQUBE_ENV = 'SonarQubeServer' // SonarQube config name in Jenkins
        ARTIFACT_PATH = 'target/*.jar'
        CLUSTER_NAME = 'your-eks-cluster'
        K8S_MANIFEST_DIR = 'k8s'
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        skipStagesAfterUnstable()
    }

    stages {

        stage('Checkout Source Code') {
            steps {
                sshagent(['github-ssh-key']) {
                    git url: "${GIT_REPO}", branch: "${BRANCH}"
                }
            }
        }

        stage('Build with Maven') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('SonarQube Analysis') {
            environment {
                SONAR_TOKEN = credentials('sonarqube-token')
            }
            steps {
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    sh """
                        mvn sonar:sonar \
                            -Dsonar.login=$SONAR_TOKEN
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials-id']]) {
                    script {
                        def imageTag = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
                        sh """
                            aws ecr get-login-password --region $AWS_REGION | \
                                docker login --username AWS --password-stdin $ECR_REPO
                            
                            docker build -t $ECR_REPO:$imageTag .
                            docker push $ECR_REPO:$imageTag

                            echo "IMAGE_TAG=$imageTag" > image-tag.env
                        """
                    }
                }
            }
        }

        stage('Scan Docker Image') {
            steps {
                script {
                    def imageTag = sh(script: "cat image-tag.env | cut -d'=' -f2", returnStdout: true).trim()
                    sh "trivy image --exit-code 1 --severity CRITICAL,HIGH $ECR_REPO:$imageTag"
                }
            }
        }

        stage('Upload Artifacts to S3') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials-id']]) {
                    sh """
                        aws s3 cp ${ARTIFACT_PATH} s3://${S3_BUCKET}/builds/${env.BUILD_NUMBER}/ --region $AWS_REGION
                    """
                }
            }
        }

        stage('Deploy to AWS EKS') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials-id']]) {
                    script {
                        def imageTag = sh(script: "cat image-tag.env | cut -d'=' -f2", returnStdout: true).trim()
                        sh """
                            aws eks update-kubeconfig --region $AWS_REGION --name $CLUSTER_NAME

                            # Replace image tag in manifest dynamically
                            sed -i "s|IMAGE_PLACEHOLDER|$ECR_REPO:$imageTag|g" $K8S_MANIFEST_DIR/deployment.yaml

                            kubectl apply -f $K8S_MANIFEST_DIR/
                        """
                    }
                }
            }
        }
    }

    post {
        failure {
            script {
                echo 'Pipeline failed. Sending notifications...'
            }

            // Email Notification
            mail to: 'devops-team@example.com',
                 subject: "Jenkins Build Failed - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "The Jenkins build has failed. Check the logs at ${env.BUILD_URL}.",
                 charset: 'UTF-8',
                 mimeType: 'text/plain'

            // Slack Notification
            withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_WEBHOOK')]) {
                sh """
                    curl -X POST -H 'Content-type: application/json' \
                    --data '{"text":"🚨 *Build Failed*: ${env.JOB_NAME} #${env.BUILD_NUMBER}\\n🔗 ${env.BUILD_URL}"}' \
                    $SLACK_WEBHOOK
                """
            }
        }

        success {
            echo 'Build and Deployment succeeded!'
        }
    }
}



# 2 # 

pipeline {
    agent any
    
    environment {
        // AWS Credentials and Region
        AWS_ACCESS_KEY_ID = credentials('aws-access-key-id') // AWS IAM Access Key
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key') // AWS IAM Secret Key
        AWS_REGION = 'us-east-1'  // Set to your AWS region
        ECR_REPO_NAME = 'my-app-repository'  // Name of your ECR repository
        ECR_URI = "aws_account_id.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}"
        EKS_CLUSTER_NAME = 'my-eks-cluster'  // Name of your EKS cluster
        KUBECONFIG_PATH = '/tmp/kubeconfig'  // Path for kubeconfig file
        
        // Docker Image Details
        IMAGE_TAG = "latest"
    }
    
    stages {
        
        // Stage 1: Checkout the code from repository
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }
        
        // Stage 2: Build Docker Image
        stage('Build Docker Image') {
            steps {
                script {
                    // Build Docker image using Dockerfile
                    sh 'docker build -t ${ECR_URI}:${IMAGE_TAG} .'
                }
            }
        }
        
        // Stage 3: Login to AWS ECR
        stage('Login to ECR') {
            steps {
                script {
                    // Authenticate Docker to AWS ECR using AWS CLI
                    sh """
                    aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_URI}
                    """
                }
            }
        }
        
        // Stage 4: Push Docker Image to ECR
        stage('Push Docker Image') {
            steps {
                script {
                    // Push the Docker image to AWS ECR
                    sh 'docker push ${ECR_URI}:${IMAGE_TAG}'
                }
            }
        }
        
        // Stage 5: Deploy to AWS EKS
        stage('Deploy to EKS') {
            steps {
                script {
                    // Set up kubectl to communicate with the EKS cluster
                    sh """
                    aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION} --kubeconfig ${KUBECONFIG_PATH}
                    """
                    
                    // Deploy the application to EKS
                    sh """
                    kubectl set image deployment/my-app-deployment my-app=${ECR_URI}:${IMAGE_TAG} --kubeconfig ${KUBECONFIG_PATH}
                    """
                }
            }
        }
        
        // Optional: Stage to Clean Up (Clear Docker cache)
        stage('Cleanup') {
            steps {
                script {
                    sh 'docker rmi ${ECR_URI}:${IMAGE_TAG}'
                }
            }
        }
    }
    
    post {
        always {
            // Clean up Docker images if needed
            cleanWs()
        }
    }
}

1. IAM Permissions for ECR (Elastic Container Registry)
To push and pull Docker images from ECR, Jenkins needs the following permissions:

Basic ECR Permissions:
ecr:GetAuthorizationToken: To authenticate Docker with the ECR registry.

ecr:BatchCheckLayerAvailability: To check the availability of image layers in ECR.

ecr:PutImage: To push Docker images to the ECR repository.

ecr:BatchGetImage: To pull Docker images from ECR.

ecr:DescribeRepositories: To describe ECR repositories.

ecr:ListImages: To list the images in a repository.

Example ECR Policy for Jenkins:

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:PutImage",
        "ecr:BatchGetImage",
        "ecr:DescribeRepositories",
        "ecr:ListImages"
      ],
      "Resource": "arn:aws:ecr:us-east-1:123456789012:repository/my-app-repository"
    }
  ]
}
This policy grants Jenkins the necessary permissions for pushing and pulling images from my-app-repository in ECR. Make sure to modify the ARN to match your repository.


2. IAM Permissions for EKS (Elastic Kubernetes Service)
To deploy and manage applications on EKS via kubectl, Jenkins requires permissions to interact with the Kubernetes API server in the EKS cluster.

Basic EKS Permissions:
eks:DescribeCluster: To describe the EKS cluster and fetch details like endpoint and certificate.

eks:ListClusters: To list all EKS clusters in your AWS account (optional but helpful for automation).

eks:UpdateKubeconfig: To configure kubectl to communicate with your EKS cluster.

sts:AssumeRole: If you're using IAM roles for Service Accounts (IRSA) or have separate roles for EKS access, Jenkins will need permission to assume these roles.

Example EKS Policy for Jenkins:
    {
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "eks:DescribeCluster",
        "eks:ListClusters",
        "eks:UpdateKubeconfig"
      ],
      "Resource": "arn:aws:eks:us-east-1:123456789012:cluster/my-eks-cluster"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sts:AssumeRole"
      ],
      "Resource": "arn:aws:iam::123456789012:role/my-eks-role"
    }
  ]
}
This policy allows Jenkins to describe the EKS cluster named my-eks-cluster, update the kubeconfig for kubectl, and assume a role (e.g., for accessing EKS nodes).
    
3. Assuming IAM Role (if applicable)
If you are using IAM roles to grant permissions, ensure that Jenkins has permission to assume the role. This is often the case when running Jenkins in an EC2 instance or ECS container that needs to assume an IAM role.
 {
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::123456789012:role/my-eks-role"
    }
  ]
}
This would allow Jenkins to assume my-eks-role to interact with EKS and other AWS services. You’d need to associate this policy with the EC2 instance profile or ECS task role running Jenkins.

4. IAM Role for Jenkins (for EC2-based Jenkins)
If Jenkins is running on an EC2 instance, you should attach an IAM role to the instance with the necessary permissions. The role will be assumed by Jenkins automatically, and it can access EKS and ECR using AWS SDK and AWS CLI.

Steps to Attach IAM Role to EC2 Instance:
Create an IAM Role with the above ECR and EKS policies.

Attach this role to the EC2 instance where Jenkins is running.
5. Security Best Practices
Use Principle of Least Privilege (PoLP):

Only grant the permissions necessary for Jenkins to perform its tasks.

Avoid wildcard (*) in resource definitions.

For example, specify the exact ECR repository or EKS cluster ARN instead of using "*".

Use IAM Roles for Service Accounts (IRSA):

If Jenkins is running within EKS or ECS, consider using IAM Roles for Service Accounts to grant AWS permissions to the Jenkins service account instead of managing credentials directly.

Use AWS Secrets Manager:

Store sensitive data (such as AWS access keys) in AWS Secrets Manager and securely access them in Jenkins.

Use Multi-Factor Authentication (MFA):

Enable MFA for sensitive operations (e.g., IAM, EC2, etc.) to ensure that accidental or malicious actions are mitigated.    
