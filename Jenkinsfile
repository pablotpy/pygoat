pipeline {
    agent any
    environment {
        DD_URL = 'http://django-defectdojo-nginx-1:8080'
        DD_API_KEY = credentials('dd-api-key')
    }
    stages {
        stage('Check Names') {
            steps {
                script {
                    // Pide a DefectDojo la lista de Scan Types validos
                    sh """
                        docker run --rm curlimages/curl:latest \
                            -H "Authorization: Token ${DD_API_KEY}" \
                            "${DD_URL}/api/v2/scan_types/?limit=100" > types.json
                    """
                    // Muestra los nombres que contengan "Track"
                    sh "cat types.json | grep -i 'Track'"
                }
            }
        }
    }
}