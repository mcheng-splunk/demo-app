pipeline {
    agent {
        kubernetes {
            yamlFile 'jenkins-agent-pod.yaml'
        }
    }
    environment {
        DOCKER_HUB_REPO = 'melcheng/demo-app'
        GITHUB_REPO = 'mcheng-splunk'
    }
    stages {
      stage('Checkout') {
        steps {
          git branch: 'main', url: 'https://github.com/${GITHUB_REPO}/demo-app.git'
        }
      }
      stage('Build') {
        steps {
          container('maven') {
            sh 'mvn clean package'
          }
        }
      }
      stage('Build Docker Image') {
        steps {
          container('kaniko') {
            sh '''
              /kaniko/executor \
                --dockerfile=Dockerfile \
                --context=$(pwd) \
                --destination=${DOCKER_HUB_REPO}:${BUILD_NUMBER}
            '''
          }
        }
      }
      stage('Deploy to Kubernetes') {
        steps {
          container('maven') {
            sh '''
              kubectl apply -f k8s/deployment.yaml
              kubectl apply -f k8s/service.yaml
            '''
          }
        }
      }
    }
  }  

