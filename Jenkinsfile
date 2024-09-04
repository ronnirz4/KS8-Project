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
         KUBECONFIG = "${env.WORKSPACE}/.kube/config"  // Add KUBECONFIG environment variable
        DOCKER_HUB_CREDENTIALS = credentials('dockerhub')  // Add DockerHub credentials
    }

        stage('Build Docker Image') {
            steps {
                container('jenkins-agent') {   // Ensure Docker commands run in the jenkins-agent container
                    script {
                        echo "Checking Docker installation"
                        sh 'docker --version || echo "Docker command failed"'

                        echo "Building Docker Image"
                        def commitHash = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                        // Use the Dockerfile in the polybot/ folder to build the PolyBot image
                        sh "docker build -t ronn4/mypolybot:${commitHash} -f polybot/Dockerfile ."
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                container('jenkins-agent') {   // Ensure Docker push runs in the jenkins-agent container
                    script {
                        def commitHash = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                        withCredentials([string(credentialsId: 'dockerhub', variable: 'DOCKER_HUB_CREDENTIALS')]) {
                            sh "echo ${DOCKER_HUB_CREDENTIALS} | docker login -u ronn4 --password-stdin"
                        }
                        sh "docker push ronn4/mypolybot:${commitHash}"
                    }
                }
            }
        }


        stage('Deploy to Kubernetes') {
            steps {
                container('jenkins-agent') {   // Ensure Kubernetes deployment runs in the jenkins-agent container with helm
                    withEnv(["KUBECONFIG=${env.KUBECONFIG}"]) {
                        sh 'helm upgrade --install my-polybot-app ./my-polybot-app-chart --namespace demoapp'
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
