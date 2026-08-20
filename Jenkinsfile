pipeline {
    agent any


    parameters {
        choice(name: 'BRANCH', choices: ['main', 'develop', 'feature/*'], description: 'Git branch')
        choice(name: 'ENV',    choices: ['dev', 'staging', 'prod'],  description: 'Target environment')
    }

    environment {
        REPO_URL    = 'https://github.com/BayajidAlam/jenkins-poc.git'
        DEPLOY_ROOT = '/var/jenkins-deploy'
    }

    stages {
        stage('application') {
            steps {
                dir('application') {
                    checkout([$class: 'GitSCM',
                        branches: [[name: "*/${BRANCH}"]],
                        userRemoteConfigs: [[
                            credentialsId: 'github-pat',
                            url: "${REPO_URL}"
                        ]]
                    ])
                }
            }
        }

        stage('Build') {
            steps {
                sh '''
                    cd application
                    mkdir -p build
                    date -u +'%Y-%m-%dT%H:%M:%SZ' > build/build-info.txt
                    echo "Built from commit $(git rev-parse --short HEAD)" >> build/build-info.txt
                    echo "Branch: ${BRANCH}" >> build/build-info.txt
                    echo "Env:    ${ENV}"    >> build/build-info.txt
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    cd application
                    test -f index.html
                    grep -q "Hello World" index.html
                '''
            }
        }

        stage('Package') {
            steps {
                sh 'tar -czf site.tar.gz -C application index.html'
                archiveArtifacts artifacts: 'site.tar.gz', fingerprint: true
            }
        }

        stage('Deploy') {
            steps {
                script {
                    def safeBranch = BRANCH.replaceAll('/', '-')
                    def envPort = [dev: '8088', staging: '8089', prod: '8090']
                    def port = envPort[ENV]

                    def ts  = sh(returnStdout: true, script: "date -u +'%Y%m%dT%H%M%SZ'").trim()
                    def sha = sh(returnStdout: true, script: "git -C application rev-parse --short HEAD").trim()

                    env.SITE_NAME = "${safeBranch}-${ENV}-${ts}-${sha}"
                    env.SITE_DIR  = "${DEPLOY_ROOT}/${env.SITE_NAME}"
                    env.PORT      = port

                    echo "Site: ${env.SITE_NAME} on port ${env.PORT}"
                }

                sh '''#!/bin/bash
                    set -eux

                    # Discover host gateway (Jenkins container has its own net ns).
                    HEX_GW=$(awk 'NR>1 && $2=="00000000" {print $3}' /proc/net/route | head -1 | tr -d ' ')
                    HOST_IP=$(printf '%d.%d.%d.%d\\n' "0x${HEX_GW:6:2}" "0x${HEX_GW:4:2}" "0x${HEX_GW:2:2}" "0x${HEX_GW:0:2}")

                    mkdir -p "${SITE_DIR}"
                    cp -f application/index.html "${SITE_DIR}/"

                    # Prune older <branch>-<env>-* containers before starting the new one.
                    docker ps -a --format '{{.Names}}' \
                      | grep "^$(echo ${SITE_NAME} | sed 's/-[0-9].*//')-" \
                      | grep -v "^${SITE_NAME}$" \
                      | xargs -r docker rm -f || true

                    docker run -d \
                      --name "${SITE_NAME}" \
                      --restart unless-stopped \
                      -p "${PORT}:80" \
                      -v "${SITE_DIR}:/usr/share/nginx/html:ro" \
                      nginx:alpine

                    sleep 3
                    curl -fsS "http://${HOST_IP}:${PORT}/" | grep -q "Hello World"
                    echo "Live at http://${HOST_IP}:${PORT}/"
                '''
            }
        }

        stage('Cleanup') {
            steps {
                sh '''
                    SAFE_PREFIX=$(echo "${SITE_NAME}" | sed 's/-[0-9]\\{8\\}T.*//')
                    find "${DEPLOY_ROOT}" -maxdepth 1 -type d \
                      -name "${SAFE_PREFIX}-*" \
                      ! -name "${SITE_NAME}" \
                      -exec rm -rf {} +
                '''
            }
        }
    }

    post {
        success { echo "Done — ${env.SITE_NAME} on port ${env.PORT}" }
        failure { echo 'Failed. Check logs.' }
        always  { cleanWs() }
    }
}
