pipeline {
    agent any

    environment {
        PROJECT_NAME = "pipeline-test"
        SONARQUBE_URL = "http://localhost:9000"
        // Se usa la función credentials() para obtener la credencial y su ID
        // Luego se inyecta la clave como variable de entorno
        SONARQUBE_TOKEN = credentials('sonar-token') 
        TARGET_URL = "http://172.23.41.49:5000"
    }

    stages {
        // --- Etapa 1: Configuración de Python ---
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
        
        // --- Etapa 2: Auditoría de Seguridad (pip-audit) ---
        stage('Python Security Audit') {
            steps {
                bat """
                    venv\\Scripts\\pip install pip-audit
                    if not exist dependency-check-report mkdir dependency-check-report
                    venv\\Scripts\\pip-audit -r requirements.txt -f markdown -o dependency-check-report\\pip-audit.md || exit /b 0
                """
            }
        }
        
        // --- Etapa 3: Análisis Estático (SonarQube) ---
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
                                // Nota: %SONARQUBE_TOKEN% se inyecta de forma segura por withSonarQubeEnv
                                -Dsonar.login=%SONARQUBE_TOKEN%
                        """
                    }
                }
            }
        }

        // --- Etapa 4: Análisis de Composición de Software (Dependency Check) ---
        // ¡Esta es la etapa corregida!
        stage('Dependency Check') {
            steps {
                // Inyecta la clave API de forma segura
                withCredentials([string(credentialsId: 'NVD_API_KEY_SECRET', variable: 'NVD_API_KEY')]) {
                    
                    dependencyCheck 
                        // Se añade --disableAssemblyAnalyzer (soluciona el error de .NET)
                        additionalArguments: "--scan . --format HTML --out dependency-check-report --disableAssemblyAnalyzer --enableExperimental --enableRetired --nvdApiDelay 3500", 
                        odcInstallation: 'DependencyCheck',
                        // Se usa el argumento dedicado 'nvdApiKey' (soluciona la advertencia de seguridad)
                        nvdApiKey: env.NVD_API_KEY 
                }
            }
        }

        // --- Etapa 5: Publicación de Informes ---
        stage('Publish Reports') {
            steps {
                publishHTML([
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    // Debe usar la misma carpeta y nombre de archivo generados por Dependency-Check
                    reportDir: 'dependency-check-report',
                    reportFiles: 'dependency-check-report.html',
                    reportName: 'OWASP Dependency Check Report'
                ])
            }
        }
    }
}
