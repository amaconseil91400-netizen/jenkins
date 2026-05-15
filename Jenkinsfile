pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    jenkins: slave
spec:
  containers:
  - name: docker
    image: docker:24.0.6-dind
    command:
    - cat
    tty: true
    volumeMounts:
    - mountPath: /var/run/docker.sock
      name: docker-sock
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
"""
        }
    }

    environment {
        // --- CONFIGURATION CORRIGÉE (Ajout du port 80) ---
        HARBOR_URL      = "my-harbor-core.harbor.svc.cluster.local:80"
        HARBOR_PROJECT  = "harbor"
        IMAGE_NAME      = "mon-app"
        // --------------------------------

        IMAGE_TAG       = "${env.BUILD_NUMBER}"
        FULL_IMAGE_PATH = "${HARBOR_URL}/${HARBOR_PROJECT}/${IMAGE_NAME}:${IMAGE_TAG}"
    }

    stages {
        stage('2. Build de l\'image') {
            steps {
                container('docker') {
                    sh "docker build -t ${FULL_IMAGE_PATH} ."
                }
            }
        }

        stage('3. Scan de sécurité (Trivy)') {
            steps {
                container('docker') {
                    script {
                        sh """
                        docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy:latest image --severity HIGH,CRITICAL ${FULL_IMAGE_PATH}
                        """
                    }
                }
            }
        }

        stage('4. Push vers Harbor') {
            steps {
                container('docker') {
                    withCredentials([usernamePassword(credentialsId: 'harbor-creds',
                                                     passwordVariable: 'HARBOR_PWD',
                                                     usernameVariable: 'HARBOR_USER')]) {
                        sh """
                        # Connexion en spécifiant que ce registry est HTTP (insecure)
                        echo "${HARBOR_PWD}" | docker login ${HARBOR_URL} -u ${HARBOR_USER} --password-stdin
                        docker push ${FULL_IMAGE_PATH}
                        """
                    }
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
            container('docker') {
                sh "docker rmi ${FULL_IMAGE_PATH} || true"
            }
        }
    }
} // Fin du pipeline
