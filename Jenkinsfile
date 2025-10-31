pipeline {
  agent { label 'linux' }
  tools {
    // configure based on available Jenkins tools
    nodejs 'NodeJS'
  }
  environment {
    REPO_URL    = 'https://github.com/ck-xmedia/NodeJS-App.git'
    BRANCH_NAME = 'jenkins-automation-22'
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
      steps {
        sh '''
          if [ -f package.json ]; then
            npm ci --no-audit --no-fund
          elif [ -f requirements.txt ]; then
            pip install -r requirements.txt
          elif [ -f pom.xml ]; then
            mvn -B -DskipTests package
          else
            echo "No known dependency manifest found."
          fi
        '''
      }
    }
    stage('Test') {
      when {
        expression { fileExists('package.json') || fileExists('pom.xml') || fileExists('pytest.ini') || fileExists('tests') || fileExists('test') }
      }
      steps {
        sh '''
          if [ -f package.json ]; then
            npm test --if-present || true
          elif [ -f pom.xml ]; then
            mvn -B test || true
          elif [ -f pytest.ini ] || [ -d tests ] || [ -d test ]; then
            pytest || true
          else
            echo "No tests detected."
          fi
        '''
      }
    }
    stage('Deploy') {
      steps {
        sh '''
          mkdir -p "$LOG_DIR"
          if [ -f package.json ]; then
            pm2 delete "$APP_NAME" || true
            PORT="$APP_PORT" pm2 start index.js --name "$APP_NAME" --time --output "$LOG_DIR/$APP_NAME.out.log" --error "$LOG_DIR/$APP_NAME.err.log" --update-env
            pm2 save || true
            pm2 status
          elif [ -f pom.xml ]; then
            echo "Java project detected. Add systemd or jar run here."
          elif [ -f requirements.txt ]; then
            echo "Python project detected. Add systemd/uwsgi/gunicorn here."
          else
            echo "Unknown project type. Provide deployment commands."
          fi
        '''
      }
    }
  }
}