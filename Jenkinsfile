pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
    skipDefaultCheckout(false)
  }

  environment {
    // These come from your docker-compose jenkins service,
    // but we set them here explicitly so every stage sees them.
    DOCKER_HOST      = 'tcp://docker:2376'
    DOCKER_TLS_VERIFY= '1'
    DOCKER_CERT_PATH = '/certs/client'

    // Image names/tags
    APP_IMAGE        = "app-ci-image:build"     // local tag during build
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
          # Create a minimal Dockerfile if the repo doesn't have one
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
      when { expression { return env.DOCKERHUB_CREDENTIALS_ID } }  // only if you set credentials ID from job
      steps {
        withCredentials([usernamePassword(
          credentialsId: "${DOCKERHUB_CREDENTIALS_ID}",
          usernameVariable: 'DOCKER_USERNAME',
          passwordVariable: 'DOCKER_PASSWORD'
        )]) {
          sh '''
            echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
            IMAGE_REPO="${DOCKER_USERNAME}/aws-eb-express-sample"
            docker tag "$APP_IMAGE" "$IMAGE_REPO:${BUILD_NUMBER}"
            docker tag "$APP_IMAGE" "$IMAGE_REPO:latest"
            docker push "$IMAGE_REPO:${BUILD_NUMBER}"
            docker push "$IMAGE_REPO:latest"
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

