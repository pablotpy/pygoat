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

        stage('SCA - Dependency Track (Doble Verificación)') {
            steps {
                script {
                    echo "--- 1. Subiendo Inventario a DT ---"
                    sh """
                        docker run ${DOCKER_ARGS} python:3.10-slim /bin/bash -c " \
                            apt-get update && apt-get install -y curl && \
                            pip install cyclonedx-bom && \
                            
                            cyclonedx-py requirements requirements.txt -o bom_local.json && \
                            
                            echo 'Subiendo BOM...' && \
                            curl -X POST '${DT_URL}/api/v1/bom' \
                                -H 'Content-Type: multipart/form-data' \
                                -H 'X-Api-Key: ${DT_API_KEY}' \
                                -F 'autoCreate=true' \
                                -F 'projectName=Pygoat' \
                                -F 'projectVersion=1.0' \
                                -F 'bom=@bom_local.json' && \
                                
                            echo '--- 2. BUCLE 1: Obteniendo UUID (Esperando creación) ---' && \
                            PROJECT_UUID=\"\"; \
                            for i in 1 2 3 4 5; do \
                                echo \"Intento \$i para obtener UUID...\"; \
                                curl -s -H 'X-Api-Key: ${DT_API_KEY}' '${DT_URL}/api/v1/project/lookup?name=Pygoat&version=1.0' > dt_project.json; \
                                \
                                # Validamos si el archivo tiene algo antes de pasarlo a Python \
                                if [ -s dt_project.json ]; then \
                                    # Intentamos extraer UUID de forma segura (sin que rompa el script) \
                                    EXTRACTED=\$(cat dt_project.json | python3 -c \"import sys, json; print(json.load(sys.stdin).get('uuid', ''))\" 2>/dev/null); \
                                    if [ ! -z \"\$EXTRACTED\" ]; then \
                                        PROJECT_UUID=\$EXTRACTED; \
                                        echo \"¡UUID Encontrado: \$PROJECT_UUID!\"; \
                                        break; \
                                    fi; \
                                fi; \
                                echo \"Proyecto no listo aún. Esperando 5s...\"; \
                                sleep 5; \
                            done; \
                            \
                            if [ -z \"\$PROJECT_UUID\" ]; then \
                                echo \"ERROR FATAL: No se pudo obtener el UUID después de 5 intentos.\"; \
                                cat dt_project.json; \
                                exit 1; \
                            fi && \
                            \
                            echo '--- 3. BUCLE 2: Descargando Findings (Esperando análisis) ---' && \
                            SUCCESS=0; \
                            for i in \$(seq 1 20); do \
                                echo \"Intento \$i de 20: Consultando findings...\"; \
                                \
                                curl -s -H 'X-Api-Key: ${DT_API_KEY}' \
                                    '${DT_URL}/api/v1/finding/project/\$PROJECT_UUID?suppressed=false' \
                                    -o dt_findings.json; \
                                \
                                # Logica de Bytes: Si pesa mas de 10 bytes, tiene datos \
                                FILE_SIZE=\$(wc -c < dt_findings.json); \
                                echo \"Tamaño recibido: \$FILE_SIZE bytes\"; \
                                \
                                if [ \"\$FILE_SIZE\" -gt 10 ]; then \
                                    echo \"¡EXITO! Datos recibidos.\"; \
                                    SUCCESS=1; \
                                    break; \
                                else \
                                    echo \"Analisis en curso (recibido vacio). Esperando 15s...\"; \
                                    sleep 15; \
                                fi \
                            done; \
                            \
                            if [ \$SUCCESS -eq 0 ]; then \
                                echo \"WARNING: Timeout en analisis. Usando archivo vacio.\"; \
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