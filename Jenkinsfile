pipeline {

    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 60, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '30'))
        skipDefaultCheckout(true)
    }

    environment {
        APP_NAME        = "inspection-be"
        REPO_URL        = "https://github.com/ankittyagi140/euroasiasci_inspection_BE.git"
        GIT_BRANCH      = "release"
        DOCKER_BUILDKIT = "1"
        DOCKERFILE      = "Dockerfile"
        REGISTRY        = "ghcr.io/euroasiasci"
        IMAGE           = "${REGISTRY}/${APP_NAME}"
        IMAGE_TAG       = "${BUILD_NUMBER}"
        ENV_TAG         = "prod"
        PLATFORM_JOB    = "platform/deploy-prod"
    }

    stages {

        stage('Production approval') {
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input(
                        message: "Build and deploy inspection-be #${env.BUILD_NUMBER} from release to PRODUCTION?",
                        ok: 'Deploy to production'
                    )
                }
            }
        }

        stage('Checkout') {
            steps {
                git(
                    branch: GIT_BRANCH,
                    credentialsId: 'github-pat',
                    url: REPO_URL
                )
                script {
                    env.GIT_SHA = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()
                    def branch = sh(
                        script: 'git rev-parse --abbrev-ref HEAD',
                        returnStdout: true
                    ).trim()
                    if (branch != 'release' && !branch.startsWith('HEAD')) {
                        // detached HEAD is fine after checkout of release tip
                        echo "Checked out ${branch} @ ${env.GIT_SHA}"
                    }
                }
            }
        }

        stage('Lint & Unit Test') {
            steps {
                sh '''
                set -eux
                CI_CONTAINER="inspection-be-ci-prod-${BUILD_NUMBER}"
                docker rm -f "${CI_CONTAINER}" >/dev/null 2>&1 || true
                docker run \
                    -d \
                    --name "${CI_CONTAINER}" \
                    -w /workspace \
                    python:3.13-slim-bookworm \
                    sleep 900
                docker cp "$PWD/." "${CI_CONTAINER}:/workspace"
                docker exec "${CI_CONTAINER}" \
                    bash /workspace/scripts/ci-lint-test.sh
                docker cp \
                    "${CI_CONTAINER}:/workspace/htmlcov" \
                    htmlcov || true
                docker cp \
                    "${CI_CONTAINER}:/workspace/coverage.xml" \
                    coverage.xml || true
                docker rm -f "${CI_CONTAINER}"
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                set -eux
                docker build \
                    --pull \
                    --build-arg BUILDKIT_INLINE_CACHE=1 \
                    --label org.opencontainers.image.version=${BUILD_NUMBER} \
                    --label org.opencontainers.image.revision=${GIT_SHA} \
                    --label org.opencontainers.image.source=${REPO_URL} \
                    --label org.opencontainers.image.environment=production \
                    -f ${DOCKERFILE} \
                    -t ${IMAGE}:${IMAGE_TAG} \
                    -t ${IMAGE}:${ENV_TAG} \
                    .
                '''
            }
        }

        stage('Login GHCR') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'ghcr-token',
                        usernameVariable: 'GHCR_USER',
                        passwordVariable: 'GHCR_TOKEN'
                    )
                ]) {
                    sh '''
                    echo "${GHCR_TOKEN}" | docker login ghcr.io \
                        --username "${GHCR_USER}" \
                        --password-stdin
                    '''
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                sh '''
                set -eux
                docker push ${IMAGE}:${IMAGE_TAG}
                docker push ${IMAGE}:${ENV_TAG}
                '''
            }
        }

        stage('Trigger Platform Production Deployment') {
            steps {
                script {
                    echo "Triggering Platform Production Deployment..."
                    build(
                        job: PLATFORM_JOB,
                        wait: true,
                        propagate: true,
                        parameters: [
                            string(name: 'APPLICATION', value: APP_NAME),
                            string(name: 'IMAGE_TAG', value: IMAGE_TAG)
                        ]
                    )
                }
            }
        }
    }

    post {
        success {
            archiveArtifacts(
                artifacts: 'coverage.xml,htmlcov/**',
                allowEmptyArchive: true
            )
            echo """
========================================
Backend CI Successful (PRODUCTION)
Application : ${APP_NAME}
Branch      : ${GIT_BRANCH}
Image       : ${IMAGE}:${IMAGE_TAG} (also tagged :prod)
========================================
"""
        }
        failure {
            echo """
========================================
Backend CI Failed (PRODUCTION)
Build : ${BUILD_NUMBER}
========================================
"""
        }
        always {
            sh 'docker logout ghcr.io || true'
            cleanWs()
        }
    }
}
