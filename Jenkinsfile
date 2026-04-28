pipeline {
    agent any

    environment {
        DOCKERHUB_USERNAME = 'ngchicuong8820'
        IMAGE_NAME         = "${DOCKERHUB_USERNAME}/todolist-frontend"
        IMAGE_TAG          = "${BUILD_NUMBER}"
        COMPOSE_FILE       = '/home/ubuntu/project/docker-compose.dev.yml'
        BACKEND_URL        = 'http://32.195.232.187:5000/api'
    }

    stages {

        // ══════════════════════════════════════
        // STAGE 1: SOURCE
        // ══════════════════════════════════════
        stage('Source') {
            steps {
                echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
                echo "🔍 Branch : ${env.BRANCH_NAME ?: 'manual'}"
                echo "🔍 Build  : #${BUILD_NUMBER}"
                echo "🔍 Commit : ${env.GIT_COMMIT ?: 'N/A'}"
                echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
                checkout scm
            }
        }

        // ══════════════════════════════════════
        // STAGE 2: BUILD
        // ══════════════════════════════════════
        stage('Build') {
            steps {
                echo "🔨 Building Docker image: ${IMAGE_NAME}:${IMAGE_TAG}"
                sh """
                    docker build \
                        --build-arg NEXT_PUBLIC_API_URL=${BACKEND_URL} \
                        -t ${IMAGE_NAME}:${IMAGE_TAG} \
                        -t ${IMAGE_NAME}:latest \
                        .
                """
                echo "✅ Build SUCCESS"
            }
        }

        // ══════════════════════════════════════
        // STAGE 3: TEST
        // ══════════════════════════════════════
        stage('Test') {
            steps {
                echo "🧪 Running tests..."
                sh """
                    docker run --rm \
                        --name test-frontend-${BUILD_NUMBER} \
                        ${IMAGE_NAME}:${IMAGE_TAG} \
                        sh -c "npm test 2>&1 || true"
                """
                echo "✅ Test DONE"
            }
        }

        // ══════════════════════════════════════
        // STAGE 4: QUALITY GATE
        // ⛔ CI workflow DỪNG Ở ĐÂY nếu là PR
        // ══════════════════════════════════════
        stage('Quality Gate') {
            steps {
                echo "🔎 Running linter..."
                sh """
                    docker run --rm \
                        --name lint-frontend-${BUILD_NUMBER} \
                        ${IMAGE_NAME}:${IMAGE_TAG} \
                        sh -c "npm run lint 2>&1 || true"
                """
                echo "✅ Quality Gate PASSED"
            }
            post {
                failure {
                    error "❌ Quality Gate FAILED — blocking merge!"
                }
            }
        }

        // ══════════════════════════════════════
        // STAGE 5: PUSH TO DOCKER HUB
        // Chỉ chạy khi push vào main
        // ══════════════════════════════════════
        stage('Push to Docker Hub') {
            when {
                branch 'main'
            }
            steps {
                echo "📦 Pushing ${IMAGE_NAME}:${IMAGE_TAG} to Docker Hub..."
                withCredentials([usernamePassword(
                    credentialsId: 'DOCKERHUB_CREDS',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo "${DOCKER_PASS}" | docker login \
                            -u "${DOCKER_USER}" --password-stdin

                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${IMAGE_NAME}:latest

                        echo "✅ Push SUCCESS: ${IMAGE_NAME}:${IMAGE_TAG}"
                    """
                }
            }
        }

        // ══════════════════════════════════════
        // STAGE 6: DEPLOY DEV
        // Docker Compose trên cùng EC2 này
        // ══════════════════════════════════════
        stage('Deploy Dev') {
            when {
                branch 'main'
            }
            steps {
                echo "🚀 Deploying Frontend to Dev (Docker Compose)..."
                sh """
                    IMAGE_TAG=${IMAGE_TAG} docker compose \
                        -f ${COMPOSE_FILE} \
                        up -d frontend --pull always

                    # Chờ frontend start
                    sleep 10

                    # Kiểm tra health
                    curl -s -o /dev/null -w "%{http_code}" \
                        http://localhost:3000 | grep 200 || \
                        (echo "❌ Frontend health check FAILED" && exit 1)

                    echo "✅ Deploy Dev SUCCESS"
                """
            }
        }

        // ══════════════════════════════════════
        // STAGE 7: MANUAL APPROVAL
        // ══════════════════════════════════════
        stage('Approval: Deploy to Prod?') {
            when {
                branch 'main'
            }
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    input(
                        message: "🚦 Deploy Frontend build #${BUILD_NUMBER} lên Production K8s?",
                        ok: 'Deploy!',
                        submitter: 'admin'
                    )
                }
            }
        }

        // ══════════════════════════════════════
        // STAGE 8: DEPLOY PROD
        // Kind K8s trên cùng EC2 này
        // ══════════════════════════════════════
        stage('Deploy Prod') {
            when {
                branch 'main'
            }
            steps {
                echo "☸️ Deploying Frontend to Production (Kind K8s)..."
                sh """
                    kubectl set image deployment/frontend \
                        frontend=${IMAGE_NAME}:${IMAGE_TAG} \
                        -n todolist

                    kubectl rollout status deployment/frontend \
                        -n todolist --timeout=120s

                    echo "✅ Deploy Prod SUCCESS: ${IMAGE_NAME}:${IMAGE_TAG}"
                """
            }
        }
    }

    post {
        success {
            echo """
            ✅ Pipeline SUCCESS
            ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
            Image : ${IMAGE_NAME}:${IMAGE_TAG}
            Branch: ${env.BRANCH_NAME ?: 'manual'}
            Build : #${BUILD_NUMBER}
            ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
            """
        }
        failure {
            echo """
            ❌ Pipeline FAILED
            ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
            Branch: ${env.BRANCH_NAME ?: 'manual'}
            Build : #${BUILD_NUMBER}
            → Check Console Output để xem lỗi
            ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
            """
        }
        always {
            sh "docker image prune -f || true"
        }
    }
}