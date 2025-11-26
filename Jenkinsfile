pipeline {
    agent any

    environment {
        PROJECT_NAME = "pipeline-test"
        SONARQUBE_URL = "http://sonarqube:9000"
        SONARQUBE_TOKEN = "squ_32ab2e416d859f825c47d48892ae63d40ba60be6"
        TARGET_URL = "http://172.23.202.60:5000"
    }

    stages {

        stage('Install Python') {
            steps {
                sh '''
                    apt update
                    apt install -y python3 python3-venv python3-pip
                '''
            }
        }

        stage('Setup Environment') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                '''
            }
        }

        stage('Start Vulnerable Server') {
            steps {
                sh '''
                    . venv/bin/activate
                    nohup python3 vulnerable_server.py > server.log 2>&1 &
                    sleep 5
                '''
            }
        }

        stage('Python Security Audit') {
            steps {
                sh '''
                    . venv/bin/activate
                    pip install pip-audit
                    mkdir -p dependency-check-report
                    pip-audit -r requirements.txt -f markdown -o dependency-check-report/pip-audit.md || true
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarQubeScanner'
                    withSonarQubeEnv('SonarQubeScanner') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                                -Dsonar.projectKey=$PROJECT_NAME \
                                -Dsonar.sources=. \
                                -Dsonar.host.url=$SONARQUBE_URL \
                                -Dsonar.login=$SONARQUBE_TOKEN
                        """
                    }
                }
            }
        }

        stage('Dependency Check') {
            environment {
                NVD_API_KEY = credentials('nvdApiKey')
            }
            steps {
                dependencyCheck additionalArguments: "--scan . --format HTML --out dependency-check-report --enableExperimental --enableRetired --nvdApiKey ${NVD_API_KEY}", 
                                 odcInstallation: 'DependencyCheck'
            }
        }

        stage('OWASP ZAP Scan') {
    steps {
        sh '''
            mkdir -p zap-report

            docker pull owasp/zap2docker-stable

            docker run \
                --network=host \
                -v $(pwd)/zap-report:/zap/wrk \
                owasp/zap2docker-stable \
                zap-baseline.py \
                -t http://172.23.202.60:5000 \
                -r zap_report.html
        '''
    }
    post {
        success {
            archiveArtifacts artifacts: 'zap-report/zap_report.html', fingerprint: true
        }
    }
}


        stage('Publish Reports') {
            steps {
                publishHTML([
                    allowMissing: true,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'dependency-check-report',
                    reportFiles: 'dependency-check-report.html',
                    reportName: 'OWASP Dependency Check Report'
                ])

                publishHTML([
                    allowMissing: true,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'zap-report',
                    reportFiles: 'zap_report.html',
                    reportName: 'OWASP ZAP Scan Report'
                ])
            }
        }
    }
}
