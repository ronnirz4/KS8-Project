pipeline {
    agent {
        kubernetes {
            yaml '''
            apiVersion: v1
            kind: Pod
            spec:
              containers:
              - name: jenkins-slave
                image: ronn4/repo1:jenkins-agent
                command:
                - cat
                tty: true
            '''
        }
    }

    options {
        buildDiscarder(logRotator(daysToKeepStr: '30'))
        disableConcurrentBuilds()
        timestamps()
    }

    environment {
        APP_IMAGE_NAME = 'app-image-latest'
        WEB_IMAGE_NAME = 'web-image-latest'
        DOCKER_REPO = 'ronn4/repo1'
        DOCKERHUB_CREDENTIALS = 'dockerhub'
    }

    stages {

        stage('Build Docker Images') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${DOCKERHUB_CREDENTIALS}", usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    script {
                        echo 'Building Docker images...'
                        sh 'docker --version'
                        sh """
                            cd polybot
                            docker build -t ${DOCKER_REPO}:${APP_IMAGE_NAME} .
                            docker build -t ${DOCKER_REPO}:${WEB_IMAGE_NAME} .
                        """
                        echo 'Docker images built successfully'
                    }
                }
            }
        }

        stage('Push Docker Images') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${DOCKERHUB_CREDENTIALS}", usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    script {
                        echo 'Pushing Docker images to Docker Hub...'
                        sh """
                            echo ${PASS} | docker login -u ${USER} --password-stdin
                            docker push ${DOCKER_REPO}:${APP_IMAGE_NAME}
                            docker push ${DOCKER_REPO}:${WEB_IMAGE_NAME}
                        """
                        echo 'Docker images pushed successfully'
                    }
                }
            }
        }

        stage('Deploy with Helm') {
            steps {
                script {
                    echo 'Deploying with Helm...'
                    sh 'helm version'
                    sh 'kubectl config set-context --current --namespace=demoapp'
                    sh """
                    helm upgrade --install deploy-demo-0.1.0 ./my-python-app-chart \
                    --namespace demoapp \
                    --set image.repository=${DOCKER_REPO} \
                    --set image.tag=latest \
                    --set replicas=3
                    """
                    echo 'Deployment successful'
                }
            }
        }
    }

    post {
        always {
            script {
                sh 'docker rmi ${DOCKER_REPO}/${APP_IMAGE_NAME}:latest || true'
                sh 'docker rmi ${DOCKER_REPO}/${WEB_IMAGE_NAME}:latest || true'
            }
            cleanWs()
        }
    }
}
