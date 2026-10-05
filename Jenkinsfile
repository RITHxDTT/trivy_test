pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                echo '📥 Checking out source code...'
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo '🔨 Building application...'
                sh 'python3 --version'
            }
        }

        stage('Test') {
            steps {
                echo '🧪 Testing Python code...'
                sh 'python3 -m py_compile app.py'
            }
        }

        stage('Complete') {
            steps {
                echo '🚀 Application passed all stages!'
            }
        }
    }

    post {
        success {
            echo '✅ Jenkins build SUCCESS!'
        }

        failure {
            echo '❌ Jenkins build FAILED!'
        }
    }
}