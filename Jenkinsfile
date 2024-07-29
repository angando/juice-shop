pipeline {
    agent any
    environment {
        SCANNER_HOME = tool 'SonarQubeScanner'
        SONAR_TOKEN = credentials('sonarqube-token')
        DOCKER_HOME = tool 'Docker'
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
                    sh "${DOCKER_HOME}/bin/docker build -t juice-shop ."
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
                    def exitCode = sh(script: "${DOCKER_HOME}/bin/docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --exit-code 1 --severity HIGH juice-shop", returnStatus: true)
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
                    sh "${DOCKER_HOME}/bin/docker run -d --name  juice-shop --network zapnet -p 3000:3000 juice-shop"
                }
            }
        }
        stage('ZAP Scan') {
            steps {
                script {
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                        sh '''
                        ${DOCKER_HOME}/bin/docker run --network zapnet --user root -v $(pwd):/zap/wrk/:rw -t zaproxy/zap-stable zap-baseline.py -t http://juice-shop:3000 -r zap_report.html
                        '''
                    }
                }
            }
        }
        stage('Nikto Scan') {
            steps {
                script {
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                        sh '''
                        ${DOCKER_HOME}/bin/docker run --network zapnet --rm -v $(pwd):/tmp osodevops/nikto:latest -h http://juice-shop:3000 -o /tmp/nikto_report.html
                        '''
                    }
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
