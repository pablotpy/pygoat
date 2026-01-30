pipeline {
    agent any

    environment {
        DT_URL = 'http://dependecy-track-dtrack-apiserver-1:8080' 
        DD_URL = 'http://django-defectdojo-nginx-1:8080'
        DD_API_KEY = credentials('dd-api-key')
        DT_API_KEY = credentials('dt-api-key')
        DD_ENGAGEMENT_ID = '4' 
        // Configuración Docker base (Entrypoint vacío para poder usar bash)
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

        stage('SAST - Bandit (Solo Criticas)') {
            steps {
                script {
                    echo "--- Ejecutando Bandit (Filtrado) ---"
                    sh """
                        docker run ${DOCKER_ARGS} python:3.10-slim /bin/bash -c " \
                            pip install bandit && \
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
                    echo "--- Generando SBOM ---"
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

        stage('Secrets - Gitleaks (Nuclear)') {
            steps {
                script {
                    echo "--- Ejecutando Gitleaks v8.18.1 ---"
                    // CAMBIO AQUI: Agregado '-u root:root' antes de la imagen
                    sh """
                        docker run ${DOCKER_ARGS} -u root:root zricethezav/gitleaks:v8.18.1 /bin/bash -c " \
                            git config --global --add safe.directory '*' && \
                            gitleaks detect -v --source . --log-opts='--all' --report-path gitleaks_report.json --exit-code 0 \
                        "
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