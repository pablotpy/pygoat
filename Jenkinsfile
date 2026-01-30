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

        stage('SCA - Dependency Track (Con Reintentos)') {
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
                            echo 'Subiendo BOM...' && \
                            curl -X POST '${DT_URL}/api/v1/bom' \
                                -H 'Content-Type: multipart/form-data' \
                                -H 'X-Api-Key: ${DT_API_KEY}' \
                                -F 'autoCreate=true' \
                                -F 'projectName=Pygoat' \
                                -F 'projectVersion=1.0' \
                                -F 'bom=@bom_local.json' && \
                                
                            echo '--- 2. Esperando procesamiento inicial (10s) ---' && \
                            sleep 10 && \
                            
                            echo '--- 3. Obteniendo UUID ---' && \
                            curl -s -H 'X-Api-Key: ${DT_API_KEY}' '${DT_URL}/api/v1/project/lookup?name=Pygoat&version=1.0' > dt_project.json && \
                            PROJECT_UUID=\$(cat dt_project.json | python3 -c \"import sys, json; print(json.load(sys.stdin)['uuid'])\") && \
                            echo \"UUID Detectado: \$PROJECT_UUID\" && \
                            
                            echo '--- 4. Bucle de Espera (Polling) para Findings ---' && \
                            # Intentamos descargar hasta 10 veces (esperando 15s entre cada vez = 2.5 min max)
                            SUCCESS=0; \
                            for i in 1 2 3 4 5 6 7 8 9 10; do \
                                echo \"Intento \$i: Descargando findings...\" && \
                                curl -s -H 'X-Api-Key: ${DT_API_KEY}' \
                                    '${DT_URL}/api/v1/finding/project/\$PROJECT_UUID?suppressed=false' \
                                    -o dt_findings.json; \
                                \
                                # Verificamos si el archivo tiene tamaño mayor a 0 y es un JSON válido (empieza con corchete)
                                if [ -s dt_findings.json ] && head -n 1 dt_findings.json | grep -q '\\['; then \
                                    echo \"¡Findings descargados correctamente!\"; \
                                    SUCCESS=1; \
                                    break; \
                                else \
                                    echo \"Aún no hay resultados o DT está procesando. Esperando 15s...\"; \
                                    sleep 15; \
                                fi \
                            done; \
                            \
                            if [ \$SUCCESS -eq 0 ]; then \
                                echo \"ERROR: Tiempo de espera agotado. DT no respondió con findings válidos.\"; \
                                # Creamos un archivo vacio valido [] para que no rompa el siguiente paso, \
                                # pero avisamos del error \
                                echo '[]' > dt_findings.json; \
                            fi && \
                            \
                            ls -lh dt_findings.json \
                        "
                    """
                }
            }
        }

        stage('Upload to DefectDojo') {
            steps {
                script {
                    echo "--- Subiendo Reportes ---"
                    // Verificamos existencia
                    sh "ls -lh bandit_report.json gitleaks_report.json dt_findings.json"
                    
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
                            
                            # 3. DT FINDINGS
                            curl -v -X POST '${DD_URL}/api/v2/import-scan/' \
                                -H 'Authorization: Token ${DD_API_KEY}' \
                                -H 'Content-Type: multipart/form-data' \
                                -F 'active=true' \
                                -F 'verified=true' \
                                -F 'minimum_severity=High' \
                                -F 'close_old_findings=true' \
                                -F 'scan_type=Dependency Track' \
                                -F 'engagement=${DD_ENGAGEMENT_ID}' \
                                -F 'file=@dt_findings.json' \
                        "
                    """
                }
            }
        }
    }
}