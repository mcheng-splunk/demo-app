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
    stage('Snyk Dependency Scan') {
      steps {
        container('snyk') {
          withCredentials([string(credentialsId: 'Snyk-token', variable: 'SNYK_TOKEN')]) {
            sh '''
              snyk auth $SNYK_TOKEN
              snyk test --severity-threshold=high || true
            '''
          }
        }
      }
    } // end of Synk Dependency Scan
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
      stage("Quality Gate") {
        steps {
          timeout(time: 1, unit: 'HOURS') {
          // Parameter indicates whether to set pipeline to UNSTABLE if Quality Gate fails
          // true = set pipeline to UNSTABLE, false = don't
          waitForQualityGate abortPipeline: true
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
    stage('Trivy Scan') {
      steps {
        // Step 1: Run Trivy scan in trivy container
        container('trivy') {
            script {
                def trivyReportFile = "${WORKSPACE}/trivy_report_${JOB_NAME}_${BUILD_NUMBER}.json"
                def htmlReport = "${WORKSPACE}/trivy_report_${JOB_NAME}_${BUILD_NUMBER}.html"
                def templateFile = "${WORKSPACE}/html.tpl"

                echo "Scanning Docker image ${DOCKER_HUB_REPO}:${BUILD_NUMBER}..."
                sh """
                    # Run scan once (JSON)
                    trivy image --format json --output ${trivyReportFile} ${DOCKER_HUB_REPO}:${BUILD_NUMBER}

                    # Convert JSON to HTML (no rescan)
                    trivy convert \
                      --format template \
                      --template "@${WORKSPACE}/trivy/html.tpl"  \
                      --output ${htmlReport} \
                      ${trivyReportFile}
                """

                echo "HTML report generated: ${htmlReport}"

                // Archive only HTML for Jenkins UI download
                archiveArtifacts artifacts: "trivy_report_${JOB_NAME}_${BUILD_NUMBER}.html", fingerprint: true
            }
        } // end of trivy container

        // Step 2: Combine metadata and send to Splunk in a curl-capable container
        container('maven') {
            script {
                def trivyReportFile = "${WORKSPACE}/trivy_report_${JOB_NAME}_${BUILD_NUMBER}.json"
                def combinedFile = "${WORKSPACE}/trivy_combined_${JOB_NAME}_${BUILD_NUMBER}.json"

                // Read Trivy JSON
                def trivyData = readJSON file: trivyReportFile

                // Combine metadata with Trivy report
                def combinedMap = [
                    index: "jenkins_statistics",
                    sourcetype: "json:jenkins",
                    host: "jenkins",
                    source: "jenkins",
                    event: [
                        event_tag: "job_scan",
                        job_name: "${JOB_NAME}",
                        node_name: "${NODE_NAME}",
                        build_number: BUILD_NUMBER,
                        build_url: "${BUILD_URL}",
                        trivy_report: trivyData
                    ]
                ]

                // Write combined JSON
                def combinedJson = groovy.json.JsonOutput.toJson(combinedMap)
                writeFile file: combinedFile, text: combinedJson

                echo "Sending Trivy report to Splunk..."
                withCredentials([
                    string(credentialsId: 'splunk-hec-token', variable: 'HEC_TOKEN'),
                    string(credentialsId: 'splunk-hec-url', variable: 'SPLUNK_HEC_URL')
                ]) {
                    sh """
                        curl -k -s \$SPLUNK_HEC_URL \
                            -H "Authorization: Splunk \$HEC_TOKEN" \
                            -H "Content-Type: application/json" \
                            -d @${combinedFile} || true
                    """
                }
            }
        } // end of maven container
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
            }
	        }
      }
    } // end of post 
  }  

