pipeline {
    agent any

    environment {
        PROJECT_NAME = "pipeline-test"
        SONARQUBE_URL = "http://localhost:9000"
        SONARQUBE_TOKEN = credentials('sonar-token')
        TARGET_URL = "http://172.23.41.49:5000"
    }

    stages {

        stage('Setup Python on Windows') {
            steps {
                bat """
                    python --version
                    py -3 -m venv venv
                    venv\\Scripts\\pip install --upgrade pip
                    venv\\Scripts\\pip install -r requirements.txt
                """
            }
        }

        stage('Python Security Audit') {
            steps {
                bat """
                    venv\\Scripts\\pip install pip-audit
                    if not exist dependency-check-report mkdir dependency-check-report
                    venv\\Scripts\\pip-audit -r requirements.txt -f markdown -o dependency-check-report\\pip-audit.md || exit /b 0
                """
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarQubeScanner'
                    withSonarQubeEnv('SonarQubeScanner') {
                        bat """
                            "${scannerHome}\\bin\\sonar-scanner.bat" ^
                                -Dsonar.projectKey=%PROJECT_NAME% ^
                                -Dsonar.sources=. ^
                                -Dsonar.host.url=%SONARQUBE_URL% ^
                                -Dsonar.login=%SONARQUBE_TOKEN%
                        """
                    }
                }
            }
        }

        stage('Dependency Check') {
        steps {
            // Usamos withCredentials para obtener el valor del Texto Secreto
            withCredentials([string(credentialsId: 'nvdApiKey', variable: 'NVD_API_KEY')]) {
                
                // Ejecutamos el step dependencyCheck
                dependencyCheck additionalArguments: "--scan . --format HTML --out dependency-check-report --enableExperimental --enableRetired --nvdApiKey ${env.NVD_API_KEY} --nvdApiDelay 3500", 
                                 odcInstallation: 'DependencyCheck'
            }
        }
    }

        stage('Publish Reports') {
            steps {
                publishHTML([
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'dependency-check-report',
                    reportFiles: 'dependency-check-report.html',
                    reportName: 'OWASP Dependency Check Report'
                ])
            }
        }
    }
}
