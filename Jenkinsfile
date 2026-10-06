pipeline {
    agent any

    environment {
        IMAGE_NAME = "trivy-test"
        TRIVY = "/usr/local/bin/trivy"
        PATH = "/usr/local/bin:${env.PATH}"
    }

    stages {

        // ==========================================
        // 1. TRIVY DEPENDENCY SCAN
        // ==========================================
        stage('Trivy Dependency Scan') {
            steps {
                echo '🔍 Scanning project dependencies...'

                sh '''
                    rm -f trivy-report.json
                    rm -f trivy-failed

                    set +e

                    ${TRIVY} fs \
                      --scanners vuln \
                      --severity HIGH,CRITICAL \
                      --format json \
                      --output trivy-report.json \
                      --exit-code 1 \
                      --no-progress \
                      .

                    TRIVY_EXIT=$?

                    if [ "$TRIVY_EXIT" -ne 0 ]; then
                        echo "❌ Vulnerable dependency detected."
                        touch trivy-failed
                        exit "$TRIVY_EXIT"
                    fi

                    echo "✅ Dependency security scan passed."
                '''
            }
        }


        // ==========================================
        // 2. BUILD DOCKER IMAGE
        // Only runs if Trivy passes
        // ==========================================
        stage('Build Docker Image') {
            steps {
                echo '🐳 Dependency security scan passed.'
                echo '🐳 Building Docker image...'

                sh '''
                    docker build \
                      -t ${IMAGE_NAME}:${BUILD_NUMBER} \
                      .
                '''

                echo '✅ Docker image built successfully.'
            }
        }


        // ==========================================
        // 3. BUILD COMPLETE
        // ==========================================
        stage('Build Complete') {
            steps {
                echo '========================================'
                echo '✅ DEPENDENCY SCAN PASSED'
                echo '✅ DOCKER BUILD PASSED'
                echo '🚀 BUILD SUCCESS'
                echo '========================================'
            }
        }
    }


    post {

        // ==========================================
        // SUCCESS
        // ==========================================
        success {

            echo '========================================'
            echo '✅ PIPELINE SUCCESS'
            echo '========================================'

            script {

                // Get Git information
                def commitAuthor = sh(
                    script: 'git log -1 --pretty=format:%an',
                    returnStdout: true
                ).trim()

                def commitMessage = sh(
                    script: 'git log -1 --pretty=format:%s',
                    returnStdout: true
                ).trim()

                def shortCommit = sh(
                    script: 'git rev-parse --short HEAD',
                    returnStdout: true
                ).trim()


                // Load Telegram credentials
                withCredentials([

                    string(
                        credentialsId: 'telegram-bot-token',
                        variable: 'TELEGRAM_BOT_TOKEN'
                    ),

                    string(
                        credentialsId: 'telegram-chat-id',
                        variable: 'TELEGRAM_CHAT_ID'
                    ),

                    string(
                        credentialsId: 'telegram-topic-id',
                        variable: 'TELEGRAM_TOPIC_ID'
                    )

                ]) {

                    withEnv([
                        "COMMIT_AUTHOR=${commitAuthor}",
                        "COMMIT_MESSAGE=${commitMessage}",
                        "SHORT_COMMIT=${shortCommit}"
                    ]) {

                        sh(
                            returnStatus: true,
                            script: '''
                                MESSAGE="✅ Jenkins Build SUCCESS

Project: $JOB_NAME
Build: #$BUILD_NUMBER
Branch: $GIT_BRANCH
Developer: $COMMIT_AUTHOR
Commit ID: $SHORT_COMMIT

Commit:
$COMMIT_MESSAGE

🔐 Security Report

✅ Dependency Scan: PASSED
✅ No blocking HIGH/CRITICAL vulnerabilities
✅ Docker Build: PASSED

🐳 Docker Image:
$IMAGE_NAME:$BUILD_NUMBER

🚀 Build completed successfully."

                                curl -sS -X POST \
                                  "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
                                  --data-urlencode "chat_id=${TELEGRAM_CHAT_ID}" \
                                  --data-urlencode "message_thread_id=${TELEGRAM_TOPIC_ID}" \
                                  --data-urlencode "text=${MESSAGE}"
                            '''
                        )
                    }
                }
            }
        }


        // ==========================================
        // FAILURE
        // ==========================================
        failure {

            echo '========================================'
            echo '❌ PIPELINE FAILED'
            echo '========================================'

            script {

                // Get Git information
                def commitAuthor = sh(
                    script: 'git log -1 --pretty=format:%an',
                    returnStdout: true
                ).trim()

                def commitMessage = sh(
                    script: 'git log -1 --pretty=format:%s',
                    returnStdout: true
                ).trim()

                def shortCommit = sh(
                    script: 'git rev-parse --short HEAD',
                    returnStdout: true
                ).trim()


                // Default error message
                def reportSummary = """
❌ Pipeline failed.

Please check Jenkins Console Output.
"""


                // ==================================
                // TRIVY DEPENDENCY FAILURE REPORT
                // ==================================
                if (
                    fileExists('trivy-failed') &&
                    fileExists('trivy-report.json')
                ) {

                    reportSummary = sh(
                        script: '''
python3 <<'PY'

import json


# ==========================================
# Read Trivy JSON report
# ==========================================

with open("trivy-report.json") as f:
    report = json.load(f)


vulnerabilities = []


for result in report.get("Results", []):

    for vuln in result.get("Vulnerabilities") or []:

        vulnerabilities.append(vuln)


# ==========================================
# Count severity
# ==========================================

high = sum(
    1 for vuln in vulnerabilities
    if vuln.get("Severity") == "HIGH"
)


critical = sum(
    1 for vuln in vulnerabilities
    if vuln.get("Severity") == "CRITICAL"
)


# ==========================================
# Find vulnerable packages
# ==========================================

packages = {}


for vuln in vulnerabilities:

    package = vuln.get(
        "PkgName",
        "Unknown"
    )

    version = vuln.get(
        "InstalledVersion",
        "Unknown"
    )

    packages[f"{package} {version}"] = True


# ==========================================
# Create Telegram report
# ==========================================

print("🚨 DEPENDENCY SECURITY FAILED")

print()

print("📊 Security Summary")
print(f"HIGH: {high}")
print(f"CRITICAL: {critical}")

print()

print("⚠️ Vulnerable Packages")


for package in list(packages.keys())[:5]:

    print(f"• {package}")


print()

print("🚨 Vulnerabilities")


# Only show first 5 vulnerabilities
# to keep Telegram message short

for vuln in vulnerabilities[:5]:

    cve = vuln.get(
        "VulnerabilityID",
        "Unknown"
    )

    severity = vuln.get(
        "Severity",
        "Unknown"
    )

    package = vuln.get(
        "PkgName",
        "Unknown"
    )

    installed = vuln.get(
        "InstalledVersion",
        "Unknown"
    )

    fixed = vuln.get(
        "FixedVersion"
    ) or "No fix available"


    print(
        f"• {cve} [{severity}]"
    )

    print(
        f"  Package: {package}"
    )

    print(
        f"  Installed: {installed}"
    )

    print(
        f"  Fix: {fixed}"
    )

    print()


# ==========================================
# Show remaining vulnerability count
# ==========================================

if len(vulnerabilities) > 5:

    print(
        f"...and {len(vulnerabilities) - 5} more vulnerabilities."
    )


print()

print("❌ Docker build BLOCKED.")

print(
    "Fix the vulnerable dependencies and push again."
)

PY
                        ''',
                        returnStdout: true
                    ).trim()
                }


                // ==================================
                // TELEGRAM CREDENTIALS
                // ==================================
                withCredentials([

                    string(
                        credentialsId: 'telegram-bot-token',
                        variable: 'TELEGRAM_BOT_TOKEN'
                    ),

                    string(
                        credentialsId: 'telegram-chat-id',
                        variable: 'TELEGRAM_CHAT_ID'
                    ),

                    string(
                        credentialsId: 'telegram-topic-id',
                        variable: 'TELEGRAM_TOPIC_ID'
                    )

                ]) {

                    withEnv([

                        "COMMIT_AUTHOR=${commitAuthor}",
                        "COMMIT_MESSAGE=${commitMessage}",
                        "SHORT_COMMIT=${shortCommit}",
                        "REPORT_SUMMARY=${reportSummary}"

                    ]) {

                        sh(
                            returnStatus: true,
                            script: '''
                                MESSAGE="🚨  Build FAILED

Project: $JOB_NAME
Build: #$BUILD_NUMBER
Branch: $GIT_BRANCH
Developer: $COMMIT_AUTHOR
Commit ID: $SHORT_COMMIT

Commit:
$COMMIT_MESSAGE

$REPORT_SUMMARY"

                                curl -sS -X POST \
                                  "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
                                  --data-urlencode "chat_id=${TELEGRAM_CHAT_ID}" \
                                  --data-urlencode "message_thread_id=${TELEGRAM_TOPIC_ID}" \
                                  --data-urlencode "text=${MESSAGE}"
                            '''
                        )
                    }
                }
            }
        }


        // ==========================================
        // ALWAYS
        // ==========================================
        always {

            echo '========================================'
            echo "Build Number: ${BUILD_NUMBER}"
            echo "Docker Image: ${IMAGE_NAME}:${BUILD_NUMBER}"
            echo '========================================'


            // Save Trivy report inside Jenkins
            archiveArtifacts(
                artifacts: 'trivy-report.json',
                allowEmptyArchive: true
            )
        }
    }
}