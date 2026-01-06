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
          container('kubectl') {
             sh '''
               echo "Current namespace:"
               kubectl config view --minify --output 'jsonpath={..namespace}'

               echo "Test deployment apply..."
               kubectl apply -f k8s/deployment.yaml
             '''
          }
        }
      }
    }
  }  

