pipeline {
    agent {
        kubernetes {
            yamlFile 'jenkins-agent-pod.yaml'
        }
    }
    environment {
        DOCKER_HUB_REPO = 'melcheng/demo-app'
        BUILD_START = "${System.currentTimeMillis()}"
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
      stage('SonarQube'){
        steps {
	  container('maven') {
            withSonarQubeEnv(
              installationName: 'SonarQube', 
              credentialsId: 'sonarqube-integration'
          ) {
            // This expands the evironment variables SONAR_CONFIG_NAME, SONAR_HOST_URL, SONAR_AUTH_TOKEN that can be used by any script.
              echo "SONAR_HOST_URL: ${env.SONAR_HOST_URL}"
	      
	      // Run the SonarQube analysis
              sh '''
                mvn clean verify org.sonarsource.scanner.maven:sonar-maven-plugin:3.7.0.1746:sonar \
                        -Dsonar.projectKey=demo-app \
                        -Dsonar.projectName='demo-app' \
                        -Dsonar.host.url=$SONAR_HOST_URL \
                        -Dsonar.token=$SONAR_AUTH_TOKEN
              '''
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
    post {
      always {
        withCredentials([
	  string(credentialsId: 'splunk-hec-token', variable: 'HEC_TOKEN'),
	  string(credentialsId: 'splunk-hec-url', variable: 'SPLUNK_HEC_URL')]) {
            script {
                // compute job duration in seconds
                // def duration = (currentBuild.duration ?: 0) / 1000.0
		
		def durationMs = System.currentTimeMillis() - currentBuild.startTimeInMillis
		def duration = durationMs / 1000.0

                // prepare JSON payload
                def payloadFile = "/tmp/splunk_payload_${JOB_NAME}_${BUILD_NUMBER}.json"
                def jsonPayload = """{
                  "index": "jenkins_statistics",
                  "sourcetype": "json:jenkins",
                  "host": "jenkins",
                  "source": "jenkins",
                  "event": {
                      "event_tag": "job_monitor",
                      "job_name": "${JOB_NAME}",
                      "node_name": "${NODE_NAME}",
                      "job_duration": ${duration},
                      "build_number": ${BUILD_NUMBER},
                      "build_url": "${BUILD_URL}"
                  }
                }"""
                writeFile file: payloadFile, text: jsonPayload

                // Use shell variable for the file path to avoid interpolation warning
		sh """
  		  curl -k -s \$SPLUNK_HEC_URL \
    		  -H "Authorization: Splunk \$HEC_TOKEN" \
    		  -H "Content-Type: application/json" \
    		  -d @${payloadFile} || true
		"""
                //sh '''
                //    payload_file=''' + payloadFile + '''
                //    curl -k -s $SPLUNK_HEC \
                //        -H "Authorization: Splunk $HEC_TOKEN" \
                //        -H "Content-Type: application/json" \
                //        -d @$payload_file || true
                //'''
            }
	  }
        }
    }
  }  

