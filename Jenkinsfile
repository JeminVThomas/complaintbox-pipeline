
pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
    }

    environment {
        AWS_REGION = 'ap-south-1'
        AWS_ACCOUNT_ID = '994878981749'
        ECR_REPOSITORY = 'complaintbox'
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_TAG = "${BUILD_NUMBER}"
        IMAGE_URI = "${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}"
        SONAR_SCANNER = tool 'SonarScanner'
        SKIP_BUILD = 'false'
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'github',
                    url: 'https://github.com/JeminVThomas/complaintbox-pipeline.git'

                script {
                    def message = sh(
                        script: 'git log -1 --pretty=%s',
                        returnStdout: true
                    ).trim()

                    if (message.contains('[skip ci]')) {
                        env.SKIP_BUILD = 'true'
                        echo 'Skipping automated manifest commit.'
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            when {
                expression { env.SKIP_BUILD != 'true' }
            }
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''
                        "$SONAR_SCANNER/bin/sonar-scanner"
                    '''
                }
            }
        }

        stage('Quality Gate') {
            when {
                expression { env.SKIP_BUILD != 'true' }
            }
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build') {
            when {
                expression { env.SKIP_BUILD != 'true' }
            }
            steps {
                sh '''
                    docker build \
                        -t "$ECR_REPOSITORY:$IMAGE_TAG" .

                    docker tag \
                        "$ECR_REPOSITORY:$IMAGE_TAG" \
                        "$IMAGE_URI"
                '''
            }
        }

        stage('ECR Login') {
            when {
                expression { env.SKIP_BUILD != 'true' }
            }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'aws-ecr-credentials',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    sh '''
                        aws ecr get-login-password \
                            --region "$AWS_REGION" | \
                        docker login \
                            --username AWS \
                            --password-stdin "$ECR_REGISTRY"
                    '''
                }
            }
        }

        stage('Push Image to ECR') {
            when {
                expression { env.SKIP_BUILD != 'true' }
            }
            steps {
                sh '''
                    docker push "$IMAGE_URI"
                '''
            }
        }

        stage('Update Kubernetes Manifest') {
            when {
                expression { env.SKIP_BUILD != 'true' }
            }
            steps {
                withCredentials([
                    gitUsernamePassword(
                        credentialsId: 'github',
                        gitToolName: 'Default'
                    )
                ]) {
                    sh '''
                        git config user.name "Jenkins"
                        git config user.email "jenkins@complaintbox.local"

                        test -f k8s/deployment.yaml

                        sed -i "s|image:.*${ECR_REGISTRY}/${ECR_REPOSITORY}:.*|image: ${IMAGE_URI}|" \
                            k8s/deployment.yaml

                        grep -F "image: ${IMAGE_URI}" \
                            k8s/deployment.yaml

                        git add k8s/deployment.yaml

                        if git diff --cached --quiet; then
                            echo "No manifest changes required."
                        else
                            git commit -m "Update ComplaintBox image to ${IMAGE_TAG} [skip ci]"
                            git push origin HEAD:main
                        fi
                    '''
                }
            }
        }
    }

    post {
        success {
            script {
                if (env.SKIP_BUILD == 'true') {
                    echo 'Automated manifest commit skipped.'
                } else {
                    echo "Image pushed successfully: ${IMAGE_URI}"
                    echo 'Kubernetes manifest updated.'
                    echo 'Argo CD will deploy the image when configured.'
                }
            }
        }

        failure {
            echo 'Pipeline failed. Check the console output.'
        }

        always {
            sh '''
                docker logout "$ECR_REGISTRY" || true
            '''
        }
    }
}

