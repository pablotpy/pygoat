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

        stage('SCA - Dependency Track (Enriched SBOM)') {
            steps {
                script {
                    def dtScript = """#!/bin/bash
                    set -e
                    echo "--- Instalando herramientas ---"
                    apt-get update -qq && apt-get install -y curl -qq
                    pip install cyclonedx-bom -q
                    
                    echo "--- Generando BOM Local ---"
                    cyclonedx-py requirements requirements.txt -o bom_local.json
                    
                    echo "--- Subiendo BOM a DT ---"
                    curl -s -X POST "\$DT_URL/api/v1/bom" \
                        -H "Content-Type: multipart/form-data" \
                        -H "X-Api-Key: \$DT_API_KEY" \
                        -F "autoCreate=true" \
                        -F "projectName=Pygoat" \
                        -F "projectVersion=1.0" \
                        -F "bom=@bom_local.json"
                    
                    echo ""
                    echo "--- BUCLE 1: Obteniendo UUID ---"
                    PROJECT_UUID=""
                    for i in 1 2 3 4 5; do
                        curl -s -H "X-Api-Key: \$DT_API_KEY" "\$DT_URL/api/v1/project/lookup?name=Pygoat&version=1.0" > dt_project.json
                        if [ -s dt_project.json ]; then
                            UUID_EXTRACTED=\$(cat dt_project.json | python3 -c "import sys, json; print(json.load(sys.stdin).get('uuid', ''))" 2>/dev/null)
                            if [ ! -z "\$UUID_EXTRACTED" ]; then
                                PROJECT_UUID=\$UUID_EXTRACTED
                                echo "UUID Encontrado: \$PROJECT_UUID"
                                break
                            fi
                        fi
                        echo "Esperando creación del proyecto..."
                        sleep 5
                    done
                    
                    if [ -z "\$PROJECT_UUID" ]; then echo "ERROR: No UUID"; exit 1; fi
                    
                    echo "--- BUCLE 2: Esperando Análisis ---"
                    SUCCESS=0
                    # Esperamos a que la API de Findings devuelva datos (señal de que acabó)
                    for i in \$(seq 1 20); do
                        echo "Intento \$i: Verificando estado..."
                        curl -s -H "X-Api-Key: \$DT_API_KEY" "\$DT_URL/api/v1/finding/project/\$PROJECT_UUID?suppressed=false" -o check_status.json
                        SIZE=\$(wc -c < check_status.json)
                        
                        if [ "\$SIZE" -gt 10 ]; then
                            echo "¡Análisis completo detectado!"
                            SUCCESS=1
                            break
                        else
                            echo "DT sigue analizando. Esperando 10s..."
                            sleep 10
                        fi
                    done
                    
                    if [ \$SUCCESS -eq 1 ]; then
                        echo "--- Descargando BOM Enriquecido (Con Vulnerabilidades) ---"
                        # CAMBIO CLAVE: Descargamos el BOM completo, no la lista suelta de findings
                        curl -s -H "X-Api-Key: \$DT_API_KEY" \
                            "\$DT_URL/api/v1/bom/cyclonedx/project/\$PROJECT_UUID" \
                            -o bom_enriched.json
                            
                        echo "BOM descargado. Tamaño: \$(wc -c < bom_enriched.json)"
                    else
                        echo "WARNING: Timeout. Usando BOM local (sin vulns) para no romper pipeline."
                        cp bom_local.json bom_enriched.json
                    fi
                    """
                    
                    writeFile file: 'run_dt.sh', text: dtScript
                    sh "chmod +x run_dt.sh"
                    
                    sh """
                        docker run ${DOCKER_ARGS} \
                            -e DT_URL='${DT_URL}' \
                            -e DT_API_KEY='${DT_API_KEY}' \
                            python:3.10-slim /bin/bash ./run_dt.sh
                    """
                }
            }
        }

        stage('Upload to DefectDojo') {
            steps {
                script {
                    echo "--- Subiendo Reportes ---"
                    // Ahora buscamos bom_enriched.json
                    sh "ls -lh bandit_report.json gitleaks_report.json bom_enriched.json"
                    
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
                            
                            # 3. ENRICHED SBOM (Como CycloneDX Scan)
                            # Este es el estándar de oro. DefectDojo leerá las vulnerabilidades
                            # que Dependency-Track escribió dentro de este JSON.
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