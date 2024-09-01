pipeline {
    agent {
        kubernetes {
            label 'jenkins-agent'
            defaultContainer 'jnlp'
            yaml '''
            apiVersion: v1
            kind: Pod
            spec:
              containers:
              - name: jnlp
                image: ronn4/repo1:jenkins-agent
                alwaysPullImage: true
                imagePullSecrets:
                - name: regcred
                args: ['\${computer.jnlpmac}', '\${computer.name}']
            '''
        }
    }

    options {
        buildDiscarder(logRotator(daysToKeepStr: '30'))
        disableConcurrentBuilds()
        timestamps()
    }

    environment {
        APP_IMAGE_NAME ='app-image'
        WEB_IMAGE_NAME = 'web-image'
        DOCKER_COMPOSE_FILE = 'compose.yaml'
        DOCKER_REPO = 'ronn4/repo1'
        DOCKERHUB_CREDENTIALS = 'dockerhub'

    }
            
        }
        stage('Login, Tag, and Push Images') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'NEXUS_CREDENTIALS_ID', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    script {
                        echo 'Starting Login, Tag, and Push Images Stage...'
                        try {
                            sh 'git rev-parse --short HEAD > gitCommit.txt'
                            def GITCOMMIT = readFile('gitCommit.txt').trim()
                            def GIT_TAG = "${GITCOMMIT}"
                            def IMAGE_TAG = "v1.0.0-${BUILD_NUMBER}-${GIT_TAG}"
                            sh """
                                cd polybot
                                docker login -u ${USER} -p ${PASS} ${NEXUS_PROTOCOL}://${NEXUS_URL}/repository/${NEXUS_REPO}
                                docker tag ${APP_IMAGE_NAME}:latest ${NEXUS_URL}/${APP_IMAGE_NAME}:${IMAGE_TAG}
                                docker tag ${WEB_IMAGE_NAME}:latest ${NEXUS_URL}/${WEB_IMAGE_NAME}:${IMAGE_TAG}
                                docker push ${NEXUS_URL}/${APP_IMAGE_NAME}:${IMAGE_TAG}
                                docker push ${NEXUS_URL}/${WEB_IMAGE_NAME}:${IMAGE_TAG}
                            """
                            echo 'Login, Tag, and Push Images Stage Completed'
                        } catch (Exception e) {
                            echo "Error in Login, Tag, and Push Images Stage: ${e}"
                            throw e
                        }
                    }
                }
            }
        }
        stage('Deploy with Helm') {
            steps {
                script {
                    // Ensure Helm is installed in the pod
                    sh 'helm version'

                    // Set up Kubernetes context for the desired namespace
                    sh 'kubectl config set-context --current --namespace=demoapp'

                    // Deploy the application using your Helm chart
                    sh """
                    helm upgrade --install deploy-demo-0.1.0 ./my-python-app-chart \
                    --namespace demoapp \
                    --set image.repository=${DOCKER_REPO} \
                    --set image.tag=${GIT_COMMIT} \
                    --set replicas=3
                    """
                }
            }
        }
    }
}
