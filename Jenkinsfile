pipeline {
    agent {
        kubernetes {
            yamlFile 'jenkins-agent-pod.yaml'
        }
    }
    environment {
        DOCKER_HUB_REPO = 'melcheng/demo-app'
    }
    stages {
      stage('Checkout') {
        steps {
          git branch: 'main', url: 'https://github.com/mcheng-splunk/demo-app.git'
        }
      }
      stage('Build') {
        steps {
          container('maven') {
            sh 'mvn clean package'
          }
        }
      }
      stage('Check Kaniko Docker Config') {
        steps {
          container('kaniko') {
            sh '''
              echo "Checking /kaniko/.docker"
              ls -l /kaniko/.docker
              cat /kaniko/.docker/config.json
            '''
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

