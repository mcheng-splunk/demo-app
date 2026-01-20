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
	    sh '''
                mvn clean verify org.sonarsource.scanner.maven:sonar-maven-plugin:5.5.0.6356:sonar \
                    -Dsonar.projectKey=demo-app \
                    -Dsonar.projectName='demo-app' \
                    -Dsonar.host.url=http://18.142.54.82:30850 \
                    -Dsonar.token=sqp_37a8172c364b4d30835af23bbf64237845b46bd4
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

