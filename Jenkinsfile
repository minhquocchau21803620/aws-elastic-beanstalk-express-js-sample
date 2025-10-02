pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    // Using Node 16 Docker image to install dependencies
    stage('Install deps (Node 16)') {
      steps {
        sh '''
          docker run --rm -v "$PWD":/app -w /app node:16 bash -lc "
            node --version && npm --version &&
            npm install --save
          "
        '''
      }
    }
  }
}
