pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 10, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    triggers {
        // Build on every push to the GitHub webhook.
        githubPush()
    }

    stages {
        stage('Build') {
            steps {
                echo 'Hello, World! - Building the application...'
                echo "Branch: ${env.GIT_BRANCH ?: env.BRANCH_NAME ?: 'N/A'}"
                echo "Commit: ${env.GIT_COMMIT ?: 'N/A'}"
                echo "Author: ${env.GIT_AUTHOR_NAME ?: 'N/A'}"
                sh """
                    set -eux
                    mkdir -p build
                    date -u +'%Y-%m-%dT%H:%M:%SZ' > build/build-info.txt
                    echo "Built from commit ${GIT_COMMIT:-local}" >> build/build-info.txt
                    echo 'Build artifacts ready.'
                """
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
                    docker rm -f hello-world-cd || true

                    # Serve the static site on port 80 of this container
                    docker run -d \
                      --name hello-world-cd \
                      --restart unless-stopped \
                      -p 80:80 \
                      -v "$WORKSPACE:/usr/share/nginx/html:ro" \
                      nginx:alpine

                    # Wait for nginx to come up, then verify
                    sleep 3
                    curl -fsS http://localhost/ | grep -q "Hello World"
                    echo 'Deployment verified: Hello World is live.'
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
