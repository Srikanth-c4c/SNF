pipeline {

    agent any

    environment {
        SONARQUBE_SERVER = 'http://onprem-sonarqube.local:9000'
        SONARQUBE_TOKEN = credentials('sonarqube-token')
        GIT_REPO = 'https://github.com/org/repo.git'
        GIT_CREDENTIALS = 'git-credentials'
        SSH_USER = 'user'
        SSH_HOST = 'onprem-server'
        DEPLOY_DIR = '/var/www/html'
        PROMETHEUS_URL = "http://prometheus-server:9090"
        ELK_URL = "http://elk-server:9200"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main', credentialsId: "${GIT_CREDENTIALS}", url: "${GIT_REPO}"
            }
        }

        stage('Run PHP Tests') {
            steps {
                script {
                    sh """
                        echo "Running PHP Lint Check..."
                        php -l index.php
                        echo "Running Unit Tests..."
                        ./vendor/bin/phpunit --testdox
                    """
                }
            }
        }

        stage('Static Code Analysis - SonarQube') {
            steps {
                script {
                    sh 'sonar-scanner -Dsonar.projectKey=php_project -Dsonar.sources=. -Dsonar.host.url=$SONARQUBE_SERVER -Dsonar.login=$SONARQUBE_TOKEN'
                }
            }
        }

        stage('Build & Package') {
            steps {
                script {
                    sh 'zip -r app.zip .'
                }
            }
        }

        stage('Security Scan (DAST) - OWASP ZAP') {
            steps {
                script {
                    sh 'zap-cli quick-scan --self-contained --start-options "-config api.disablekey=true" http://your-app-url'
                }
            }
        }

        stage('Deploy to Server via SSH') {
            steps {
                script {
                    sshagent(['server-ssh-key']) {
                        sh """
                            echo "Deploying application..."
                            scp app.zip ${SSH_USER}@${SSH_HOST}:${DEPLOY_DIR}/
                            ssh ${SSH_USER}@${SSH_HOST} "unzip -o ${DEPLOY_DIR}/app.zip -d ${DEPLOY_DIR}/ && systemctl restart apache2"
                        """
                    }
                }
            }
        }

        stage('Monitoring - Prometheus') {
            steps {
                script {
                    sh "curl -s ${PROMETHEUS_URL} || echo 'Prometheus is not reachable'"
                }
            }
        }

        stage('Log Management - ELK Stack') {
            steps {
                script {
                    sh "curl -s ${ELK_URL}/_cluster/health || echo 'ELK is not reachable'"
                }
            }
        }

    }

    post {
        success {
            echo "Deployment completed successfully!"
        }
        failure {
            echo "Deployment failed!"
        }
    }
}
