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
                      -e GOCACHE=/tmp/go-build \
                      -e GOPATH=/tmp/go \
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
                      -e HOME=/tmp/home \
                      -e npm_config_cache=/tmp/npm-cache \
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

                    echo "Initial service startup wait..."
                    sleep 5
                '''
            }
        }

        stage('Deployment Health Check') {
             steps {
                  sh '''
                       echo "===== DEPLOYMENT HEALTH CHECK ====="

                       MAX_ATTEMPTS=12
                       SLEEP_SECONDS=5
                       HEALTHY=false

            for ATTEMPT in $(seq 1 ${MAX_ATTEMPTS}); do
                echo ""
                echo "===== HEALTH CHECK ATTEMPT ${ATTEMPT}/${MAX_ATTEMPTS} ====="

                MYSQL_STATUS=$(docker inspect \
                    --format '{{.State.Health.Status}}' \
                    shopverse-mysql 2>/dev/null || echo "missing")

                FRONTEND_STATUS=$(docker inspect \
                    --format '{{.State.Health.Status}}' \
                    shopverse-frontend 2>/dev/null || echo "missing")

                NGINX_STATUS=$(docker inspect \
                    --format '{{.State.Health.Status}}' \
                    shopverse-nginx 2>/dev/null || echo "missing")

                BACKEND_HEALTH=$(docker exec \
                    shopverse-nginx \
                    sh -c "wget -q -O - http://backend:8080/health" \
                    2>/dev/null || true)

                PUBLIC_HEALTH=$(curl -fsS \
                    --max-time 5 \
                    "http://${PUBLIC_HOST}/health" \
                    2>/dev/null || true)

                echo "MySQL status:    ${MYSQL_STATUS}"
                echo "Frontend status: ${FRONTEND_STATUS}"
                echo "Nginx status:    ${NGINX_STATUS}"
                echo "Backend response: ${BACKEND_HEALTH}"
                echo "Public response:  ${PUBLIC_HEALTH}"

                if [ "${MYSQL_STATUS}" = "healthy" ] && \
                   [ "${FRONTEND_STATUS}" = "healthy" ] && \
                   [ "${NGINX_STATUS}" = "healthy" ] && \
                   echo "${BACKEND_HEALTH}" | grep -q '"status":"healthy"' && \
                   echo "${PUBLIC_HEALTH}" | grep -q '"status":"healthy"'; then

                    HEALTHY=true
                    echo ""
                    echo "All deployment health checks passed."
                    break
                fi

                if [ "${ATTEMPT}" -lt "${MAX_ATTEMPTS}" ]; then
                    echo "Services are not ready yet. Waiting ${SLEEP_SECONDS}s..."
                    sleep "${SLEEP_SECONDS}"
                fi
            done

            echo ""
            echo "===== FINAL COMPOSE STATUS ====="
            docker compose ps

            if [ "${HEALTHY}" != "true" ]; then
                echo "Deployment health checks failed after ${MAX_ATTEMPTS} attempts."
                exit 1
            fi
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

                            echo "Restoring previous version ${PREVIOUS_TAG}..."
                            sed -i "s/^IMAGE_TAG=.*/IMAGE_TAG=${PREVIOUS_TAG}/" .env 2>/dev/null || true

                            docker compose up -d \
                              --force-recreate \
                              --remove-orphans

                            MAX_ATTEMPTS=12
                            SLEEP_SECONDS=5
                            ROLLBACK_HEALTHY=false

                            for ATTEMPT in $(seq 1 ${MAX_ATTEMPTS}); do
                                echo ""
                                echo "===== ROLLBACK HEALTH CHECK ${ATTEMPT}/${MAX_ATTEMPTS} ====="

                                MYSQL_STATUS=$(docker inspect \
                                  --format "{{.State.Health.Status}}" \
                                  shopverse-mysql 2>/dev/null || echo "missing")

                                FRONTEND_STATUS=$(docker inspect \
                                  --format "{{.State.Health.Status}}" \
                                  shopverse-frontend 2>/dev/null || echo "missing")

                                NGINX_STATUS=$(docker inspect \
                                  --format "{{.State.Health.Status}}" \
                                  shopverse-nginx 2>/dev/null || echo "missing")

                                BACKEND_HEALTH=$(docker exec \
                                  shopverse-nginx \
                                  sh -c "wget -q -O - http://backend:8080/health" \
                                  2>/dev/null || true)

                                PUBLIC_HEALTH=$(curl -fsS \
                                  --max-time 5 \
                                  "http://${PUBLIC_HOST}/health" \
                                  2>/dev/null || true)

                                echo "MySQL status:     ${MYSQL_STATUS}"
                                echo "Frontend status:  ${FRONTEND_STATUS}"
                                echo "Nginx status:     ${NGINX_STATUS}"
                                echo "Backend response: ${BACKEND_HEALTH}"
                                echo "Public response:  ${PUBLIC_HEALTH}"

                                if [ "${MYSQL_STATUS}" = "healthy" ] && \
                                   [ "${FRONTEND_STATUS}" = "healthy" ] && \
                                   [ "${NGINX_STATUS}" = "healthy" ] && \
                                    echo "${BACKEND_HEALTH}" | grep -q '"status":"healthy"' && \
                                    echo "${PUBLIC_HEALTH}" | grep -q '"status":"healthy"'; then

                                    ROLLBACK_HEALTHY=true
                                    echo "Rollback health checks passed."
                                    break
                                fi

                                if [ "${ATTEMPT}" -lt "${MAX_ATTEMPTS}" ]; then
                                    echo "Rollback services are not ready yet. Waiting ${SLEEP_SECONDS}s..."
                                    sleep "${SLEEP_SECONDS}"
                                fi
                            done

                            echo ""
                            echo "===== ROLLBACK COMPOSE STATUS ====="
                            docker compose ps

                            if [ "${ROLLBACK_HEALTHY}" != "true" ]; then
                                echo "Rollback health checks failed."
                                exit 1
                            fi

                            echo "Rollback to version ${PREVIOUS_TAG} completed and verified successfully."
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
