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
          echo "Validatig Raw Manifests..."
          sh 'kubectl apply --dry-run=client -f core/networking/cloudflared/values.yaml'
          sh 'kubectl apply --dry-run=client -f core/git/gitea/values.yaml'
          sh 'kubectl apply --dry-run=client -f core/observability/rbac.yaml'
          sh 'kubectl apply --dry-run=client -f core/cicd/jenkins/rbac.yaml'
        }
        container('helm') {
          echo "Validating Helm Charts..."

          sh 'helm repo add headlamp https://kubernetes-sigs.github.io/headlamp/'
          sh 'helm template headlamp headlamp/headlamp -f core/observability/headlamp/values.yaml'
          sh 'helm repo add jenkins https://charts.jenkins.io'
          sh 'helm template jenkins jenkins/jenkins -f core/cicd/jenkins/values.yaml'
        }
      }
    }
  }
}
