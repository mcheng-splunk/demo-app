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
      stage('SonarQube Scan') {
        steps {
          container('maven') {
            withSonarQubeEnv('sonarqube') {
		sh 'mvn clean package sonar:sonar'
            }
          }
        }
      }	
      stage('Quality Gate') {
        steps {
          timeout(time: 5, unit: 'MINUTES') {
	    script {
              def qg = waitForQualityGate()
              if (qg.status != 'OK') {
                  error "Pipeline aborted due to quality gate failure: ${qg.status}"
              }
	    }
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
               echo "Retreive lastest image tag:"
               sed -i "s|image: melcheng/demo-app:.*|image: ${DOCKER_HUB_REPO}:${BUILD_NUMBER}|g" k8s/deployment.yaml

               echo "Test deployment apply..."
               kubectl apply -f k8s/deployment.yaml
             '''
          }
        }
      }
    }
  }  

