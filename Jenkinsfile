pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
    }

    environment {
        AWS_REGION = 'ap-south-1'
        ECR_REPOSITORY = 'complaintbox'
        IMAGE_TAG = "${BUILD_NUMBER}"
        SCANNER_HOME = tool 'SonarScanner'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build -t $ECR_REPOSITORY:$IMAGE_TAG .
                '''
            }
        }

        stage('ECR Login') {
    steps {
        withCredentials([
            usernamePassword(
                credentialsId: 'aws-ecr-credentials',
                usernameVariable: 'AWS_ACCESS_KEY_ID',
                passwordVariable: 'AWS_SECRET_ACCESS_KEY'
            )
        ]) {
            script {
                env.AWS_ACCOUNT_ID = sh(
                    script: 'aws sts get-caller-identity --query Account --output text',
                    returnStdout: true
                ).trim()

                env.ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
            }

            sh '''
                aws ecr get-login-password \
                    --region $AWS_REGION | \
                docker login \
                    --username AWS \
                    --password-stdin $ECR_REGISTRY
            '''
        }
    }
}

        stage('Push Image to ECR') {
            steps {
                sh '''
                    docker tag $ECR_REPOSITORY:$IMAGE_TAG \
                        $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG

                    docker push \
                        $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
                '''
            }
        }
    }

    post {
        success {
            echo "ComplaintBox image pushed successfully."
            echo "Image tag: ${IMAGE_TAG}"
        }

        failure {
            echo 'Pipeline failed.'
        }
    }
}
