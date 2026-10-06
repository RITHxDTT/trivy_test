pipeline {
    agent any

    environment {
        IMAGE_NAME = "trivy-test"
    }

    stages {

        stage('Trivy Dependency Scan') {
            steps {
                echo '🔍 Checking dependencies for vulnerabilities...'

                sh '''
                    trivy fs \
                      --scanners vuln \
                      --severity HIGH,CRITICAL \
                      --exit-code 1 \
                      --no-progress \
                      .
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                echo '🐳 Dependencies are safe. Building Docker image...'

                sh '''
                    docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                echo '🔍 Scanning final Docker image...'

                sh '''
                    trivy image \
                      --severity HIGH,CRITICAL \
                      --exit-code 1 \
                      --no-progress \
                      ${IMAGE_NAME}:${BUILD_NUMBER}
                '''
            }
        }

        stage('Security Passed') {
            steps {
                echo '✅ Security checks passed!'
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline SUCCESS - dependencies and image are safe.'
        }

        failure {
            echo '🚨 Pipeline FAILED - HIGH/CRITICAL vulnerability detected!'
        }
    }
}