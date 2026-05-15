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
  - name: jnlp
    image: jenkins/inbound-agent:3355.v388858a_47b_33-3-jdk21
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
    volumeMounts:
    - mountPath: /home/jenkins/agent
      name: workspace-volume
  nodeSelector:
    kubernetes.io/os: linux
  restartPolicy: Never
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
  - name: workspace-volume
    emptyDir: {}
"""
        }
    }

    environment {
        // Centralisation de l'adresse de Harbor et du projet par défaut 'library'
        REGISTRY     = "10.43.108.59:80"
        IMAGE_NAME   = "library/mon-app"
        IMAGE_TAG    = "${BUILD_NUMBER}"
        FULL_IMAGE   = "${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
    }

    stages {
        stage("1. Checkout SCM") {
            steps {
                // Cette étape est gérée automatiquement par Jenkins Declarative Pipeline
                checkout scm
            }
        }

        stage("2. Build de l'image") {
            steps {
                container('docker') {
                    sh "docker build -t ${FULL_IMAGE} ."
                }
            }
        }

        stage("3. Scan de sécurité (Trivy)") {
            steps {
                container('docker') {
                    script {
                        sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image --severity HIGH,CRITICAL ${FULL_IMAGE}"
                    }
                }
            }
        }

        stage("4. Push vers Harbor") {
            steps {
                container('docker') {
                    // Remplace 'harbor-creds' par l'ID exact de tes credentials Jenkins (admin / Harbor12345)
                    withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PWD')]) {
                        // Utilisation des simples guillemets '' pour protéger le mot de passe
                        sh 'echo "$HARBOR_PWD" | docker login ${REGISTRY} -u "$HARBOR_USER" --password-stdin'
                        sh "docker push ${FULL_IMAGE}"
                    }
                }
            }
        }
    }

    post {
        always {
            container('docker') {
                echo "Nettoyage de l'image locale..."
                sh "docker rmi ${FULL_IMAGE} || true"
            }
        }
        success {
            echo "✅ Pipeline réussi ! L'image a été poussée avec succès sur Harbor."
        }
        failure {
            echo "❌ Échec : Vérifiez les logs pour comprendre l'erreur."
        }
    }
}
