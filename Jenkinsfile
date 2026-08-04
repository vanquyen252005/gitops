pipeline {
    agent any
    environment {
        IMAGE = "docker.io/vanquyen2522005/gitops-demo"
        TAG = "${BUILD_NUMBER}"
    }
    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/vanquyen252005/gitops.git'
            }
        }
        stage('Build Image') {
            steps {
                sh 'docker build -t $IMAGE:$TAG .'
            }
        }
        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS')]) {
                    sh '''
                    echo $PASS | docker login -u $USER --password-stdin
                    docker push $IMAGE:$TAG
                    '''
                }
            }
        }
        stage('Update Helm') {
            steps {
                sh """
                sed -i 's/tag:.*/tag: ${TAG}/' chart/values.yaml
                git config user.email "jenkins@lab.local"
                git config user.name "jenkins"
                git add .
                git commit -m "Update image ${TAG}"
                git push
                """
            }
        }
    }
}
