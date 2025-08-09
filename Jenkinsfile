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
