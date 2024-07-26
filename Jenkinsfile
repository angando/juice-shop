pipeline {
    agent any
    environment {
        SCANNER_HOME = tool 'SonarQubeScanner'
        SONAR_TOKEN = credentials('sonarqube-token')
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
                withSonarQubeEnv('SonarQube') {
                    sh '''
                    ${SCANNER_HOME}/bin/sonar-scanner \
                    -Dsonar.projectKey=juice-shop \
                    -Dsonar.sources=. \
                    -Dsonar.host.url=http://sonarqube:9000 \
                    -Dsonar.login=${SONAR_TOKEN}
                    '''
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
                script {
                    def exitCode = sh(script: 'docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --exit-code 1 --severity HIGH juice-shop', returnStatus: true)
                    if (exitCode != 0) {
                        echo "Trivy scan found vulnerabilities with HIGH severity"
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }
        stage('Deploy') {
            steps {
                script {
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                        docker.run('juice-shop', '-d -p 3000:3000')
                    }
                }
            }
        }
        stage('ZAP Scan') {
            steps {
                script {
                    zap(
                        zapHome: '/var/lib/jenkins/tools/owasp-zap',
                        target: 'http://localhost:3000',
                        failOnError: false,
                        reportTitle: 'ZAP Scan Report',
                        reportFilename: 'zap_report.html'
                    )
                }
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}
