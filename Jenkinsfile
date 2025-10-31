pipeline {
  agent { label 'linux' }

  tools {
    // configure based on available Jenkins tools
  }

  environment {
    REPO_URL    = 'https://github.com/ck-xmedia/NodeJS-App.git'
    BRANCH_NAME = 'jenkins-automation-23'
    APP_PORT    = '8080'
    LOG_DIR     = "${WORKSPACE}/logs"
    APP_NAME    = 'nodejs-app'
  }

  options {
    skipDefaultCheckout(true)
    buildDiscarder(logRotator(numToKeepStr: '10'))
    timestamps()
  }

  stages {
    stage('Checkout') {
      steps {
        deleteDir()
        git branch: env.BRANCH_NAME, url: env.REPO_URL
      }
    }

    stage('Install Dependencies') {
      steps {
        sh '''
if [ -f package.json ]; then
  npm ci --no-audit --no-fund
elif [ -f requirements.txt ]; then
  pip install -r requirements.txt
elif [ -f pom.xml ]; then
  mvn -B -DskipTests package
else
  echo "No known dependency manifest found. Skipping install."
fi
'''
      }
    }

    stage('Test') {
      steps {
        sh '''
if [ -f package.json ]; then
  if node -e "const p=require('./package.json');process.exit(p.scripts&&p.scripts.test?0:1)"; then
    npm test
  else
    echo "No test script found. Skipping."
  fi
elif [ -f pom.xml ]; then
  mvn -B test
elif [ -f pytest.ini ] || [ -d tests ]; then
  if command -v pytest >/dev/null 2>&1; then
    pytest -q
  else
    echo "pytest not installed. Skipping."
  fi
else
  echo "No tests detected. Skipping."
fi
'''
      }
    }

    stage('Deploy') {
      when { expression { return fileExists('package.json') } }
      steps {
        sh '''
mkdir -p "$LOG_DIR"
export PORT="$APP_PORT"

if command -v pm2 >/dev/null 2>&1; then
  pm2 stop "$APP_NAME" || true
  pm2 start index.js --name "$APP_NAME" --update-env --time --log "$LOG_DIR/$APP_NAME.log"
  pm2 save || true
else
  npx pm2 stop "$APP_NAME" || true
  npx pm2 start index.js --name "$APP_NAME" --update-env --time --log "$LOG_DIR/$APP_NAME.log"
  npx pm2 save || true
fi
'''
      }
    }
  }

  post {
    always {
      sh 'mkdir -p "$LOG_DIR"'
      archiveArtifacts artifacts: 'logs/**/*.log', allowEmptyArchive: true
    }
  }
}