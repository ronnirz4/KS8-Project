@Library('Shared-lib') _

pipeline {
    agent {
    kubernetes {
      yaml '''
        apiVersion: v1
        kind: Pod
        spec:
          containers:
          - name: jenkins-agent
            image: jenkins-agent:latest
            command:
            - cat
            tty: true
        '''
        defaultContainer:'jnlp'
    }
  }
        stage('Deploy with Helm') {
            steps {
                script {
                    // Ensure Helm is installed in the pod
                    bat 'helm version'

                    // Set up Kubernetes context for the desired namespace
                    bat 'kubectl config set-context --current --namespace=demoapp'

                    // Deploy the application using your Helm chart
                    bat """
                    helm upgrade --install deploy-demo-0.1.0 ./my-python-app-chart \
                    --namespace demo \
                    --set image.repository=${DOCKER_REPO} \
                    --set image.tag=${GIT_COMMIT} \
                    --set replicas=3
                    """
                }
            }
        }
    }
}
