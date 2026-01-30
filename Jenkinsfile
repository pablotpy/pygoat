pipeline {
    agent any

    environment {
        DT_URL = 'http://dependecy-track-dtrack-apiserver-1:8080' 
        DD_URL = 'http://django-defectdojo-nginx-1:8080'
        DD_API_KEY = credentials('dd-api-key')
        DT_API_KEY = credentials('dt-api-key')
        DD_ENGAGEMENT_ID = '4' 
        DOCKER_ARGS = '--rm --entrypoint="" --network devsecops-net -v /var/jenkins_home:/var/jenkins_home -w ${WORKSPACE}'
    }

    stages {
        stage('Limpieza') {
            steps { cleanWs() }
        }
        
        stage('Checkout (Full Master)') {
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

        stage('SAST - Bandit (Solo Criticas/Altas)') {
            steps {
                script {
                    echo "--- Ejecutando Bandit ---"
                    // Nota: Quitamos el filtro estricto AQUI para dejar que DefectDojo filtre después.
                    // Generamos un reporte completo para no perder datos antes de tiempo.
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

        stage('SCA - Dependency Track (Flujo Completo)') {
            steps {
                script {
                    echo "--- 1. Subiendo Inventario a DT ---"
                    sh """
                        docker run ${DOCKER_ARGS} python:3.10-slim /bin/bash -c " \
                            apt-get update && apt-get install -y curl && \
                            pip install cyclonedx-bom && \
                            
                            # Generar BOM
                            cyclonedx-py requirements requirements.txt -o bom_local.json && \
                            
                            # Subir a DT
                            curl -X POST '${DT_URL}/api/v1/bom' \
                                -H 'Content-Type: multipart/form-data' \
                                -H 'X-Api-Key: ${DT_API_KEY}' \
                                -F 'autoCreate=true' \
                                -F 'projectName=Pygoat' \
                                -F 'projectVersion=1.0' \
                                -F 'bom=@bom_local.json' && \
                                
                            echo '--- 2. Esperando analisis (30s) ---' && \
                            sleep 30 && \
                            
                            echo '--- 3. Obteniendo UUID ---' && \
                            # Obtenemos el UUID e imprimimos para depurar
                            curl -s -H 'X-Api-Key: ${DT_API_KEY}' '${DT_URL}/api/v1/project/lookup?name=Pygoat&version=1.0' > dt_project.json && \
                            cat dt_project.json && \
                            
                            # Extraemos UUID con python
                            PROJECT_UUID=\$(cat dt_project.json | python3 -c \"import sys, json; print(json.load(sys.stdin)['uuid'])\") && \
                            echo \"UUID es: \$PROJECT_UUID\" && \
                            
                            echo '--- 4. Descargando Reporte Enriquecido ---' && \
                            # Descargamos el JSON final
                            curl -s -H 'X-Api-Key: ${DT_API_KEY}' \
                                '${DT_URL}/api/v1/bom/cyclonedx/project/\$PROJECT_UUID' \
                                -o bom_enriched.json && \
                            
                            # Verificamos si se descargó bien (si pesa 0 bytes, fallará luego)
                            ls -lh bom_enriched.json \
                        "
                    """
                }
            }
        }

        stage('Upload to DefectDojo') {
            steps {
                script {
                    echo "--- Subiendo Reportes (Filtro: High & Critical) ---"
                    // Verificamos existencia de archivos antes de subir
                    sh "ls -lh bandit_report.json gitleaks_report.json bom_enriched.json"
                    
                    sh """
                        docker run ${DOCKER_ARGS} curlimages/curl:latest /bin/sh -c " \
                            # 1. BANDIT (Filtro High)
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
                            
                            # 2. GITLEAKS (Filtro High)
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
                            
                            # 3. SBOM ENRIQUECIDO (Filtro High)
                            curl -v -X POST '${DD_URL}/api/v2/import-scan/' \
                                -H 'Authorization: Token ${DD_API_KEY}' \
                                -H 'Content-Type: multipart/form-data' \
                                -F 'active=true' \
                                -F 'verified=true' \
                                -F 'minimum_severity=High' \
                                -F 'close_old_findings=true' \
                                -F 'scan_type=CycloneDX Scan' \
                                -F 'engagement=${DD_ENGAGEMENT_ID}' \
                                -F 'file=@bom_enriched.json' \
                        "
                    """
                }
            }
        }
    }
}