pipeline {
    agent any

    environment {
        BACKEND_IMAGE = 'shopverse-backend'
        FRONTEND_IMAGE = 'shopverse-frontend'
        COMPOSE_PROJECT_NAME = 'shopverse'
        PUBLIC_HOST = '13.203.207.180'
        VITE_API_URL = '/api'
        STATE_DIR = '/home/ubuntu/.shopverse'
        DOCKER_BUILDKIT = '1'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    stages {

        stage('Checkout Code') {
            steps {
                deleteDir()
                checkout scm

                sh '''
                    echo "Current branch:"
                    git branch --show-current

                    echo "Current commit:"
                    git rev-parse --short HEAD

                    echo "Repository status:"
                    git status --short
                '''
            }
        }

        stage('Backend Test') {
            steps {
                sh '''
                    docker run --rm \
                      --user "$(id -u):$(id -g)" \
                      -v "$WORKSPACE/backend:/app" \
                      -w /app \
                      golang:1.24-alpine \
                      sh -c 'go mod download && go test ./...'
                '''
            }
        }

        stage('Frontend Test and Build') {
            steps {
                sh '''
                    docker run --rm \
                      --user "$(id -u):$(id -g)" \
                      -v "$WORKSPACE/frontend:/app" \
                      -w /app \
                      node:18-alpine \
                      sh -c 'npm ci && npm run build'
                '''
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    echo "Building backend image..."
                    docker build \
                      --label com.shopverse.app=shopverse \
                      -t ${BACKEND_IMAGE}:${BUILD_NUMBER} \
                      ./backend

                    echo "Building frontend image..."
                    docker build \
                      --label com.shopverse.app=shopverse \
                      --build-arg VITE_API_URL=${VITE_API_URL} \
                      -t ${FRONTEND_IMAGE}:${BUILD_NUMBER} \
                      ./frontend

                    echo "Built images:"
                    docker images | grep -E 'shopverse-backend|shopverse-frontend'
                '''
            }
        }

        stage('Trivy Security Scan') {
            steps {
                sh '''
                    trivy image \
                      --severity HIGH,CRITICAL \
                      --ignore-unfixed \
                      --format table \
                      --output backend-trivy.txt \
                      ${BACKEND_IMAGE}:${BUILD_NUMBER}

                    trivy image \
                      --severity HIGH,CRITICAL \
                      --ignore-unfixed \
                      --format table \
                      --output frontend-trivy.txt \
                      ${FRONTEND_IMAGE}:${BUILD_NUMBER}

                    echo "===== BACKEND HIGH/CRITICAL SUMMARY ====="
                    grep -E 'Total:|HIGH|CRITICAL' backend-trivy.txt || true

                    echo "===== FRONTEND HIGH/CRITICAL SUMMARY ====="
                    grep -E 'Total:|HIGH|CRITICAL' frontend-trivy.txt || true
                '''
            }
        }

        stage('Prepare Deployment Environment') {
            steps {
                withCredentials([
                    string(credentialsId: 'shopverse-mysql-root-password', variable: 'MYSQL_ROOT_PASSWORD'),
                    string(credentialsId: 'shopverse-mysql-password', variable: 'MYSQL_PASSWORD'),
                    string(credentialsId: 'shopverse-jwt-secret', variable: 'JWT_SECRET')
                ]) {
                    sh '''
                        mkdir -p "${STATE_DIR}"
                        chmod 700 "${STATE_DIR}"

                        umask 077

                        cat > .env <<ENVFILE
MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
MYSQL_DATABASE=shopverse
MYSQL_USER=shopverse
MYSQL_PASSWORD=${MYSQL_PASSWORD}
JWT_SECRET=${JWT_SECRET}
FRONTEND_ORIGIN=http://${PUBLIC_HOST}
VITE_API_URL=${VITE_API_URL}
IMAGE_TAG=${BUILD_NUMBER}
ENVFILE

                        echo "Deployment environment prepared."
                    '''
                }
            }
        }

        stage('Capture Previous Version') {
            steps {
                script {
                    env.PREVIOUS_TAG = sh(
                        script: '''
                            if [ -f "${STATE_DIR}/last-successful-tag" ]; then
                                cat "${STATE_DIR}/last-successful-tag"
                            else
                                docker inspect shopverse-backend \
                                  --format '{{.Config.Image}}' 2>/dev/null \
                                  | sed 's/.*://' || true
                            fi
                        ''',
                        returnStdout: true
                    ).trim()

                    echo "Previous successful version: ${env.PREVIOUS_TAG ?: 'none'}"
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    env.DEPLOY_ATTEMPTED = 'true'
                }

                sh '''
                    echo "Deploying version ${BUILD_NUMBER}..."

                    docker compose up -d \
                      --force-recreate \
                      --remove-orphans

                    echo "Waiting for services..."
                    sleep 20
                '''
            }
        }

        stage('Deployment Health Check') {
            steps {
                sh '''
                    echo "===== COMPOSE STATUS ====="
                    docker compose ps

                    echo "===== MYSQL HEALTH ====="
                    MYSQL_STATUS=$(docker inspect \
                      --format '{{.State.Health.Status}}' \
                      shopverse-mysql)

                    echo "MySQL status: ${MYSQL_STATUS}"

                    if [ "${MYSQL_STATUS}" != "healthy" ]; then
                        echo "MySQL health check failed."
                        exit 1
                    fi

                    echo "===== FRONTEND HEALTH ====="
                    FRONTEND_STATUS=$(docker inspect \
                      --format '{{.State.Health.Status}}' \
                      shopverse-frontend)

                    echo "Frontend status: ${FRONTEND_STATUS}"

                    if [ "${FRONTEND_STATUS}" != "healthy" ]; then
                        echo "Frontend health check failed."
                        exit 1
                    fi

                    echo "===== NGINX HEALTH ====="
                    NGINX_STATUS=$(docker inspect \
                      --format '{{.State.Health.Status}}' \
                      shopverse-nginx)

                    echo "Nginx status: ${NGINX_STATUS}"

                    if [ "${NGINX_STATUS}" != "healthy" ]; then
                        echo "Nginx health check failed."
                        exit 1
                    fi

                    echo "===== BACKEND HEALTH ====="
                    BACKEND_HEALTH=$(docker exec \
                      shopverse-nginx \
                      sh -c "wget -q -O - \"\$(printf 'http://%s:8080/health' backend)\"")

                    echo "Backend response: ${BACKEND_HEALTH}"

                    echo "${BACKEND_HEALTH}" | grep -q '"status":"healthy"'

                    echo "===== FRONTEND RESPONSE ====="
                    docker exec \
                      shopverse-nginx \
                      sh -c "wget -q -O - \"\$(printf 'http://%s/' frontend)\"" \
                      > /tmp/frontend-response.html

                    test -s /tmp/frontend-response.html

                    echo "Frontend response received successfully."

                    echo "===== PUBLIC NGINX CHECK ====="
                    curl -fsS -I \
                      "http://${PUBLIC_HOST}" \
                      | head -n 1

                    echo "===== PUBLIC BACKEND HEALTH ====="
                    curl -fsS \
                      "http://${PUBLIC_HOST}/health"

                    echo
                    echo "All deployment health checks passed."
                '''
            }
        }

        stage('Record Successful Version') {
            steps {
                sh '''
                    echo "${BUILD_NUMBER}" > "${STATE_DIR}/last-successful-tag"
                    chmod 600 "${STATE_DIR}/last-successful-tag"

                    echo "Successful version recorded:"
                    cat "${STATE_DIR}/last-successful-tag"
                '''
            }
        }

        stage('Cleanup Old Application Images') {
            steps {
                sh '''
                    echo "Cleaning old ShopVerse application images..."

                    for REPO in "${BACKEND_IMAGE}" "${FRONTEND_IMAGE}"; do
                        docker images "${REPO}" \
                          --format '{{.Repository}}:{{.Tag}}' \
                          | grep -E ':[0-9]+$' \
                          | sort -t: -k2,2nr \
                          | tail -n +4 \
                          | while read IMAGE; do

                                TAG="${IMAGE##*:}"

                                if [ "${TAG}" != "${BUILD_NUMBER}" ] && \
                                   [ "${TAG}" != "${PREVIOUS_TAG}" ]; then
                                    echo "Removing old image: ${IMAGE}"
                                    docker image rm "${IMAGE}" || true
                                fi
                          done
                    done

                    echo "Application image cleanup completed."

                    echo "Remaining ShopVerse images:"
                    docker images | grep -E 'shopverse-backend|shopverse-frontend' || true
                '''
            }
        }
    }

    post {

        always {
            archiveArtifacts(
                artifacts: 'backend-trivy.txt,frontend-trivy.txt',
                allowEmptyArchive: false
            )
        }

        failure {
            script {
                if (env.DEPLOY_ATTEMPTED == 'true' &&
                    env.PREVIOUS_TAG?.trim()) {

                    echo "Deployment failed."
                    echo "Attempting rollback to version ${env.PREVIOUS_TAG}..."

                    sh '''
                        if docker image inspect \
                          "${BACKEND_IMAGE}:${PREVIOUS_TAG}" >/dev/null 2>&1 && \
                           docker image inspect \
                          "${FRONTEND_IMAGE}:${PREVIOUS_TAG}" >/dev/null 2>&1; then

                            sed -i "s/^IMAGE_TAG=.*/IMAGE_TAG=${PREVIOUS_TAG}/" .env 2>/dev/null || true

                            docker compose up -d \
                              --force-recreate \
                              --remove-orphans

                            sleep 20

                            docker compose ps

                            echo "Rollback to version ${PREVIOUS_TAG} completed."
                        else
                            echo "Previous application images are not available."
                            echo "Rollback could not be performed."
                        fi
                    '''
                } else {
                    echo "No previous deployment available for rollback."
                }
            }
        }

        cleanup {
            sh '''
                rm -f .env
            '''
        }
    }
}
