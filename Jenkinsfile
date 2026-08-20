pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 10, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    triggers {
        githubPush()
    }

    environment {
        DEPLOY_PORT = '8088'
        SITE_NAME   = 'hello-world-cd'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo 'Hello, World! - Building the application...'
                echo "Branch: ${env.BRANCH_NAME ?: env.GIT_BRANCH ?: 'N/A'}"
                echo "Commit: ${env.GIT_COMMIT ?: 'N/A'}"
                echo "Author: ${env.GIT_AUTHOR_NAME ?: 'N/A'}"
                sh '''
                    set -eux
                    mkdir -p build
                    date -u +'%Y-%m-%dT%H:%M:%SZ' > build/build-info.txt
                    echo "Built from commit $(git rev-parse --short HEAD)" >> build/build-info.txt
                    echo 'Build artifacts ready.'
                '''
            }
        }

        stage('Test') {
            steps {
                echo 'Running tests...'
                sh '''
                    set -eux
                    test -f index.html
                    grep -q "Hello World" index.html
                    echo 'All tests passed.'
                '''
            }
        }

        stage('Package') {
            steps {
                echo 'Packaging the static site...'
                sh 'tar -czf build/site.tar.gz index.html'
                archiveArtifacts artifacts: 'build/site.tar.gz', fingerprint: true
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploying Hello World site...'
                sh '''#!/bin/bash
                    set -eux
                    # Stop any previous deployment
                    docker rm -f "${SITE_NAME}" || true

                    # Copy the built site to a persistent location on the
                    # Docker host. The Jenkins workspace is wiped by
                    # cleanWs() in the `post { always }` block, so we must
                    # not mount the workspace directly — nginx would serve
                    # an empty directory after every build.
                    SITE_DIR="/var/jenkins-deploy/${SITE_NAME}"
                    mkdir -p "${SITE_DIR}"
                    cp -f index.html "${SITE_DIR}/"

                    # Serve the static site on host port 8088 -> container 80.
                    docker run -d \
                      --name "${SITE_NAME}" \
                      --restart unless-stopped \
                      -p "${DEPLOY_PORT}:80" \
                      -v "${SITE_DIR}:/usr/share/nginx/html:ro" \
                      nginx:alpine

                    # Wait for nginx to come up, then verify.
                    # The Jenkins container has its own network namespace,
                    # so `localhost` inside it is NOT the host. We hit the
                    # host via the docker bridge gateway IP, discovered from
                    # /proc/net/route (no `ip` command in the jenkins image).
                    sleep 3
                    HEX_GW=$(awk 'NR>1 && $2=="00000000" {print $3}' /proc/net/route | head -1 | tr -d ' ')
                    HOST_IP=$(printf '%d.%d.%d.%d\n' "0x${HEX_GW:6:2}" "0x${HEX_GW:4:2}" "0x${HEX_GW:2:2}" "0x${HEX_GW:0:2}")
                    echo "Host gateway: ${HOST_IP}"
                    curl -fsS "http://${HOST_IP}:${DEPLOY_PORT}/" | grep -q "Hello World"
                    echo "Deployment verified: Hello World is live on host port ${DEPLOY_PORT}."
                '''
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully! 🎉 — Hello World is live.'
        }
        failure {
            echo 'Pipeline failed. Check the logs above.'
        }
        always {
            echo 'Cleaning up workspace...'
            cleanWs()
        }
    }
}
