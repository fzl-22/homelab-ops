pipeline {
  agent {
    kubernetes {
      yaml '''
      apiVersion: v1
      kind: pod
      spec:
        serviceAccountName: default
        containers:
        - name: kubectl
          image: bitnami/kubectl:latest
          command: ['/bin/sh', '-c', 'sleep infinity']
          tty: true
          securityContext:
            runAsUser: 0
      '''
    }
  }

  options {
    disableConcurrentBuilds()
  }

  stages {
    stage('Validate') {
      steps {
        container('kubectl') {
          sh 'kubectl apply --dry-run=client -f core/networking/cloudflared/values.yaml'
          sh 'kubectl apply --dry-run=client -f core/git/gitea/values.yaml'
          sh 'kubectl apply --dry-run=client -f core/observability/headlamp/values.yaml'
          sh 'kubectl apply --dry-run=client -f core/cicd/jenkins/values.yaml'
        }
      }
    }
  }
}
