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
      imagePullPolicy: IfNotPresent
      command:
        - sh
        - -c
      args:
        - sleep 7d
      tty: true
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
        /*
         * Không cần stage Checkout.
         * Jenkins tự checkout repository vì job đang sử dụng
         * "Pipeline script from SCM".
         */

        stage('Build Image') {
            steps {
                container('docker') {
                    sh '''
                        echo "Checking Docker daemon..."

                        DOCKER_READY=0

                        for i in $(seq 1 30); do
                            if docker info >/dev/null 2>&1; then
                                DOCKER_READY=1
                                break
                            fi

                            echo "Waiting for Docker daemon: attempt $i/30"
                            sleep 2
                        done

                        if [ "$DOCKER_READY" -ne 1 ]; then
                            echo "ERROR: Docker daemon is not ready"
                            exit 1
                        fi

                        echo "Docker daemon is ready"
                        docker version

                        echo "Building image: $IMAGE:$TAG"
                        docker build -t "$IMAGE:$TAG" .
                    '''
                }
            }
        }

        stage('Push Image') {
            steps {
                container('docker') {
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

                            echo "Pushing image: $IMAGE:$TAG"
                            docker push "$IMAGE:$TAG"

                            docker logout
                        '''
                    }
                }
            }
        }

        stage('Update Helm') {
            steps {
                sh '''
                    echo "Updating chart/values.yaml to tag: ${TAG}"

                    sed -i \
                      "s|^[[:space:]]*tag:.*|  tag: \\"${TAG}\\"|" \
                      chart/values.yaml

                    echo "Updated values.yaml:"
                    grep -A3 '^image:' chart/values.yaml

                    git config user.email "jenkins@lab.local"
                    git config user.name "jenkins"

                    git add chart/values.yaml

                    if git diff --cached --quiet; then
                        echo "No Helm configuration changes"
                    else
                        git commit -m "Update image tag to ${TAG}"
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

                        echo "Pushing Helm configuration to GitHub"

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
            echo "Docker image: ${IMAGE}:${TAG}"
        }

        failure {
            echo "Pipeline failed. Check the failed stage in Console Output."
        }
    }
}
