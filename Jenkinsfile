pipeline {
    agent any

    environment {
        IMAGE = "docker.io/vanquyen2522005/gitops-demo"
        TAG = "${BUILD_NUMBER}"
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
                    passwordVariable: 'PASS'
                )]) {
                    sh '''
                    echo $PASS | docker login -u $USER --password-stdin
                    docker push $IMAGE:$TAG
                    '''
                }
            }
        }

        stage('Update Helm Values') {
            steps {
                sh '''
                sed -i "s/tag:.*/tag: \\"$TAG\\"/" chart/values.yaml
                git config user.email "jenkins@lab.local"
                git config user.name "jenkins"
                git add chart/values.yaml
                git commit -m "Update image tag $TAG" || true
                git push
                '''
            }
        }
    }
}
