pipeline {
    agent any

    environment {
        IMAGE = "docker.io/vanquyen2522005/gitops-demo"
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
        stage('Build Image') {
	when {
    expression { env.SKIP_CI != 'true' }
	}
            steps {
                sh 'docker build -t $IMAGE:$TAG .'
            }
        }

        stage('Push Image') {
	when {
    expression { env.SKIP_CI != 'true' }
	}
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
when {
    expression { env.SKIP_CI != 'true' }
}
    steps {
        withCredentials([usernamePassword(
            credentialsId: 'github',
            usernameVariable: 'GIT_USER',
            passwordVariable: 'GIT_TOKEN'
        )]) {
            sh '''
            sed -i "s/tag:.*/tag: \\"$TAG\\"/" chart/values.yaml

            git config user.email "jenkins@lab.local"
            git config user.name "jenkins"

            git add chart/values.yaml
            git commit -m "Update image tag $TAG [skip ci]" || echo "No changes to commit"

            git push https://${GIT_USER}:${GIT_TOKEN}@github.com/vanquyen252005/gitops.git HEAD:main
            '''
        }
    }
}
    }
}
