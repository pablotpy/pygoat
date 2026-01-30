pipeline {
    agent any

    environment {
        // Rutas de red internas
        DT_URL = 'http://dependecy-track-dtrack-apiserver-1:8080' 
        DD_URL = 'http://django-defectdojo-nginx-1:8080'
        
        // Credenciales
        DD_API_KEY = credentials('dd-api-key')
        DT_API_KEY = credentials('dt-api-key')
        DD_ENGAGEMENT_ID = '1' 
        
        // Configuración Docker
        DOCKER_ARGS = '--rm --network devsecops-net -v /var/jenkins_home:/var/jenkins_home -w ${WORKSPACE}'
    }

    stages {
        stage('Limpieza') {
            steps {
                cleanWs()
            }
        }
        
        stage('Checkout') {
            steps {
                git branch: 'desarrollo', url: 'https://github.com/pablotpy/pygoat.git'
            }
        }

        stage('SAST - Bandit') {
            steps {
                script {
                    echo "--- Ejecutando Bandit ---"
                    sh """
                        docker run ${DOCKER_ARGS} python:3.10-slim /bin/bash -c " \
                            pip install bandit && \
                            echo 'Iniciando escaneo...' && \
                            bandit -r . -f json -o bandit_report.json || true && \
                            ls -lh bandit_report.json \
                        "
                    """
                }
            }
        }

        stage('SCA - Dependency Track') {
            steps {
                script {
                    echo "--- Generando SBOM (XML) ---"
                    // CORRECCIÓN: Banderas (-o, --format) van ANTES del archivo requirements.txt
                    sh """
                        docker run ${DOCKER_ARGS} python:3.10-slim /bin/bash -c " \
                            apt-get update && apt-get install -y curl && \
                            pip install cyclonedx-bom && \
                            cyclonedx-py requirements --format xml --output bom.xml requirements.txt && \
                            echo 'Verificando archivo BOM:' && \
                            ls -lh bom.xml && \
                            curl -v -X POST '${DT_URL}/api/v1/bom' \
                                -H 'Content-Type: multipart/form-data' \
                                -H 'X-Api-Key: ${DT_API_KEY}' \
                                -F 'autoCreate=true' \
                                -F 'projectName=Pygoat' \
                                -F 'projectVersion=1.0' \
                                -F 'bom=@bom.xml' \
                        "
                    """
                }
            }
        }

        stage('Secrets - Gitleaks') {
            steps {
                script {
                    echo "--- Ejecutando Gitleaks ---"
                    sh """
                        docker run ${DOCKER_ARGS} zricethezav/gitleaks:latest \
                        detect -v --source . --report-path gitleaks_report.json --exit-code 0
                    """
                }
            }
        }

        stage('Upload to DefectDojo') {
            steps {
                script {
                    echo "--- Subiendo a DefectDojo ---"
                    // Verifica que los archivos existen antes de subir
                    sh "ls -lh bandit_report.json gitleaks_report.json"
                    
                    sh """
                        docker run ${DOCKER_ARGS} curlimages/curl:latest /bin/sh -c " \
                            curl -v -X POST '${DD_URL}/api/v2/import-scan/' \
                                -H 'Authorization: Token ${DD_API_KEY}' \
                                -H 'Content-Type: multipart/form-data' \
                                -F 'active=true' \
                                -F 'verified=true' \
                                -F 'scan_type=Bandit' \
                                -F 'engagement=${DD_ENGAGEMENT_ID}' \
                                -F 'file=@bandit_report.json' && \
                            curl -v -X POST '${DD_URL}/api/v2/import-scan/' \
                                -H 'Authorization: Token ${DD_API_KEY}' \
                                -H 'Content-Type: multipart/form-data' \
                                -F 'active=true' \
                                -F 'verified=true' \
                                -F 'scan_type=Gitleaks Scan' \
                                -F 'engagement=${DD_ENGAGEMENT_ID}' \
                                -F 'file=@gitleaks_report.json' \
                        "
                    """
                }
            }
        }
    }
}