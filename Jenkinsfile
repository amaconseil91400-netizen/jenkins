pipeline {
    agent any

    environment {
        // --- CONFIGURATION À MODIFIER ---
        HARBOR_URL     = "core.harbor.domain.com"
        HARBOR_PROJECT = "harbor"
        IMAGE_NAME     = "mon-app"
        // --------------------------------
        
        IMAGE_TAG      = "${env.BUILD_NUMBER}"
        FULL_IMAGE_PATH = "${HARBOR_URL}/${HARBOR_PROJECT}/${IMAGE_NAME}:${IMAGE_TAG}"
    }

    stages {
        stage('1. Récupération du Code') {
            steps {
                // Jenkins clone votre dépôt GitHub
                git credentialsId: 'github-creds', 
                    url: 'https://github.com/amaconseil91400-netizen/jenkinsgit',
                    branch: 'main'
            }
        }

        stage('2. Build de l\'image') {
            steps {
                script {
                    // Construction de l'image Docker
                    sh "docker build -t ${FULL_IMAGE_PATH} ."
                }
            }
        }

        stage('3. Scan de sécurité (Trivy)') {
            steps {
                script {
                    // On utilise Docker pour lancer Trivy afin d'éviter de l'installer sur la VM
                    // Cela scanne l'image locale et affiche les failles High et Critical
                    sh """
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                    aquasec/trivy:latest image --severity HIGH,CRITICAL ${FULL_IMAGE_PATH}
                    """
                }
            }
        }

        stage('4. Push vers Harbor') {
            steps {
                // Utilise les identifiants créés dans l'interface Jenkins
                withCredentials([usernamePassword(credentialsId: 'harbor-creds', 
                                                 passwordVariable: 'HARBOR_PWD', 
                                                 usernameVariable: 'HARBOR_USER')]) {
                    sh """
                    echo ${HARBOR_PWD} | docker login ${HARBOR_URL} -u ${HARBOR_USER} --password-stdin
                    docker push ${FULL_IMAGE_PATH}
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Succès : L'image ${FULL_IMAGE_PATH} est sur Harbor !"
        }
        failure {
            echo "❌ Échec : Vérifiez les logs pour comprendre l'erreur."
        }
        always {
            // Supprime l'image locale pour ne pas saturer le disque de la VM Vagrant
            sh "docker rmi ${FULL_IMAGE_PATH} || true"
        }
    }
}
