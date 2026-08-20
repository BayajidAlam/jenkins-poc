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
                sh '''
                    set -eux
                    # Stop any previous deployment
                    docker rm -f "${SITE_NAME}" || true

                    # Serve the static site on host port 8088 -> container 80
                    # (host port 80 is already used by the Jenkins container's
                    # 80:80 mapping in docker-compose.yml, so we use 8088 instead)
                    docker run -d \
                      --name "${SITE_NAME}" \
                      --restart unless-stopped \
                      -p "${DEPLOY_PORT}:80" \
                      -v "${WORKSPACE}:/usr/share/nginx/html:ro" \
                      nginx:alpine

                    # Wait for nginx to come up, then verify
                    sleep 3
                    curl -fsS "http://localhost:${DEPLOY_PORT}/" | grep -q "Hello World"
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
