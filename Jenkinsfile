pipeline {
    agent any

    environment {
        IMAGE_NAME = "trivy-test"
        TRIVY = "/usr/local/bin/trivy"
        PATH = "/usr/local/bin:${env.PATH}"
    }

    stages {

        // ============================================
        // 1. TRIVY DEPENDENCY SCAN
        // ============================================
        stage('Trivy Dependency Scan') {
            steps {
                echo '🔍 Checking dependencies for vulnerabilities...'

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
                        echo "❌ Trivy dependency scan failed."
                        touch trivy-failed
                        exit "$TRIVY_EXIT"
                    fi

                    echo "✅ Dependency security scan passed."
                '''
            }
        }


        // ============================================
        // 2. BUILD DOCKER IMAGE
        // ============================================
        stage('Build Docker Image') {
            steps {
                echo '🐳 Dependencies are safe.'
                echo '🐳 Building Docker image...'

                sh '''
                    docker build \
                      -t ${IMAGE_NAME}:${BUILD_NUMBER} \
                      .
                '''

                echo '✅ Docker image built successfully.'
            }
        }


        // ============================================
        // 3. TRIVY IMAGE SCAN
        // ============================================
        stage('Trivy Image Scan') {
            steps {
                echo '🔍 Scanning final Docker image...'

                sh '''
                    rm -f trivy-image-report.json
                    rm -f trivy-image-failed

                    set +e

                    ${TRIVY} image \
                      --severity HIGH,CRITICAL \
                      --format json \
                      --output trivy-image-report.json \
                      --exit-code 1 \
                      --no-progress \
                      ${IMAGE_NAME}:${BUILD_NUMBER}

                    TRIVY_EXIT=$?

                    if [ "$TRIVY_EXIT" -ne 0 ]; then
                        echo "❌ Trivy image scan failed."
                        touch trivy-image-failed
                        exit "$TRIVY_EXIT"
                    fi

                    echo "✅ Docker image security scan passed."
                '''
            }
        }


        // ============================================
        // 4. SECURITY PASSED
        // ============================================
        stage('Security Passed') {
            steps {
                echo '========================================'
                echo '✅ SECURITY CHECKS PASSED'
                echo '✅ Dependency scan passed'
                echo '✅ Docker image built'
                echo '✅ Docker image scan passed'
                echo '🚀 Application can continue to deployment'
                echo '========================================'
            }
        }
    }


    // ================================================
    // POST ACTIONS
    // ================================================
    post {

        // ============================================
        // SUCCESS
        // ============================================
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

                def shortCommit = sh(
                    script: 'git rev-parse --short HEAD',
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
✅ Docker Build: PASSED
✅ Docker Image Scan: PASSED

🐳 Image:
$IMAGE_NAME:$BUILD_NUMBER

🚀 Application is ready for the next deployment stage."

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


        // ============================================
        // FAILURE
        // ============================================
        failure {

            echo '========================================'
            echo '❌ PIPELINE FAILED'
            echo '========================================'

            script {

                // ------------------------------------
                // Get Git information
                // ------------------------------------

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


                // Default failure message
                def reportSummary = """
❌ Pipeline failed.

Check Jenkins Console Output for more information.
"""


                // ====================================
                // DEPENDENCY SCAN FAILED
                // ====================================

                if (
                    fileExists('trivy-failed') &&
                    fileExists('trivy-report.json')
                ) {

                    reportSummary = sh(
                        script: '''
python3 <<'PY'

import json

with open("trivy-report.json") as f:
    report = json.load(f)


vulnerabilities = []

for result in report.get("Results", []):
    for vuln in result.get("Vulnerabilities") or []:
        vulnerabilities.append(vuln)


high = sum(
    1 for vuln in vulnerabilities
    if vuln.get("Severity") == "HIGH"
)

critical = sum(
    1 for vuln in vulnerabilities
    if vuln.get("Severity") == "CRITICAL"
)


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


print("🔍 Trivy Dependency Scan: FAILED")

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

for vuln in vulnerabilities[:5]:

    cve = vuln.get(
        "VulnerabilityID",
        "Unknown"
    )

    severity = vuln.get(
        "Severity",
        "Unknown"
    )

    fixed = vuln.get(
        "FixedVersion"
    ) or "No fix available"

    print(
        f"• {cve} [{severity}]"
    )

    print(
        f"  Fix: {fixed}"
    )


if len(vulnerabilities) > 5:

    print()

    print(
        f"...and {len(vulnerabilities) - 5} more."
    )


print()

print(
    "❌ Build blocked before Docker build."
)

PY
                        ''',
                        returnStdout: true
                    ).trim()
                }


                // ====================================
                // IMAGE SCAN FAILED
                // ====================================

                else if (
                    fileExists('trivy-image-failed') &&
                    fileExists('trivy-image-report.json')
                ) {

                    reportSummary = sh(
                        script: '''
python3 <<'PY'

import json

with open("trivy-image-report.json") as f:
    report = json.load(f)


vulnerabilities = []

for result in report.get("Results", []):
    for vuln in result.get("Vulnerabilities") or []:
        vulnerabilities.append(vuln)


high = sum(
    1 for vuln in vulnerabilities
    if vuln.get("Severity") == "HIGH"
)

critical = sum(
    1 for vuln in vulnerabilities
    if vuln.get("Severity") == "CRITICAL"
)


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


print("🐳 Trivy Docker Image Scan: FAILED")

print()

print("📊 Security Summary")
print(f"HIGH: {high}")
print(f"CRITICAL: {critical}")

print()

print("⚠️ Vulnerable Packages")

for package in list(packages.keys())[:5]:

    print(
        f"• {package}"
    )


print()

print("🚨 Vulnerabilities")


for vuln in vulnerabilities[:5]:

    cve = vuln.get(
        "VulnerabilityID",
        "Unknown"
    )

    severity = vuln.get(
        "Severity",
        "Unknown"
    )

    fixed = vuln.get(
        "FixedVersion"
    ) or "No fix available"


    print(
        f"• {cve} [{severity}]"
    )

    print(
        f"  Fix: {fixed}"
    )


if len(vulnerabilities) > 5:

    print()

    print(
        f"...and {len(vulnerabilities) - 5} more."
    )


print()

print(
    "❌ Docker image blocked from deployment."
)

PY
                        ''',
                        returnStdout: true
                    ).trim()
                }


                // ====================================
                // SEND TELEGRAM
                // ====================================

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
                        "COMMIT_MESSAGE=${commitMessage}",
                        "SHORT_COMMIT=${shortCommit}",
                        "REPORT_SUMMARY=${reportSummary}"

                    ]) {

                        sh(
                            returnStatus: true,
                            script: '''
                                MESSAGE="🚨 Jenkins Build FAILED

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
                                  --data-urlencode "text=${MESSAGE}"
                            '''
                        )
                    }
                }
            }
        }


        // ============================================
        // ALWAYS
        // ============================================
        always {

            echo '========================================'
            echo "Build Number: ${BUILD_NUMBER}"
            echo "Docker Image: ${IMAGE_NAME}:${BUILD_NUMBER}"
            echo '========================================'

            archiveArtifacts(
                artifacts: 'trivy-report.json,trivy-image-report.json',
                allowEmptyArchive: true
            )
        }
    }
}