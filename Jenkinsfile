pipeline {
    agent any

    environment {
        IMAGE_NAME = "trivy-test"
        TRIVY = "/usr/local/bin/trivy"
    }

    stages {

        stage('Trivy Dependency Scan') {
            steps {
                echo '🔍 Checking dependencies for vulnerabilities...'

                sh '''
                    ${TRIVY} fs \
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
                echo '🐳 Dependencies passed security scan.'
                echo '🐳 Building Docker image...'

                sh '''
                    docker build \
                      -t ${IMAGE_NAME}:${BUILD_NUMBER} \
                      .
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                echo '🔍 Scanning final Docker image...'

                sh '''
                    ${TRIVY} image \
                      --severity HIGH,CRITICAL \
                      --exit-code 1 \
                      --no-progress \
                      ${IMAGE_NAME}:${BUILD_NUMBER}
                '''
            }
        }

        stage('Security Passed') {
            steps {
                echo '✅ No blocking HIGH/CRITICAL vulnerabilities found!'
                echo '🚀 Application is ready for the next deployment stage.'
            }
        }
    }

    post {

        success {
            echo '========================================'
            echo '✅ PIPELINE SUCCESS'
            echo '✅ Dependency scan passed'
            echo '✅ Docker image scan passed'
            echo '========================================'
        }

        failure {
            echo '========================================'
            echo '❌ PIPELINE FAILED'
            echo '⚠️ Check the failed stage and Trivy output above.'
            echo '========================================'
        }

        always {
            echo "Build Number: ${BUILD_NUMBER}"
            echo "Docker Image: ${IMAGE_NAME}:${BUILD_NUMBER}"
        }
    }
}