pipeline {
    agent {
        kubernetes {
            yaml """
            apiVersion: v1
            kind: Pod
            spec:
              serviceAccountName: jenkins-admin
              containers:
              - name: jenkins-agent
                image: ronn4/repo1:jenkins-agent-new
                securityContext:
                  privileged: true       # Enable privileged mode for Docker
                  runAsUser: 0           # Run as root user to access Docker socket
                command:
                - sh
                - -c
                - |
                  git config --global --add safe.directory /home/jenkins/agent/workspace/K8S_Project
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
        stage('Install Python Requirements') {
            steps {
                container('jenkins-agent') {
                    // Install Python dependencies
                    sh """
                        python3 -m venv venv
                        . venv/bin/activate
                        pip install pytest unittest2 pylint flask telebot Pillow loguru matplotlib
                    """
                }
            }
        }
        stage('Unittest') {
            steps {
                container('jenkins-agent') {
                    // Run unittests
                    sh """
                        python3 -m venv venv
                        . venv/bin/activate
                        python -m pytest --junitxml results.xml polybot/test
                    """
                }
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
                            sh "docker build -t ronn4/repo1:app-image-latest polybot/"
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
                            sh "docker login -u ${USER} -p ${PASS}"
                            sh "docker push ronn4/repo1:app-image-latest"
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

        stage('Create/Update ArgoCD Application') {
            steps {
                container('jenkins-agent') {
                    script {
                        // Make sure 'application.yaml' is in the 'argocd-config' folder
                        sh 'kubectl apply -f argocd-config/application.yaml -n argocd'
                    }
                }
            }
        }

        stage('Sync ArgoCD Application') {
            steps {
                container('jenkins-agent') {
                    script {
                        sh 'kubectl -n argocd patch application my-python-app --type merge -p \'{"metadata": {"annotations": {"argocd.argoproj.io/sync-wave": "-1"}}}\''
                    }
                }
            }
        }
    }
}