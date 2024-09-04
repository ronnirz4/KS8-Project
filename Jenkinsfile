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
        APP_IMAGE_NAME ='app-image'
        WEB_IMAGE_NAME = 'web-image'
        DOCKER_COMPOSE_FILE = 'compose.yaml'
        DOCKER_REPO = 'ronn4/repo1'
        DOCKERHUB_CREDENTIALS = 'dockerhub'
        SNYK_API_TOKEN = 'SNYK_API_TOKEN'
    }

    stages {
        stage('Checkout and Extract Git Commit Hash') {
            steps {
                echo 'Starting Checkout Stage...'
                checkout scm
                echo 'Completed Checkout Stage'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo 'Starting Docker Build Stage...'
                    try {
                        sh """
                            docker-compose -f ${DOCKER_COMPOSE_FILE} build
                        """
                        echo 'Docker Build Stage Completed'
                    } catch (Exception e) {
                        echo "Error in Docker Build Stage: ${e}"
                        throw e
                    }
                }
            }
        }

        stage('Login, Tag, and Push Images') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    script {
                        echo 'Starting Login, Tag, and Push Images Stage...'
                        try {
                            sh 'git rev-parse --short HEAD > gitCommit.txt'
                            def GITCOMMIT = readFile('gitCommit.txt').trim()
                            def IMAGE_TAG = "v1.0.0-${BUILD_NUMBER}-${GITCOMMIT}"
                            sh """
                                cd polybot
                                docker login -u ${USER} -p ${PASS}
                                docker tag ${APP_IMAGE_NAME}:latest ${DOCKER_REPO}/${APP_IMAGE_NAME}:${IMAGE_TAG}
                                docker tag ${WEB_IMAGE_NAME}:latest ${DOCKER_REPO}/${WEB_IMAGE_NAME}:${IMAGE_TAG}
                                docker push ${DOCKER_REPO}/${APP_IMAGE_NAME}:${IMAGE_TAG}
                                docker push ${DOCKER_REPO}/${WEB_IMAGE_NAME}:${IMAGE_TAG}
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

        stage('Static Code Linting and Unittest') {
            parallel {
                stage('Static code linting') {
                    steps {
                        script {
                            echo 'Starting Static Code Linting Stage...'
                            try {
                                sh """
                                    python -m pylint -f parseable --reports=no polybot/*.py > pylint.log
                                    cat pylint.log
                                """
                                echo 'Static Code Linting Stage Completed'
                            } catch (Exception e) {
                                echo "Error in Static Code Linting Stage: ${e}"
                                throw e
                            }
                        }
                    }
                }

                stage('Unittest') {
                    steps {
                        script {
                            echo 'Starting Unittest Stage...'
                            try {
                                sh 'python -m pytest --junitxml results.xml polybot/test'
                                echo 'Unittest Stage Completed'
                            } catch (Exception e) {
                                echo "Error in Unittest Stage: ${e}"
                                throw e
                            }
                        }
                    }
                }
            }
        }

        stage('Deploy with Helm') {
            steps {
                script {
                    sh 'helm version'
                    sh 'kubectl config set-context --current --namespace=demoapp'
                    def GITCOMMIT = readFile('gitCommit.txt').trim()
                    sh """
                    helm upgrade --install deploy-demo-0.1.0 ./my-python-app-chart \
                    --namespace demoapp \
                    --set image.repository=${DOCKER_REPO} \
                    --set image.tag=${GITCOMMIT} \
                    --set replicas=3
                    """
                }
            }
        }
    }
}
