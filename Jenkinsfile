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

        stage('Run PHP Tests with Code Coverage') {
            steps {
                script {
                    sh """
                        echo "Running PHP Syntax Check..."
                        php -l index.php
                        
                        echo "Running PHPUnit Tests with Code Coverage..."
                        php -d xdebug.mode=coverage ./vendor/bin/phpunit --coverage-clover=coverage.xml
                    """
                }
            }
        }

        stage('Static Code Analysis - SonarQube') {
            steps {
                script {
                    sh """
                        echo "Running SonarQube Analysis..."
                        sonar-scanner \
                          -Dsonar.projectKey=php_project \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=$SONARQUBE_SERVER \
                          -Dsonar.login=$SONARQUBE_TOKEN \
                          -Dsonar.php.coverage.reportPaths=coverage.xml
                    """
                }
            }
        }

        stage('Deploy to Server via SSH') {
            steps {
                script {
                    sshagent(['server-ssh-key']) {
                        sh """
                            echo "Deploying application files..."
                            rsync -avz --delete . ${SSH_USER}@${SSH_HOST}:${DEPLOY_DIR}/
                            ssh ${SSH_USER}@${SSH_HOST} "sudo chown -R www-data:www-data ${DEPLOY_DIR}/ && sudo chmod -R 755 ${DEPLOY_DIR}/ && sudo systemctl restart apache2"
                        """
                    }
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

        stage('Ensure Monitoring is Configured') {
            steps {
                script {
                    sshagent(['server-ssh-key']) {
                        sh """
                            echo "Checking Prometheus Exporter installation..."
                            ssh ${SSH_USER}@${SSH_HOST} "dpkg -s prometheus-php-fpm-exporter >/dev/null 2>&1 || (echo 'Installing Prometheus Exporter...' && sudo apt install prometheus-php-fpm-exporter -y && sudo systemctl enable prometheus-php-fpm-exporter && sudo systemctl start prometheus-php-fpm-exporter)"
                        """
                    }
                }
            }
        }

        stage('Verify Monitoring - Prometheus & Grafana') {
            steps {
                script {
                    sh """
                        echo "Checking Prometheus Monitoring..."
                        curl -s ${PROMETHEUS_URL}/api/v1/targets | jq .data.activeTargets
                        echo "Grafana Dashboards Available at http://your-grafana-server:3000"
                    """
                }
            }
        }

        stage('Ensure Log Forwarding is Configured') {
            steps {
                script {
                    sshagent(['server-ssh-key']) {
                        sh """
                            echo "Checking Filebeat installation..."
                            ssh ${SSH_USER}@${SSH_HOST} "dpkg -s filebeat >/dev/null 2>&1 || (echo 'Installing Filebeat...' && sudo apt install filebeat -y && sudo systemctl enable filebeat && sudo systemctl start filebeat)"
                        """
                    }
                }
            }
        }

        stage('Verify Logs in ELK') {
            steps {
                script {
                    sh """
                        echo "Checking ELK Stack Logs..."
                        curl -s ${ELK_URL}/_cat/indices?v | grep 'filebeat'
                    """
                }
            }
        }

    }

    post {
        success {
            echo "Deployment completed successfully with code coverage, monitoring, and logging enabled!"
        }
        failure {
            echo "Pipeline execution failed!"
        }
    }
}
