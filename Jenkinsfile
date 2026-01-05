pipeline {
    agent {
        kubernetes {
            yamlFile 'jenkins-agent-pod.yaml'
        }
    }
    environment {
        DOCKER_CREDENTIALS = credentials('dockerhub-creds')
        KUBECONFIG_CRED = credentials('kubeconfig')
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/youruser/demo-app.git'
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
                container('docker') {
                    sh 'docker build -t your-dockerhub-username/demo-app:${BUILD_NUMBER} .'
                    sh 'echo $DOCKER_CREDENTIALS_PSW | docker login -u $DOCKER_CREDENTIALS_USR --password-stdin'
                    sh 'docker push your-dockerhub-username/demo-app:${BUILD_NUMBER}'
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                    sh 'kubectl --kubeconfig=$KUBECONFIG_FILE apply -f k8s/deployment.yaml'
                }
            }
        }
    }
}

