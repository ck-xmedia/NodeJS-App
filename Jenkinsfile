pipeline {
  agent { label 'linux' }

  environment {
    REPO_URL    = 'https://github.com/ck-xmedia/NodeJS-App.git'
    BRANCH_NAME = 'jenkins-automation-24'
    APP_PORT    = '8080'
    LOG_DIR     = "${WORKSPACE}/logs"
    APP_NAME    = 'nodejs-app'
  }

  stages {
    stage('Checkout') {
      steps {
        deleteDir()
        git url: "${REPO_URL}", branch: "${BRANCH_NAME}"
      }
    }

    stage('Install Dependencies') {
      steps {
        sh 'npm ci'
      }
    }

    stage('Test') {
      steps {
        sh 'npm test --if-present'
      }
    }

    stage('Deploy') {
      steps {
        sh '''
          set -e
          mkdir -p "$LOG_DIR"
          if pm2 describe "$APP_NAME" > /dev/null 2>&1; then
            pm2 reload "$APP_NAME" --update-env
          else
            PORT="$APP_PORT" NODE_ENV=production pm2 start index.js \
              --name "$APP_NAME" \
              --output "$LOG_DIR/$APP_NAME-out.log" \
              --error "$LOG_DIR/$APP_NAME-err.log" \
              --time
          fi
          pm2 save || true
        '''
      }
    }
  }
}