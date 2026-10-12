
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 30, unit: 'MINUTES')
    }

    environment {
        DEPLOY_DIR = '/opt/retail-analytics'
        COMPOSE_FILE = 'docker-compose.yml'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git log -1 --oneline'
            }
        }

        stage('Validate Files') {
            steps {
                sh '''
                    set -eu
                    test -f docker-compose.yml
                    test -f backend/Dockerfile
                    test -f frontend/Dockerfile
                    test -f frontend/nginx.conf
                    echo "Required deployment files found."
                '''
            }
        }

        stage('Prepare Deployment') {
            steps {
                sh '''
                    set -eu

                    test -f "$DEPLOY_DIR/.env" || {
                        echo "ERROR: Production .env file is missing from $DEPLOY_DIR"
                        exit 1
                    }

                    rsync -a --delete \
                        --exclude='.env' \
                        --exclude='.git/' \
                        "$WORKSPACE/" "$DEPLOY_DIR/"

                    chmod 600 "$DEPLOY_DIR/.env"
                '''
            }
        }

        stage('Build and Deploy') {
            steps {
                sh '''
                    set -eu

                    docker compose \
                        --project-directory "$DEPLOY_DIR" \
                        --env-file "$DEPLOY_DIR/.env" \
                        -f "$DEPLOY_DIR/$COMPOSE_FILE" \
                        config --quiet

                    docker compose \
                        --project-directory "$DEPLOY_DIR" \
                        --env-file "$DEPLOY_DIR/.env" \
                        -f "$DEPLOY_DIR/$COMPOSE_FILE" \
                        up -d --build
                '''
            }
        }

        stage('Initialize Database') {
            steps {
                sh '''
                    set -eu

                    docker compose \
                        --project-directory "$DEPLOY_DIR" \
                        --env-file "$DEPLOY_DIR/.env" \
                        -f "$DEPLOY_DIR/$COMPOSE_FILE" \
                        exec -T backend \
                        python -m backend.database.init_db
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    set -eu

                    docker compose \
                        --project-directory "$DEPLOY_DIR" \
                        --env-file "$DEPLOY_DIR/.env" \
                        -f "$DEPLOY_DIR/$COMPOSE_FILE" \
                        ps

                    for attempt in $(seq 1 12); do
                        if curl --fail --silent http://localhost/health; then
                            echo ""
                            echo "Deployment health check passed."
                            exit 0
                        fi

                        echo "Waiting for application health ($attempt/12)..."
                        sleep 5
                    done

                    echo "ERROR: Health check failed."
                    docker compose \
                        --project-directory "$DEPLOY_DIR" \
                        --env-file "$DEPLOY_DIR/.env" \
                        -f "$DEPLOY_DIR/$COMPOSE_FILE" \
                        logs --tail=80

                    exit 1
                '''
            }
        }
    }

    post {
        success {
            echo 'Deployment successful. Application: http://18.206.96.103/'
        }

        failure {
            echo 'Deployment failed. Review the stage logs and Docker Compose logs.'
        }

        always {
            echo 'Jenkins pipeline finished.'
        }
    }
}
