pipeline {
    agent any

    environment {
        AWS_REGION = "ap-southeast-2"
        AWS_ACCOUNT_ID = "475309741409"

        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_NAME = "gitops-demo"
        IMAGE = "${ECR_REGISTRY}/${IMAGE_NAME}"

        TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Check Skip CI') {
            steps {
                script {
                    def msg = sh(
                        script: 'git log -1 --pretty=%B',
                        returnStdout: true
                    ).trim()

                    if (msg.contains('[skip ci]')) {
                        env.SKIP_CI = 'true'
                        echo "Skip CI because this commit was created by Jenkins."
                    } else {
                        env.SKIP_CI = 'false'
                    }
                }
            }
        }


        stage('Checkout') {
            when {
                expression {
                    env.SKIP_CI != 'true'
                }
            }

            steps {
                checkout scm
            }
        }


        stage('Build & Test') {
            when {
                expression {
                    env.SKIP_CI != 'true'
                }
            }

            steps {
                sh '''
                    echo "Running build and tests..."

                    # Nếu project Java/Maven:
                    # ./mvnw test

                    # Nếu project Node:
                    # npm install
                    # npm test

                    # Nếu hiện tại chưa có test:
                    echo "No test command configured yet."
                '''
            }
        }


        stage('Build Docker Image') {
            when {
                expression {
                    env.SKIP_CI != 'true'
                }
            }

            steps {
                sh '''
                    echo "Building Docker image..."

                    docker build \
                        -t ${IMAGE}:${TAG} \
                        -t ${IMAGE}:latest \
                        .
                '''
            }
        }


        stage('Login to Amazon ECR') {
            when {
                expression {
                    env.SKIP_CI != 'true'
                }
            }

            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'aws-ecr']
                ]) {

                    sh '''
                        echo "Logging into Amazon ECR..."

                        aws ecr get-login-password \
                            --region ${AWS_REGION} \
                        | docker login \
                            --username AWS \
                            --password-stdin ${ECR_REGISTRY}
                    '''
                }
            }
        }


        stage('Push Image to ECR') {
            when {
                expression {
                    env.SKIP_CI != 'true'
                }
            }

            steps {
                sh '''
                    echo "Pushing image to Amazon ECR..."

                    docker push ${IMAGE}:${TAG}
                    docker push ${IMAGE}:latest
                '''
            }
        }


        stage('Update Helm Values') {
            when {
                expression {
                    env.SKIP_CI != 'true'
                }
            }

            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'github',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_TOKEN'
                    )
                ]) {

                    sh '''
                        echo "Updating GitOps repository..."

                        sed -i "s/tag:.*/tag: \\"${TAG}\\"/" chart/values.yaml

                        git config user.email "jenkins@lab.local"
                        git config user.name "jenkins"

                        git add chart/values.yaml

                        git commit \
                            -m "Update image tag ${TAG} [skip ci]" \
                            || echo "No changes to commit"

                        git push \
                            https://${GIT_USER}:${GIT_TOKEN}@github.com/vanquyen252005/gitops.git \
                            HEAD:main
                    '''
                }
            }
        }
    }


    post {

        success {
            echo "CI pipeline completed successfully."
            echo "Image pushed to:"
            echo "${IMAGE}:${TAG}"
        }

        failure {
            echo "CI pipeline failed."
        }

        always {
            sh '''
                docker logout ${ECR_REGISTRY} || true
            '''
        }
    }
}
