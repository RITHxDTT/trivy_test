pipeline {
    agent any

    environment {
        IMAGE_NAME = "trivy-test"
        TRIVY = "/usr/local/bin/trivy"
        PATH = "/usr/local/bin:${env.PATH}"
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
            echo '========================================'

            script {
                def commitAuthor = sh(
                    script: 'git log -1 --pretty=format:%an',
                    returnStdout: true
                ).trim()

                def commitMessage = sh(
                    script: 'git log -1 --pretty=format:%s',
                    returnStdout: true
                ).trim()

                withCredentials([
                    string(
                        credentialsId: 'telegram-bot-token',
                        variable: 'TELEGRAM_BOT_TOKEN'
                    ),
                    string(
                        credentialsId: 'telegram-chat-id',
                        variable: 'TELEGRAM_CHAT_ID'
                    )
                ]) {
                    withEnv([
                        "COMMIT_AUTHOR=${commitAuthor}",
                        "COMMIT_MESSAGE=${commitMessage}"
                    ]) {
                        sh(
                            returnStatus: true,
                            script: '''
                                MESSAGE="✅ Jenkins Build SUCCESS

Project: $JOB_NAME
Build: #$BUILD_NUMBER
Branch: $GIT_BRANCH
Developer: $COMMIT_AUTHOR

Commit:
$COMMIT_MESSAGE

✅ Dependency scan passed
✅ Docker image built
✅ Docker image security scan passed

Image: $IMAGE_NAME:$BUILD_NUMBER"

                                curl -sS -X POST \
                                  "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
                                  --data-urlencode "chat_id=${TELEGRAM_CHAT_ID}" \
                                  --data-urlencode "text=${MESSAGE}"
                            '''
                        )
                    }
                }
            }
        }

        failure {
            echo '========================================'
            echo '❌ PIPELINE FAILED'
            echo '⚠️ Check the failed stage and console output.'
            echo '========================================'

            script {
                def commitAuthor = sh(
                    script: 'git log -1 --pretty=format:%an',
                    returnStdout: true
                ).trim()

                def commitMessage = sh(
                    script: 'git log -1 --pretty=format:%s',
                    returnStdout: true
                ).trim()

                withCredentials([
                    string(
                        credentialsId: 'telegram-bot-token',
                        variable: 'TELEGRAM_BOT_TOKEN'
                    ),
                    string(
                        credentialsId: 'telegram-chat-id',
                        variable: 'TELEGRAM_CHAT_ID'
                    )
                ]) {
                    withEnv([
                        "COMMIT_AUTHOR=${commitAuthor}",
                        "COMMIT_MESSAGE=${commitMessage}"
                    ]) {
                        sh(
                            returnStatus: true,
                            script: '''
                                MESSAGE="🚨 Jenkins Build FAILED

Project: $JOB_NAME
Build: #$BUILD_NUMBER
Branch: $GIT_BRANCH
Developer: $COMMIT_AUTHOR

Commit:
$COMMIT_MESSAGE

❌ Pipeline failed

Possible reasons:
• Vulnerable dependency detected
• Docker build failed
• Trivy image scan failed
• Other pipeline error

Please check Jenkins Console Output."

                                curl -sS -X POST \
                                  "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
                                  --data-urlencode "chat_id=${TELEGRAM_CHAT_ID}" \
                                  --data-urlencode "text=${MESSAGE}"
                            '''
                        )
                    }
                }
            }
        }

        always {
            echo "Build Number: ${BUILD_NUMBER}"
            echo "Docker Image: ${IMAGE_NAME}:${BUILD_NUMBER}"
        }
    }
}