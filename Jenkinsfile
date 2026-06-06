pipeline {
    agent {
        label 'quickcart-agent'
    }

    environment {
        APP_NAME = 'quickcart-api'
        VERSION = '1.0.0'
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out QuickCart source code...'
                git branch: 'main',
                    url: 'https://github.com/Mexcelcloud/codealpha-gradle-build.git'
            }
        }

        stage('Build') {
            steps {
                echo 'Building QuickCart API with Gradle...'
                sh 'gradle clean build -x test'
            }
        }

        stage('Test') {
            steps {
                echo 'Running automated tests...'
                sh 'gradle test'
            }
            post {
                always {
                    junit 'build/reports/tests/test/*.xml'
                }
            }
        }

        stage('Package') {
            steps {
                echo 'Verifying deployable artifact...'
                sh 'ls -lh build/libs/'
            }
        }

    }

    post {
        success {
            echo "BUILD SUCCESSFUL — ${APP_NAME}-${VERSION}.jar is ready for deployment"
        }
        failure {
            echo "BUILD FAILED — Check the logs above for errors"
        }
    }
}