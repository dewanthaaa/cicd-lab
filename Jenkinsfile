pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 20, unit: 'MINUTES')
        skipDefaultCheckout(true)
    }

    environment {
        PROJECT_NAME          = 'cicd-lab'
        ENV_NAME              = 'development'
        REGISTRY              = 'localhost:5000/cicd-lab'
        DEPLOYMENT_NAME       = 'cicd-lab-development'
        K8S_NAMESPACE         = 'cicd-lab'
        K8S_SECRET_NAME       = 'cicd-lab-development-env'
        INFISICAL_CONNECTION  = 'cicd-lab-infisical'
        INFISICAL_AUTH        = 'cicd-lab-infisical-auth'
        INFISICAL_SECRET      = 'cicd-lab-development'
        DOCKER_BUILDKIT       = '1'
    }

    stages {
        stage('Preflight') {
            steps {
                sh '''#!/bin/sh
                    set -eu
                    command -v git >/dev/null
                    command -v docker >/dev/null
                    command -v kubectl >/dev/null
                    docker version --format '{{.Server.Version}}' >/dev/null
                    kubectl config current-context >/dev/null
                '''
            }
        }

        stage('Checkout Source') {
            steps {
                deleteDir()
                checkout scm
            }
        }

        stage('Wait for Infisical Secret') {
            steps {
                sh '''#!/bin/sh
                    set -eu

                    kubectl -n "$K8S_NAMESPACE" get \
                        "infisicalconnection/$INFISICAL_CONNECTION" >/dev/null
                    kubectl -n "$K8S_NAMESPACE" get \
                        "infisicalauth/$INFISICAL_AUTH" >/dev/null
                    kubectl -n "$K8S_NAMESPACE" get \
                        "infisicalstaticsecret/$INFISICAL_SECRET" >/dev/null

                    kubectl -n "$K8S_NAMESPACE" wait \
                        "infisicalauth/$INFISICAL_AUTH" \
                        --for=condition=secrets.infisical.com/IsReady=True \
                        --timeout=180s
                    kubectl -n "$K8S_NAMESPACE" wait \
                        "infisicalstaticsecret/$INFISICAL_SECRET" \
                        --for=condition=secrets.infisical.com/LastReconcileStatus=True \
                        --timeout=180s

                    present=$(kubectl -n "$K8S_NAMESPACE" get secret "$K8S_SECRET_NAME" \
                        -o go-template='{{if index .data "POC_MESSAGE"}}present{{end}}')
                    if [ "$present" != 'present' ]; then
                        echo >&2 'FATAL: Secret hasil sinkronisasi wajib memiliki key POC_MESSAGE'
                        exit 1
                    fi
                    echo 'Infisical Secret siap; tidak ada nilai secret yang dicetak.'
                '''
            }
        }

        stage('Build & Push Image') {
            steps {
                script {
                    def shortCommit = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()
                    def commitHash = sh(
                        script: 'git rev-parse HEAD',
                        returnStdout: true
                    ).trim()

                    env.GIT_COMMIT_FULL = commitHash
                    env.IMAGE_TAG = "${env.ENV_NAME}-${env.BUILD_NUMBER}-${shortCommit}"
                    env.FULL_IMAGE = "${env.REGISTRY}:${env.IMAGE_TAG}"

                    sh '''#!/bin/sh
                        set -eu
                        docker build \
                            --pull \
                            --build-arg "APP_VERSION=$IMAGE_TAG" \
                            --label "org.opencontainers.image.revision=$GIT_COMMIT_FULL" \
                            --label "ci.jenkins.build=$BUILD_NUMBER" \
                            -t "$FULL_IMAGE" .

                        # Buktikan image menolak startup jika POC_MESSAGE tidak ada.
                        if docker run --rm \
                            --entrypoint /docker-entrypoint.d/10-require-poc-message.sh \
                            "$FULL_IMAGE"; then
                            echo >&2 'FATAL: image seharusnya gagal tanpa POC_MESSAGE'
                            exit 1
                        fi

                        # Buktikan startup check lolos tanpa mencetak nilai variable.
                        docker run --rm \
                            -e POC_MESSAGE=jenkins-validation \
                            --entrypoint /docker-entrypoint.d/10-require-poc-message.sh \
                            "$FULL_IMAGE"

                        docker push "$FULL_IMAGE"
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    env.DEPLOY_ATTEMPTED = 'true'
                    sh '''#!/bin/sh
                        set -eu

                        if ! kubectl -n "$K8S_NAMESPACE" get \
                            "deployment/$DEPLOYMENT_NAME" >/dev/null 2>&1; then
                            echo >&2 "FATAL: apply k8s.yaml lebih dulu; Deployment $DEPLOYMENT_NAME belum ada"
                            exit 1
                        fi

                        kubectl -n "$K8S_NAMESPACE" set image \
                            "deployment/$DEPLOYMENT_NAME" \
                            "$DEPLOYMENT_NAME=$FULL_IMAGE"
                        kubectl -n "$K8S_NAMESPACE" annotate \
                            "deployment/$DEPLOYMENT_NAME" \
                            "kubernetes.io/change-cause=image=$FULL_IMAGE commit=$GIT_COMMIT_FULL jenkins=$BUILD_NUMBER" \
                            --overwrite

                        if ! kubectl -n "$K8S_NAMESPACE" rollout status \
                            "deployment/$DEPLOYMENT_NAME" --timeout=180s; then
                            echo >&2 'ROLLOUT FAILED - rolling back'
                            kubectl -n "$K8S_NAMESPACE" rollout undo \
                                "deployment/$DEPLOYMENT_NAME"
                            kubectl -n "$K8S_NAMESPACE" rollout status \
                                "deployment/$DEPLOYMENT_NAME" --timeout=180s
                            exit 1
                        fi
                    '''
                }
            }
        }

        stage('Health Check') {
            steps {
                sh '''#!/bin/sh
                    set -eu

                    rollback() {
                        kubectl -n "$K8S_NAMESPACE" rollout undo \
                            "deployment/$DEPLOYMENT_NAME"
                        kubectl -n "$K8S_NAMESPACE" rollout status \
                            "deployment/$DEPLOYMENT_NAME" --timeout=180s
                    }

                    ready=$(kubectl -n "$K8S_NAMESPACE" get deployment "$DEPLOYMENT_NAME" \
                        -o jsonpath='{.status.readyReplicas}')
                    wanted=$(kubectl -n "$K8S_NAMESPACE" get deployment "$DEPLOYMENT_NAME" \
                        -o jsonpath='{.spec.replicas}')
                    image=$(kubectl -n "$K8S_NAMESPACE" get deployment "$DEPLOYMENT_NAME" \
                        -o jsonpath='{.spec.template.spec.containers[0].image}')

                    if [ "${ready:-0}" != "$wanted" ] || [ "$image" != "$FULL_IMAGE" ]; then
                        echo >&2 'FATAL: replica atau image hasil deployment tidak sesuai'
                        rollback
                        exit 1
                    fi

                    pod=$(kubectl -n "$K8S_NAMESPACE" get pod \
                        -l "app=$DEPLOYMENT_NAME" \
                        -o jsonpath='{.items[0].metadata.name}')
                    if ! kubectl -n "$K8S_NAMESPACE" exec "$pod" -- \
                        sh -c 'test -n "$POC_MESSAGE"'; then
                        echo >&2 'FATAL: POC_MESSAGE tidak tersedia dalam container'
                        rollback
                        exit 1
                    fi
                    if ! kubectl -n "$K8S_NAMESPACE" exec "$pod" -- \
                        sh -c 'wget -qO- http://127.0.0.1/ | grep -q "id=\"infisical-message\""'; then
                        echo >&2 'FATAL: halaman Nginx hasil template tidak dapat diakses'
                        rollback
                        exit 1
                    fi

                    echo "Deployment sehat: ${ready}/${wanted} replica, image=$image"
                '''
            }
        }

        stage('Promote Image') {
            steps {
                sh '''#!/bin/sh
                    set -eu
                    docker tag "$FULL_IMAGE" "$REGISTRY:$ENV_NAME"
                    docker push "$REGISTRY:$ENV_NAME"
                '''
            }
        }
    }

    post {
        success {
            echo "SUCCESS DEPLOY => ${env.FULL_IMAGE}"
        }
        failure {
            echo "FAILED DEPLOY => ${env.FULL_IMAGE ?: 'image-not-built'} (attempted=${env.DEPLOY_ATTEMPTED ?: 'false'})"
        }
        cleanup {
            deleteDir()
        }
    }
}
