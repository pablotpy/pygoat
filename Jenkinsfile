pipeline {
    agent any

    environment {
        DT_URL = 'http://dependecy-track-dtrack-apiserver-1:8080' 
        DD_URL = 'http://django-defectdojo-nginx-1:8080'
        DD_API_KEY = credentials('dd-api-key')
        DT_API_KEY = credentials('dt-api-key')
        DD_ENGAGEMENT_ID = '4' 
        // Mapeamos el workspace actual para que el script sea visible dentro de docker
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

        stage('SCA - Dependency Track (Script File)') {
            steps {
                script {
                    // PASO 1: Creamos el script BASH complejo en un archivo separado
                    // Esto evita errores de sintaxis y comillas en Jenkins
                    def dtScript = """#!/bin/bash
                    set -e
                    
                    echo "--- Instalando herramientas ---"
                    apt-get update -qq && apt-get install -y curl -qq
                    pip install cyclonedx-bom -q
                    
                    echo "--- Generando BOM ---"
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
                    echo "--- BUCLE 1: Obteniendo UUID del Proyecto ---"
                    PROJECT_UUID=""
                    
                    # Intentamos 5 veces obtener el UUID (esperando a que se cree)
                    for i in 1 2 3 4 5; do
                        curl -s -H "X-Api-Key: \$DT_API_KEY" "\$DT_URL/api/v1/project/lookup?name=Pygoat&version=1.0" > dt_project.json
                        
                        # Si el archivo tiene datos y tiene el campo uuid
                        if [ -s dt_project.json ]; then
                            # Python seguro para extraer
                            UUID_EXTRACTED=\$(cat dt_project.json | python3 -c "import sys, json; print(json.load(sys.stdin).get('uuid', ''))" 2>/dev/null)
                            
                            if [ ! -z "\$UUID_EXTRACTED" ]; then
                                PROJECT_UUID=\$UUID_EXTRACTED
                                echo "UUID Encontrado: \$PROJECT_UUID"
                                break
                            fi
                        fi
                        echo "Intento \$i: Proyecto no listo. Esperando 5s..."
                        sleep 5
                    done
                    
                    if [ -z "\$PROJECT_UUID" ]; then
                        echo "ERROR: No se pudo obtener el UUID."
                        exit 1
                    fi
                    
                    echo "--- BUCLE 2: Esperando Findings (Analysis) ---"
                    SUCCESS=0
                    # 20 intentos de 10 segundos = 200 segundos max
                    for i in \$(seq 1 20); do
                        echo "Intento \$i de 20: Descargando..."
                        
                        curl -s -H "X-Api-Key: \$DT_API_KEY" \
                            "\$DT_URL/api/v1/finding/project/\$PROJECT_UUID?suppressed=false" \
                            -o dt_findings.json
                        
                        # Verificamos tamano (>10 bytes significa que hay datos JSON reales, no [])
                        SIZE=\$(wc -c < dt_findings.json)
                        echo "Tamaño recibido: \$SIZE bytes"
                        
                        if [ "\$SIZE" -gt 10 ]; then
                            echo "¡Datos recibidos!"
                            SUCCESS=1
                            break
                        else
                            echo "Analisis incompleto. Esperando 10s..."
                            sleep 10
                        fi
                    done
                    
                    if [ \$SUCCESS -eq 0 ]; then
                        echo "WARNING: Timeout agotado. Creando archivo vacio para no romper pipeline."
                        echo '[]' > dt_findings.json
                    fi
                    
                    ls -lh dt_findings.json
                    """
                    
                    // Escribimos el archivo en el workspace
                    writeFile file: 'run_dt_analysis.sh', text: dtScript
                    
                    // Damos permisos de ejecución
                    sh "chmod +x run_dt_analysis.sh"
                    
                    echo "--- Ejecutando Script dentro de Docker ---"
                    // Pasamos las variables de entorno con -e para que el script las vea
                    sh """
                        docker run ${DOCKER_ARGS} \
                            -e DT_URL='${DT_URL}' \
                            -e DT_API_KEY='${DT_API_KEY}' \
                            python:3.10-slim /bin/bash ./run_dt_analysis.sh
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
                                -F 'scan_type=Dependency Track Finding Packaging Format (FPF)' \
                                -F 'engagement=${DD_ENGAGEMENT_ID}' \
                                -F 'file=@dt_findings.json' \
                        "
                    """
                }
            }
        }
    }
}