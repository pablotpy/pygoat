pipeline {
    agent any

    environment {
        DT_URL = 'http://dependecy-track-dtrack-apiserver-1:8080' 
        DD_URL = 'http://django-defectdojo-nginx-1:8080'
        DD_API_KEY = credentials('dd-api-key')
        DT_API_KEY = credentials('dt-api-key')
        // Cambia el ID si quieres separar los hallazgos de master
        DD_ENGAGEMENT_ID = '4' 
        DOCKER_ARGS = '--rm --network devsecops-net -v /var/jenkins_home:/var/jenkins_home -w ${WORKSPACE}'
    }

    stages {
        stage('Limpieza') {
            steps { cleanWs() }
        }
        
        stage('Checkout (Forzando Master)') {
            steps {
                //Aquí le decimos que descargue MASTER (o main) 
                // aunque el Jenkinsfile venga de 'desarrollo'.
                checkout([
                    $class: 'GitSCM',
                    // CAMBIO: Apuntamos a master para buscar secretos antiguos
                    branches: [[name: '*/master']], 
                    doGenerateSubmoduleConfigurations: false,
                    extensions: [[$class: 'CloneOption', depth: 0, noTags: false, reference: '', shallow: false]],
                    userRemoteConfigs: [[url: 'https://github.com/pablotpy/pygoat.git']]
                ])
                
                script {
                    echo "--- Verificando en qué rama estamos ---"
                    sh "git branch -a"
                    sh "git log -1" // Para confirmar que es el último commit de master
                }
            }
        }

        stage('SAST - Bandit (Solo Críticas)') {
            steps {
                script {
                    echo "--- Ejecutando Bandit (Filtro CRÍTICO) ---"
                    // Bandit encontrará menos cosas (solo lo grave)
                    sh """
                        docker run ${DOCKER_ARGS} python:3.10-slim /bin/bash -c " \
                            pip install bandit && \
                            # FILTRO APLICADO: -lll (High Sev) -iii (High Conf)
                            bandit -r . -lll -iii -f json -o bandit_report.json || true && \
                            ls -lh bandit_report.json \
                        "
                    """
                }
            }
        }

        stage('SCA - Dependency Track') {
            steps {
                script {
                    // Nota: Si requirements.txt es distinto en master, DT mostrará cosas distintas
                    sh """
                        docker run ${DOCKER_ARGS} python:3.10-slim /bin/bash -c " \
                            apt-get update && apt-get install -y curl && \
                            pip install cyclonedx-bom && \
                            cyclonedx-py requirements requirements.txt -o bom.json && \
                            curl -v -X POST '${DT_URL}/api/v1/bom' \
                                -H 'Content-Type: multipart/form-data' \
                                -H 'X-Api-Key: ${DT_API_KEY}' \
                                -F 'autoCreate=true' \
                                -F 'projectName=Pygoat' \
                                -F 'projectVersion=1.0' \
                                -F 'bom=@bom.json' \
                        "
                    """
                }
            }
        }

        stage('Secrets - Gitleaks (Full History)') {
            steps {
                script {
                    echo "--- Buscando Secretos en Historial de Master ---"
                    // Al haber bajado 'master' con historial completo (depth:0),
                    // Gitleaks encontrará todo lo que haya pasado en esa rama.
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
                    echo "--- Subiendo Reportes ---"
                    sh """
                        docker run ${DOCKER_ARGS} curlimages/curl:latest /bin/sh -c " \
                            curl -v -X POST '${DD_URL}/api/v2/import-scan/' \
                                -H 'Authorization: Token ${DD_API_KEY}' \
                                -H 'Content-Type: multipart/form-data' \
                                -F 'active=true' \
                                -F 'verified=true' \
                                -F 'close_old_findings=true' \
                                -F 'scan_type=Bandit Scan' \
                                -F 'engagement=${DD_ENGAGEMENT_ID}' \
                                -F 'file=@bandit_report.json' && \
                            curl -v -X POST '${DD_URL}/api/v2/import-scan/' \
                                -H 'Authorization: Token ${DD_API_KEY}' \
                                -H 'Content-Type: multipart/form-data' \
                                -F 'active=true' \
                                -F 'verified=true' \
                                -F 'close_old_findings=true' \
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