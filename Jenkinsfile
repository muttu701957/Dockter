pipeline {

    agent any

    environment {

        AWS_ACCESS_KEY_ID     = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')

        AWS_REGION            = credentials('AWS_REGION')
        AWS_ACCOUNT_ID        = credentials('AWS_ACCOUNT_ID')

        EC2_USER              = credentials('EC2_USER')
        EC2_PUBLIC_IP         = credentials('EC2_PUBLIC_IP')

        ecr_repo_back         = credentials('ecr_repo_back')
        ecr_repo_front        = credentials('ecr_repo_front')

        // ------------------------------------------------------------
        // Rollback / health-check trigger logic — explicit, no magic numbers
        // ------------------------------------------------------------
        HEALTH_CHECK_ENDPOINT   = '/'     // root endpoint, checked on both services
        HEALTH_CHECK_STATUS     = '200'   // expected HTTP status for a healthy app
        HEALTH_CHECK_RETRIES    = '10'    // max probe attempts before declaring unhealthy
        HEALTH_CHECK_INTERVAL   = '5'     // seconds between retries
        // Total wait time = RETRIES * INTERVAL = 50s

        BACKEND_PORT            = '4000'
        FRONTEND_PORT           = '5174'
    }

    stages {

        stage('Checkout') {
            steps {
                git(
                    branch: 'main',
                    url: 'https://github.com/muttu701957/Dockter.git',
                    credentialsId: 'github-token'
                )
            }
        }

        stage('Detect Changes') {

            steps {

                script {

                    def changedFiles = sh(
                        script: '''
                        git diff --name-only HEAD~1 HEAD 2>/dev/null || git ls-files
                        ''',
                        returnStdout: true
                    ).trim()

                    echo changedFiles

                    def BACKEND_CHANGED = false
                    def FRONTEND_CHANGED = false

                    if (changedFiles.contains("backend/")) {
                        BACKEND_CHANGED = true
                    }

                    if (changedFiles.contains("frontend/")) {
                        FRONTEND_CHANGED = true
                    }

                    if (changedFiles.contains("Jenkinsfile")) {
                        BACKEND_CHANGED = true
                        FRONTEND_CHANGED = true
                    }

                    env.BACKEND_CHANGED = "${BACKEND_CHANGED}"
                    env.FRONTEND_CHANGED = "${FRONTEND_CHANGED}"

                    echo "Backend Changed = ${BACKEND_CHANGED}"
                    echo "Frontend Changed = ${FRONTEND_CHANGED}"

                }

            }

        }

        stage('Install Backend Dependencies') {

            when {
                expression { env.BACKEND_CHANGED == "true" }
            }

            steps {
                dir('backend') {
                    sh '''
                    npm install
                    '''
                }
            }

        }

        stage('Install Frontend Dependencies') {

            when {
                expression { env.FRONTEND_CHANGED == "true" }
            }

            steps {
                dir('frontend') {
                    sh '''
                    npm install
                    '''
                }
            }

        }

        stage('AWS Login') {

            when {
                anyOf {
                    expression { env.BACKEND_CHANGED == "true" }
                    expression { env.FRONTEND_CHANGED == "true" }
                }
            }

            steps {
                script {

                    env.ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

                    sh '''
                    aws configure set AWS_ACCESS_KEY_ID $AWS_ACCESS_KEY_ID
                    aws configure set AWS_SECRET_ACCESS_KEY $AWS_SECRET_ACCESS_KEY
                    aws configure set default.region $AWS_REGION

                    aws ecr get-login-password --region $AWS_REGION | \
                    docker login \
                    --username AWS \
                    --password-stdin \
                    $ECR_REGISTRY
                    '''

                }
            }

        }

        stage('Build Backend Image') {

            when {
                expression { env.BACKEND_CHANGED == "true" }
            }

            steps {
                script {

                    env.BACKEND_IMAGE_LATEST = "${ECR_REGISTRY}/${ecr_repo_back}:backend-latest"
                    env.BACKEND_IMAGE        = "${ECR_REGISTRY}/${ecr_repo_back}:backend-${BUILD_NUMBER}"

                    sh """
                    docker build \
                    -f backend/Dockerfile \
                    -t ${BACKEND_IMAGE} \
                    -t ${BACKEND_IMAGE_LATEST} \
                    backend

                    docker push ${BACKEND_IMAGE}
                    docker push ${BACKEND_IMAGE_LATEST}
                    """

                }
            }

        }

        stage('Build Frontend Image') {

            when {
                expression { env.FRONTEND_CHANGED == "true" }
            }

            steps {
                script {

                    env.FRONTEND_IMAGE_LATEST = "${ECR_REGISTRY}/${ecr_repo_front}:frontend-latest"
                    env.FRONTEND_IMAGE        = "${ECR_REGISTRY}/${ecr_repo_front}:frontend-${BUILD_NUMBER}"

                    sh """
                    docker build \
                    -f frontend/Dockerfile \
                    -t ${FRONTEND_IMAGE} \
                    -t ${FRONTEND_IMAGE_LATEST} \
                    frontend

                    docker push ${FRONTEND_IMAGE}
                    docker push ${FRONTEND_IMAGE_LATEST}
                    """

                }
            }

        }

        stage('Deploy Backend') {

            when {
                expression { env.BACKEND_CHANGED == "true" }
            }

            steps {
                withCredentials([
                    string(credentialsId: 'EC2_SSH_KEY_B64', variable: 'EC2_SSH_KEY_B64')
                ]) {

                    sh """
mkdir -p ~/.ssh
echo "\$EC2_SSH_KEY_B64" | base64 -d > ~/.ssh/id_rsa
chmod 600 ~/.ssh/id_rsa

ssh -i ~/.ssh/id_rsa \
-o StrictHostKeyChecking=no \
${EC2_USER}@${EC2_PUBLIC_IP} << 'REMOTE'

set -e

DEPLOY_DIR=/home/ubuntu/Dockter/backend
STATE_FILE=\$DEPLOY_DIR/.deployed_tag
NEW_TAG="${BACKEND_IMAGE}"
PREV_TAG=\$(cat "\$STATE_FILE" 2>/dev/null || echo "")

aws ecr get-login-password --region ${AWS_REGION} | \
docker login --username AWS --password-stdin ${ECR_REGISTRY}

docker pull "\$NEW_TAG"

docker stop dock-back || true
docker rm dock-back || true

docker run -dit \
--name dock-back \
--restart unless-stopped \
-p ${BACKEND_PORT}:${BACKEND_PORT} \
--env-file \$DEPLOY_DIR/.env \
"\$NEW_TAG"

HEALTHY=false
for i in \$(seq 1 ${HEALTH_CHECK_RETRIES}); do
  CODE=\$(curl -s -o /dev/null -w "%{http_code}" \
          http://localhost:${BACKEND_PORT}${HEALTH_CHECK_ENDPOINT} || echo "000")
  echo "Health check attempt \$i/${HEALTH_CHECK_RETRIES}: HTTP \$CODE"

  if [ "\$CODE" = "${HEALTH_CHECK_STATUS}" ]; then
    HEALTHY=true
    break
  fi
  sleep ${HEALTH_CHECK_INTERVAL}
done

if [ "\$HEALTHY" = "true" ]; then
  echo "\$NEW_TAG" > "\$STATE_FILE"
  echo "Backend healthy. Deployed \$NEW_TAG"
else
  echo "Backend health check FAILED after ${HEALTH_CHECK_RETRIES} attempts (~\$(( ${HEALTH_CHECK_RETRIES} * ${HEALTH_CHECK_INTERVAL} ))s). Rolling back..."

  if [ -n "\$PREV_TAG" ]; then
    docker pull "\$PREV_TAG"
    docker stop dock-back || true
    docker rm dock-back || true
    docker run -dit \
    --name dock-back \
    --restart unless-stopped \
    -p ${BACKEND_PORT}:${BACKEND_PORT} \
    --env-file \$DEPLOY_DIR/.env \
    "\$PREV_TAG"

    ROLLBACK_CODE=\$(curl -s -o /dev/null -w "%{http_code}" \
                    http://localhost:${BACKEND_PORT}${HEALTH_CHECK_ENDPOINT} || echo "000")

    if [ "\$ROLLBACK_CODE" = "${HEALTH_CHECK_STATUS}" ]; then
      echo "Rolled back to \$PREV_TAG successfully — it is healthy."
    else
      echo "ROLLBACK IMAGE ALSO UNHEALTHY (HTTP \$ROLLBACK_CODE). Manual intervention required."
    fi
  else
    echo "No previous successful deployment recorded in \$STATE_FILE — cannot auto-rollback. Manual intervention required."
  fi

  docker image prune -f
  exit 1
fi

docker image prune -f

REMOTE
"""

                }
            }

        }

        stage('Deploy Frontend') {

            when {
                expression { env.FRONTEND_CHANGED == "true" }
            }

            steps {
                withCredentials([
                    string(credentialsId: 'EC2_SSH_KEY_B64', variable: 'EC2_SSH_KEY_B64')
                ]) {

                    sh """
mkdir -p ~/.ssh
echo "\$EC2_SSH_KEY_B64" | base64 -d > ~/.ssh/id_rsa
chmod 600 ~/.ssh/id_rsa

ssh -i ~/.ssh/id_rsa \
-o StrictHostKeyChecking=no \
${EC2_USER}@${EC2_PUBLIC_IP} << 'REMOTE'

set -e

DEPLOY_DIR=/home/ubuntu/Dockter/frontend
STATE_FILE=\$DEPLOY_DIR/.deployed_tag
NEW_TAG="${FRONTEND_IMAGE}"
PREV_TAG=\$(cat "\$STATE_FILE" 2>/dev/null || echo "")

mkdir -p \$DEPLOY_DIR

aws ecr get-login-password --region ${AWS_REGION} | \
docker login --username AWS --password-stdin ${ECR_REGISTRY}

docker pull "\$NEW_TAG"

docker stop dock || true
docker rm dock || true

docker run -dit \
--name dock \
--restart unless-stopped \
-p ${FRONTEND_PORT}:80 \
"\$NEW_TAG"

HEALTHY=false
for i in \$(seq 1 ${HEALTH_CHECK_RETRIES}); do
  CODE=\$(curl -s -o /dev/null -w "%{http_code}" \
          http://localhost:${FRONTEND_PORT}${HEALTH_CHECK_ENDPOINT} || echo "000")
  echo "Health check attempt \$i/${HEALTH_CHECK_RETRIES}: HTTP \$CODE"

  if [ "\$CODE" = "${HEALTH_CHECK_STATUS}" ]; then
    HEALTHY=true
    break
  fi
  sleep ${HEALTH_CHECK_INTERVAL}
done

if [ "\$HEALTHY" = "true" ]; then
  echo "\$NEW_TAG" > "\$STATE_FILE"
  echo "Frontend healthy. Deployed \$NEW_TAG"
else
  echo "Frontend health check FAILED after ${HEALTH_CHECK_RETRIES} attempts (~\$(( ${HEALTH_CHECK_RETRIES} * ${HEALTH_CHECK_INTERVAL} ))s). Rolling back..."

  if [ -n "\$PREV_TAG" ]; then
    docker pull "\$PREV_TAG"
    docker stop dock || true
    docker rm dock || true
    docker run -dit \
    --name dock \
    --restart unless-stopped \
    -p ${FRONTEND_PORT}:80 \
    "\$PREV_TAG"

    ROLLBACK_CODE=\$(curl -s -o /dev/null -w "%{http_code}" \
                    http://localhost:${FRONTEND_PORT}${HEALTH_CHECK_ENDPOINT} || echo "000")

    if [ "\$ROLLBACK_CODE" = "${HEALTH_CHECK_STATUS}" ]; then
      echo "Rolled back to \$PREV_TAG successfully — it is healthy."
    else
      echo "ROLLBACK IMAGE ALSO UNHEALTHY (HTTP \$ROLLBACK_CODE). Manual intervention required."
    fi
  else
    echo "No previous successful deployment recorded in \$STATE_FILE — cannot auto-rollback. Manual intervention required."
  fi

  docker image prune -f
  exit 1
fi

docker image prune -f

REMOTE
"""

                }
            }

        }

    }

    post {

        success {
            echo "Deployment Completed Successfully"
        }

        failure {
            echo "Deployment Failed — check console output above for health-check / rollback details."
        }

        always {
            cleanWs()
        }

    }

}
