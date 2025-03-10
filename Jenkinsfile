pipeline {

    agent any

    environment {

        SONARQUBE_SERVER = 'http://onprem-sonarqube.local:9000'

        SONARQUBE_TOKEN = credentials('sonarqube-token')

    }

    stages {

        stage('Checkout Code') {

            steps {

                git branch: 'main', credentialsId: 'git-credentials', url: 'https://github.com/org/repo.git'

            }

        }
 
        stage('Static Code Analysis') {

            steps {

                script {

                    sh 'sonar-scanner -Dsonar.projectKey=php_project -Dsonar.sources=. -Dsonar.host.url=$SONARQUBE_SERVER -Dsonar.login=$SONARQUBE_TOKEN'

                }

            }

        }
 
        stage('Security Scan (DAST)') {

            steps {

                script {

                    sh 'zap-cli quick-scan --self-contained --start-options "-config api.disablekey=true" http://your-app-url'

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
 
        stage('Deploy to Server via SSH') {

            steps {

                script {

                    sshagent(['server-ssh-key']) {

                        sh 'scp app.zip user@onprem-server:/var/www/html/'

                        sh 'ssh user@onprem-server "unzip -o /var/www/html/app.zip -d /var/www/html/ && systemctl restart apache2"'

                    }

                }

            }

        }

    }

}

 
