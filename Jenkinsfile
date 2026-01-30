pipeline {
    agent any

    environment {
        DT_URL = 'http://dependecy-track-dtrack-apiserver-1:8080' 
        DD_URL = 'http://django-defectdojo-nginx-1:8080'
        DD_API_KEY = credentials('dd-api-key')
        DT_API_KEY = credentials('dt-api-key')
        DD_ENGAGEMENT_ID = '5' 
        DOCKER_ARGS = '--rm --entrypoint="" --network devsecops-net -v /var/jenkins_home:/var/jenkins_home -w ${WORKSPACE}'
    }

    stages {
        stage('Limpieza') {
            steps { cleanWs() }
        }
        
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/master']], 
                    doGenerateSubmoduleConfigurations: false,
                    extensions: [[$class: 'CloneOption', depth: 0, noTags: false, reference: '', shallow: false]],
                    userRemoteConfigs: [[url: 'https://github.com/pablotpy/pygoat.git']]
                ])
            }
        }

        stage('SAST - Bandit') {
            steps {
                script {
                    echo "--- Ejecutando Bandit ---"
                    sh """
                        docker run ${DOCKER_ARGS} python:3.10-slim /bin/bash -c " \
                            pip install bandit && \
                            bandit -r . -f json -o bandit_report.json || true \
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
                        docker run ${DOCKER_ARGS} -u root:root zricethezav/gitleaks:v8.18.1 /bin/bash -c " \
                            git config --global --add safe.directory '*' && \
                            gitleaks detect -v --source . --log-opts='--all' --report-path gitleaks_report.json --exit-code 0 \
                        "
                    """
                }
            }
        }

        stage('SCA - Generación y Distribución') {
            steps {
                script {
                    echo "--- 1. Generando SBOM Enriquecido con Trivy ---"
                    // Trivy genera un SBOM en formato CycloneDX que INCLUYE las vulnerabilidades detectadas
                    sh """
                        docker run ${DOCKER_ARGS} aquasec/trivy:latest fs . \
                            --format cyclonedx \
                            --output bom_validated.json \
                            --scanners vuln \
                            --exit-code 0
                    """
                    
                    echo "--- 2. Enviando a Dependency-Track (Gestión) ---"
                    // DT recibe el archivo y lo muestra en su web
                    sh """
                        docker run ${DOCKER_ARGS} curlimages/curl:latest -s -X POST '${DT_URL}/api/v1/bom' \
                            -H 'Content-Type: multipart/form-data' \
                            -H 'X-Api-Key: ${DT_API_KEY}' \
                            -F 'autoCreate=true' \
                            -F 'projectName=Pygoat' \
                            -F 'projectVersion=1.0' \
                            -F 'bom=@bom_validated.json'
                    """
                }
            }
        }

        stage('Upload to DefectDojo') {
            steps {
                script {
                    echo "--- Subiendo Reportes a DefectDojo ---"
                    sh "ls -lh bandit_report.json gitleaks_report.json bom_validated.json"
                    
                    sh """
                        docker run ${DOCKER_ARGS} curlimages/curl:latest /bin/sh -c " \
                            # 1. BANDIT
                            curl -v -X POST '${DD_URL}/api/v2/import-scan/' \
                                -H 'Authorization: Token ${DD_API_KEY}' \
                                -H 'Content-Type: multipart/form-data' \
                                -F 'active=true' \
                                -F 'verified=true' \
                                -F 'minimum_severity=High' \
                                -F 'close_old_findings=true' \
                                -F 'scan_type=Bandit Scan' \
                                -F 'engagement=${DD_ENGAGEMENT_ID}' \
                                -F 'file=@bandit_report.json' && \
                            
                            # 2. GITLEAKS
                            curl -v -X POST '${DD_URL}/api/v2/import-scan/' \
                                -H 'Authorization: Token ${DD_API_KEY}' \
                                -H 'Content-Type: multipart/form-data' \
                                -F 'active=true' \
                                -F 'verified=true' \
                                -F 'minimum_severity=High' \
                                -F 'close_old_findings=true' \
                                -F 'scan_type=Gitleaks Scan' \
                                -F 'engagement=${DD_ENGAGEMENT_ID}' \
                                -F 'file=@gitleaks_report.json' && \
                            
                            # 3. SCA (Sincronizado)
                            # Usamos el mismo archivo que enviamos a DT.
                            # Como lo generó Trivy, SI tiene vulnerabilidades dentro.
                            curl -v -X POST '${DD_URL}/api/v2/import-scan/' \
                                -H 'Authorization: Token ${DD_API_KEY}' \
                                -H 'Content-Type: multipart/form-data' \
                                -F 'active=true' \
                                -F 'verified=true' \
                                -F 'minimum_severity=High' \
                                -F 'close_old_findings=true' \
                                -F 'scan_type=CycloneDX Scan' \
                                -F 'engagement=${DD_ENGAGEMENT_ID}' \
                                -F 'file=@bom_validated.json' \
                        "
                    """
                }
            }
        }
    }
}