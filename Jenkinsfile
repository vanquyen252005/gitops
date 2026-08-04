pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: docker
      image: docker:27-cli
      command:
        - cat
      tty: true
      env:
        - name: DOCKER_HOST
          value: tcp://localhost:2375

    - name: dind
      image: docker:27-dind
      securityContext:
        privileged: true
      env:
        - name: DOCKER_TLS_CERTDIR
          value: ""
      args:
        - --host=tcp://0.0.0.0:2375
        - --tls=false
        - --storage-driver=vfs
'''
        }
    }

    environment {
        IMAGE = "docker.io/vanquyen2522005/gitops-demo"
        TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('Build Image') {
            steps {
                container('docker') {
                    sh '''
                        DOCKER_READY=0

                        for i in $(seq 1 30); do
                            if docker info >/dev/null 2>&1; then
                                DOCKER_READY=1
                                break
                            fi

                            echo "Waiting for Docker daemon..."
                            sleep 2
                        done

                        if [ "$DOCKER_READY" -ne 1 ]; then
                            echo "Docker daemon is not ready"
                            exit 1
                        fi

                        docker version
                        docker build -t "$IMAGE:$TAG" .
                    '''
                }
            }
        }

        stage('Push Image') {
            steps {
                container('docker') {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh '''
                            echo "$DOCKER_PASS" | docker login \
                              --username "$DOCKER_USER" \
                              --password-stdin

                            docker push "$IMAGE:$TAG"
                        '''
                    }
                }
            }
        }

        stage('Update Helm') {
            steps {
                sh '''
                    sed -i \
                      "s|^[[:space:]]*tag:.*|  tag: \\"${TAG}\\"|" \
                      chart/values.yaml

                    git config user.email "jenkins@lab.local"
                    git config user.name "jenkins"

                    git add chart/values.yaml

                    if ! git diff --cached --quiet; then
                        git commit -m "Update image ${TAG}"
                    else
                        echo "No Helm value changes"
                    fi
                '''

                withCredentials([usernamePassword(
                    credentialsId: 'github-token',
                    usernameVariable: 'GITHUB_USER',
                    passwordVariable: 'GITHUB_TOKEN'
                )]) {
                    sh '''
                        set +x
                        git push \
                          "https://${GITHUB_USER}:${GITHUB_TOKEN}@github.com/vanquyen252005/gitops.git" \
                          HEAD:main
                    '''
                }
            }
        }
    }
}
