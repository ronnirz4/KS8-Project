pipeline {
    agent {
        kubernetes {
            yaml """
            apiVersion: v1
            kind: Pod
            spec:
              containers:
              - name: jenkins-slave
                image: ronn4/repo1:jenkins-agent
                securityContext:
                  privileged: true       # Enable privileged mode for Docker
                  runAsUser: 0           # Run as root user to access Docker socket
                command:
                - sh
                - -c
                - |
                  git config --global --add safe.directory /home/jenkins/agent/workspace/kubernetes-project-pipeline
                  cat
                tty: true
                volumeMounts:
                - mountPath: /var/run/docker.sock
                  name: docker-sock
                - mountPath: /home/jenkins/agent
                  name: workspace-volume
              volumes:
              - hostPath:
                  path: /var/run/docker.sock
                name: docker-sock
              - emptyDir:
                  medium: ""
                name: workspace-volume
            """
        }
    }

    environment {
        APP_IMAGE_NAME = 'app-image-latest'
        WEB_IMAGE_NAME = 'web-image-latest'
        DOCKER_REPO = 'ronn4/repo1'
        DOCKERHUB_CREDENTIALS = 'dockerhub'
    }

    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Verify Files') {
            steps {
                sh 'ls -R ./my-python-app-chart'
            }
        }

        stage('Build Docker Image') {
            steps {
                container('jenkins-agent') {   // Ensure Docker commands run in the jenkins-agent container
                    script {
                        withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                            echo "Checking Docker installation"
                            sh 'docker --version || echo "Docker command failed"'

                            echo "Building Docker Image"
                            def commitHash = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                            // Set the build context to the polybot/ folder
                            sh "docker build -t ronn4/repo1/app-image-latest:${commitHash} polybot/"
                        }
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                container('jenkins-agent') {   // Ensure Docker push runs in the jenkins-agent container
                    script {
                        def commitHash = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                        withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                            // Using DockerHub credentials for login
                            sh "echo ${PASS} | docker login -u ${USER} --password-stdin"
                            sh "docker push ronn4/repo1/app-image-latest:${commitHash}"
                        }
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                container('jenkins-agent') {   // Ensure Kubernetes deployment runs in the jenkins-agent container with helm
                    withEnv(["KUBECONFIG=${env.KUBECONFIG}"]) {
                        sh 'helm upgrade --install my-python-app ./my-python-app-chart --namespace demoapp'
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}