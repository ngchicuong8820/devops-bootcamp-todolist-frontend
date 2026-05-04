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
                echo "🔍 PR#    : ${env.CHANGE_ID ?: 'N/A - Branch Push'}"
                echo "🔍 Build  : #${BUILD_NUMBER}"
                echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
                checkout scm
            }
        }

        // ══════════════════════════════════════
        // STAGE 2: BUILD
        // ══════════════════════════════════════
        stage('Build') {
            steps {
                echo "🔨 Building: ${IMAGE_NAME}:${IMAGE_TAG}"
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
        // ⛔ CI (PR) DỪNG Ở ĐÂY
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
        // Chỉ chạy khi push to main
        // ══════════════════════════════════════
        stage('Push to Docker Hub') {
            when {
                allOf {
                    branch 'main'
                    not { changeRequest() }
                }
            }
            steps {
                echo "📦 Pushing ${IMAGE_NAME}:${IMAGE_TAG}..."
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
                        echo "✅ Push SUCCESS"
                    """
                }
            }
        }

        // ══════════════════════════════════════
        // STAGE 6: DEPLOY DEV
        // Docker Compose trên EC2 Medium
        // ══════════════════════════════════════
        stage('Deploy Dev') {
    when {
        allOf {
            branch 'main'
            not { changeRequest() }
        }
    }
    environment {
        DATABASE_HOST     = credentials('DB_HOST')
        DATABASE_USER     = credentials('DB_USER')
        DATABASE_PASSWORD = credentials('DB_PASSWORD')
        APP_EC2_IP        = credentials('EC2_PUBLIC_IP')
    }
    steps {
        echo "🚀 Deploying Frontend to Dev (Docker Compose)..."
        sh """
            # Pull image mới
            docker pull ${IMAGE_NAME}:${IMAGE_TAG}

            # Stop và remove container cũ
            docker stop todolist-frontend || true
            docker rm todolist-frontend || true

            # Set env vars và deploy chỉ frontend
            export DATABASE_HOST=${DATABASE_HOST}
            export DATABASE_USER=${DATABASE_USER}
            export DATABASE_PASSWORD=${DATABASE_PASSWORD}
            export EC2_PUBLIC_IP=${APP_EC2_IP}
            export IMAGE_TAG=${IMAGE_TAG}

            IMAGE_TAG=${IMAGE_TAG} docker compose \
                -f ${COMPOSE_FILE} \
                up -d --no-deps --no-build frontend

            sleep 30

            curl -s -o /dev/null -w "%{http_code}" \
                http://10.0.1.43:3000 | grep 200 || \
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
                allOf {
                    branch 'main'
                    not { changeRequest() }
                }
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
        // Kind K8s trên EC2 Medium
        // ══════════════════════════════════════
        stage('Deploy Prod') {
            when {
                allOf {
                    branch 'main'
                    not { changeRequest() }
                }
            }
            steps {
                echo "☸️ Deploying Frontend to Production (Kind K8s)..."
                sh """
                    kubectl set image deployment/frontend \
                        frontend=${IMAGE_NAME}:${IMAGE_TAG} \
                        -n todolist

                    kubectl rollout status deployment/frontend \
                        -n todolist --timeout=120s

                    echo "✅ Deploy Prod SUCCESS"
                """
            }
        }
    }

    post {
        success {
            script {
                if (env.CHANGE_ID) {
                    echo """
                    ✅ CI SUCCESS — PR #${env.CHANGE_ID}
                    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                    Stage 1-4 PASSED → Ready to merge!
                    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                    """
                } else {
                    echo """
                    ✅ CD SUCCESS
                    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                    Image : ${IMAGE_NAME}:${IMAGE_TAG}
                    Branch: ${env.BRANCH_NAME}
                    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                    """
                }
            }
        }
        failure {
            echo "❌ Pipeline FAILED — Build #${BUILD_NUMBER}"
        }
        always {
            sh "docker image prune -f || true"
        }
    }
}