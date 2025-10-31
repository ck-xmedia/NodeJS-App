pipeline {
  agent { label 'linux' }

  tools {
    // Configure based on available Jenkins tools
    // Example: nodejs 'NodeJS'
  }

  environment {
    REPO_URL    = 'https://github.com/ck-xmedia/NodeJS-App.git'
    BRANCH_NAME = 'jenkins-automation-21'
    APP_PORT    = '8080'
    LOG_DIR     = "${WORKSPACE}/logs"
    APP_NAME    = 'nodejs-app'
  }

  stages {
    stage('Checkout') {
      steps {
        deleteDir()
        git branch: "${BRANCH_NAME}", url: "${REPO_URL}"
      }
    }

    stage('Install Dependencies') {
      when { expression { fileExists('package.json') } }
      steps {
        sh 'npm ci'
      }
    }

    stage('Test') {
      when { expression { fileExists('package.json') } }
      steps {
        sh 'npm run -s test --if-present'
      }
    }

    stage('Deploy') {
      steps {
        sh '''
set -e
mkdir -p "${LOG_DIR}"

if [ -f package.json ]; then
  export PORT="${APP_PORT}"
  if command -v pm2 >/dev/null 2>&1; then
    if pm2 describe "${APP_NAME}" >/dev/null 2>&1; then
      pm2 restart "${APP_NAME}" --update-env
    else
      pm2 start index.js --name "${APP_NAME}" -o "${LOG_DIR}/out.log" -e "${LOG_DIR}/err.log"
    fi
    pm2 save || true
  else
    echo "pm2 not found on agent. Please install/configure pm2 on the node."
    exit 1
  fi
else
  echo "Unknown project type. Add deployment steps for your stack."
fi
'''
      }
    }
  }
}