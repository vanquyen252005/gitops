pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: jnlp
      image: jenkins-agent-docker:1.0
      imagePullPolicy: Never
      env:
        - name: DOCKER_HOST
          value: tcp://localhost:2375

    - name: dind
      image: docker:27-dind
      imagePullPolicy: IfNotPresent
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
        stage('Check Docker') {
            steps {
                sh '''
                    echo "Running on Jenkins agent: $(hostname)"
                    git --version
                    docker --version

                    DOCKER_READY=0

                    for i in $(seq 1 30); do
                        if docker info >/dev/null 2>&1; then
                            DOCKER_READY=1
                            break
                        fi

                        echo "Waiting for Docker daemon: $i/30"
                        sleep 2
                    done

                    if [ "$DOCKER_READY" -ne 1 ]; then
                        echo "Docker daemon did not become ready"
                        exit 1
                    fi

                    echo "Docker daemon is ready"
                    docker version
                '''
            }
        }

        stage('Build Image') {
            steps {
                sh '''
                    echo "Building $IMAGE:$TAG"
                    docker build -t "$IMAGE:$TAG" .
                '''
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login \
                          --username "$DOCKER_USER" \
                          --password-stdin

                        docker push "$IMAGE:$TAG"
                        docker logout
                    '''
                }
            }
        }

        stage('Update Helm') {
            steps {
                sh '''
                    sed -i \
                      "s|^[[:space:]]*tag:.*|  tag: \\"${TAG}\\"|" \
                      chart/values.yaml

                    echo "Updated Helm image:"
                    grep -A3 '^image:' chart/values.yaml

                    git config user.email "jenkins@lab.local"
                    git config user.name "jenkins"

                    git add chart/values.yaml

                    if ! git diff --cached --quiet; then
                        git commit -m "Update image tag to ${TAG}"
                    else
                        echo "No Helm changes to commit"
                    fi
                '''

                withCredentials([
                    usernamePassword(
                        credentialsId: 'github-token',
                        usernameVariable: 'GITHUB_USER',
                        passwordVariable: 'GITHUB_TOKEN'
                    )
                ]) {
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

    post {
        success {
            echo "Pipeline completed successfully"
            echo "Published image: ${IMAGE}:${TAG}"
        }

        failure {
            echo "Pipeline failed"
        }
    }
}
