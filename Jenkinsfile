pipeline {
  agent {
    kubernetes {
      yaml '''
      apiVersion: v1
      kind: Pod
      spec:
        serviceAccountName: default
        containers:
        - name: kubectl
          image: bitnami/kubectl:latest
          command: ['/bin/sh', '-c', 'sleep infinity']
          tty: true
          securityContext:
            runAsUser: 0
        - name: helm
          image: alpine/helm:3.20.0
          command: ['/bin/sh', '-c', 'sleep infinity']
          tty: true
          securityAsContext:
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
          echo "Checking Kubernetes Manifests..."
          sh 'kubectl apply --dry-run=client -f core/networking/cloudflared/values.yaml'
          sh 'kubectl apply --dry-run=client -f core/git/gitea/values.yaml'
          sh 'kubectl apply --dry-run=client -f core/cicd/jenkins/values.yaml'
        }
        container('helm') {
          echo "Linting Helm Values..."
          sh 'helm repo add headlamp https://kubernetes-sigs.github.io/headlamp/'
          sh 'helm template headlamp headlamp/headlamp -f core/observability/headlamp/values.yaml'
        }
      }
    }
  }
}
