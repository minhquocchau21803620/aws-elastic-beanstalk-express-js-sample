pipeline {
  agent any
  options { ansiColor('xterm'); timestamps() }
  environment {
    DOCKER_HOST = 'tcp://docker:2376'
    DOCKER_TLS_VERIFY = '1'
    DOCKER_CERT_PATH = '/certs/client'
  }
  stages {
    stage('Checkout') { steps { checkout scm } }
    stage('Install deps (Node 16)') {
      steps {
        sh '''
          echo "DOCKER_HOST=$DOCKER_HOST"
          ls -l $DOCKER_CERT_PATH

          docker run --rm -v "$WORKSPACE":/app -w /app node:16 bash -lc "
            node --version && npm --version &&
            npm install --save
          "
        '''
      }
    }
  }
  post { always { cleanWs() } }
}

