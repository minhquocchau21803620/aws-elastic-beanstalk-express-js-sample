pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
    skipDefaultCheckout(false)
  }

  environment {
    // DinD over TLS (from your docker-compose)
    DOCKER_HOST       = 'tcp://docker:2376'
    DOCKER_TLS_VERIFY = '1'
    DOCKER_CERT_PATH  = '/certs/client'

    // Local tag used during build
    APP_IMAGE  = 'app-ci-image:build'
    // Repo name on Docker Hub (user part will be injected from credentials)
    IMAGE_NAME = 'aws-eb-express-sample'

    // Jenkins credentials id for Docker Hub
    DOCKERHUB_CREDENTIALS_ID = 'dockerhub'
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        sh 'echo "Workspace: $WORKSPACE" && ls -la'
      }
    }

    stage('Install deps (Node 16)') {
      steps {
        sh '''
          docker run --rm \
            -v "$WORKSPACE":/app -w /app \
            node:16 bash -lc '
              node -v && npm -v &&
              npm install --save
            '
        '''
      }
    }

    stage('Unit Tests (Node 16)') {
      steps {
        sh '''
          docker run --rm \
            -v "$WORKSPACE":/app -w /app \
            node:16 bash -lc '
              npm test --if-present
            '
        '''
      }
    }

    stage('Dependency Scan (OWASP DC)') {
      steps {
        sh '''
          rm -rf dependency-check-report
          mkdir -p dependency-check-report

          docker run --rm \
            -v "$WORKSPACE":/src \
            owasp/dependency-check:latest \
              --scan /src \
              --format "HTML" --format "JSON" \
              --project "aws-eb-express-sample" \
              --out /src/dependency-check-report \
              --failOnCVSS 7
        '''
      }
    }

    stage('Build Docker Image') {
      steps {
        sh '''
          # Create a minimal Dockerfile if repo has none
          if [ ! -f Dockerfile ]; then
            cat > Dockerfile <<'EOF'
FROM node:16-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
EXPOSE 8080
CMD ["npm","start"]
EOF
          fi

          docker build -t "$APP_IMAGE" .
        '''
      }
    }

    stage('Push Image') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: "${DOCKERHUB_CREDENTIALS_ID}",
          usernameVariable: 'DOCKER_USERNAME',
          passwordVariable: 'DOCKER_PASSWORD'
        )]) {
          sh '''
            set -eux
            echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
            REPO="${DOCKER_USERNAME}/${IMAGE_NAME}"
            docker tag "$APP_IMAGE" "$REPO:${BUILD_NUMBER}"
            docker tag "$APP_IMAGE" "$REPO:latest"
            docker push "$REPO:${BUILD_NUMBER}"
            docker push "$REPO:latest"
          '''
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'dependency-check-report/**', allowEmptyArchive: true
      cleanWs()
    }
  }
}

