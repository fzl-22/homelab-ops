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
          sh 'kubectl apply --dry-run=client -f ./core/networking/cloudflared/cloudflared.yaml'
          sh 'kubectl apply --dry-run=client -f ./core/git/gitea/gitea.yaml'
          sh 'kubectl apply --dry-run=client -f ./core/observability/headlamp/rbac.yaml'
          sh 'kubectl apply --dry-run=client -f ./core/cicd/jenkins/rbac.yaml'
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

    stage('Deploy Infrastructure') {
      when {
        branch 'master'
      }
      steps {
        // Networking Layer
        container('kubectl') {
          script {
            echo "Deploying Networking..."
            sh 'kubectl create namespace networking --dry-run=client -o yaml | kubectl apply -f -'
            sh 'kubectl apply -f ./core/networking/cloudflared/cloudflared.yaml'
          }
        }

        // Git Layer
        container('kubectl') {
          script {
            echo "Deploying Git..."
            sh 'kubectl create namespace git --dry-run=client -o yaml | kubectl apply -f -'
            sh 'kubectl apply -f ./core/git/gitea/gitea.yaml'
          }
        }

        // Observability Layer
        container('helm') {
          script {
            echo "Deploying Observability..."
            sh "helm repo add headlamp https://kubernetes-sigs.github.io/headlamp/"
            sh "helm repo update headlamp"
            sh '''
              helm upgrade --install headlamp headlamp/headlamp \
                --namespace observability \
                --values core/observability/headlamp/values.yaml \
                --wait
            '''
          }
        }

        container('kubectl') {
          script {
            sh "kubectl apply -f ./core/observability/headlamp/rbac.yaml"
          }
        }
      }
    }
  }
}
