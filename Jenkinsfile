pipeline {
    agent any
    options { timestamps(); disableConcurrentBuilds() }
    environment {
        IMAGE_NAME  = 'cicd-lab'
        REGISTRY    = 'localhost:5000/cicd-lab'   // registry lokal (port-forward minikube)
        DEPLOYMENT  = 'cicd-lab-development'
        NAMESPACE   = 'default'
    }
    stages {
        stage('Checkout') {
            steps { checkout scm }                // Jenkins yang ambil kode (bukan git clone manual)
        }
        stage('Build & Push Image') {
            steps {
                script {
                    def sha = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.IMAGE_TAG  = "dev-${sha}"
                    env.FULL_IMAGE = "${env.REGISTRY}:${env.IMAGE_TAG}"
                    sh """
                        sed -i "s/__VERSION__/${env.IMAGE_TAG}/" index.html
                        docker build -t ${env.FULL_IMAGE} .
                        docker push ${env.FULL_IMAGE}
                        docker tag ${env.FULL_IMAGE} ${env.REGISTRY}:development
                        docker push ${env.REGISTRY}:development
                    """
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                sh """
                    kubectl -n ${NAMESPACE} set image deployment/${DEPLOYMENT} \
                        ${DEPLOYMENT}=${env.FULL_IMAGE}
                    kubectl -n ${NAMESPACE} rollout restart deployment/${DEPLOYMENT}
                    kubectl -n ${NAMESPACE} rollout status deployment/${DEPLOYMENT} --timeout=120s
                """
            }
        }
    }
    post {
        success { echo "OK deployed ${env.FULL_IMAGE}" }
        failure { echo "GAGAL — cek log stage di atas" }
    }
}