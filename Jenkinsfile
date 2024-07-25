pipeline {
    agent any
    environment {
        SCANNER_HOME = tool 'SonarQubeScanner'
    }
    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/angando/juice-shop.git'
            }
        }
        stage('Build') {
            steps {
                script {
                    docker.build('juice-shop')
                }
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQubeScanner') {
                    sh '${SCANNER_HOME}/bin/sonar-scanner -Dsonar.host.url=http://sonarqube:9000'
                }
            }
        }
        stage('Dependency-Check') {
            steps {
                dependencyCheck additionalArguments: '--format XML --format HTML', odcInstallation: 'Default'
            }
        }
        stage('Trivy Scan') {
            steps {
                sh 'docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --exit-code 1 --severity HIGH juice-shop'
            }
        }
        stage('Deploy') {
            steps {
                script {
                    docker.run('juice-shop', '-d -p 3000:3000')
                }
            }
        }
        stage('ZAP Scan') {
            steps {
                sh 'docker run --network devsecops -v $(pwd):/zap/wrk/:rw -t zaproxy/zap-stable zap-baseline.py -t http://juice-shop:3000 -r zap_report.html'
            }
        }
    }
}
