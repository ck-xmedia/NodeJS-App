pipeline {
  agent any
  options {
    timestamps()
    ansiColor('xterm')
    timeout(time: 30, unit: 'MINUTES')
  }
  environment {
    REPO_URL     = 'https://github.com/ck-xmedia/NodeJS-App.git'
    BRANCH_NAME  = 'jenkins-automation-20'
    APP_PORT     = '8080'
    VENV_DIR     = "${WORKSPACE}/.venv"
    NVM_DIR      = "${WORKSPACE}/.nvm"
    NODE_VERSION = '18'
    LOG_DIR      = "${WORKSPACE}/logs"
    LOG_FILE     = "${WORKSPACE}/logs/app_${BUILD_NUMBER}.log"
    PROJECT_TYPE = ''
  }

  stages {

    stage('Checkout') {
      steps {
        script {
          deleteDir()
        }
        git branch: "${BRANCH_NAME}", url: "${REPO_URL}"
      }
    }

    stage('Environment Setup') {
      steps {
        sh '''
          set -eu

          echo "[Env] Workspace: $WORKSPACE"
          echo "[Env] Preparing directories..."
          mkdir -p "$LOG_DIR"

          echo "[Env] Detecting Node..."
          if command -v node >/dev/null 2>&1 && command -v npm >/dev/null 2>&1; then
            echo "[Env] Node and npm already installed."
            node -v
            npm -v
          else
            echo "[Env] Installing nvm + Node ${NODE_VERSION}.x locally (no sudo)..."
            mkdir -p "${NVM_DIR}"
            if [ ! -s "${NVM_DIR}/nvm.sh" ]; then
              curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
            fi
            . "${NVM_DIR}/nvm.sh"
            nvm install ${NODE_VERSION}
            nvm use ${NODE_VERSION}
            node -v
            npm -v
          fi

          echo "[Env] Final PATH: $PATH"
        '''
      }
    }

    stage('Install Dependencies') {
      steps {
        script {
          // Detect project type by common dependency files
          if (fileExists('package.json')) {
            env.PROJECT_TYPE = 'node'
          } else if (fileExists('requirements.txt') || fileExists('pyproject.toml')) {
            env.PROJECT_TYPE = 'python'
          } else if (fileExists('pom.xml')) {
            env.PROJECT_TYPE = 'java-maven'
          } else if (fileExists('build.gradle') || fileExists('gradlew')) {
            env.PROJECT_TYPE = 'java-gradle'
          } else if (fileExists('go.mod')) {
            env.PROJECT_TYPE = 'go'
          } else if (fileExists('Gemfile')) {
            env.PROJECT_TYPE = 'ruby'
          } else {
            env.PROJECT_TYPE = 'unknown'
          }

          echo "Detected project type: ${env.PROJECT_TYPE}"

          if (env.PROJECT_TYPE == 'node') {
            sh '''
              set -eu
              # Ensure nvm context for Node commands if needed
              if ! command -v node >/dev/null 2>&1 || ! command -v npm >/dev/null 2>&1; then
                . "${NVM_DIR}/nvm.sh"
                nvm install ${NODE_VERSION}
                nvm use ${NODE_VERSION}
              fi

              echo "[Deps] Installing Node dependencies..."
              if [ -f package-lock.json ]; then
                npm ci
              else
                npm install --no-audit --no-fund
              fi

              echo "[Deps] Verifying installation..."
              node -e "require('fs').accessSync('node_modules')"
              echo "[Deps] OK"
            '''
            // Optional: run tests if defined
            sh '''
              set -eu
              echo "[Test] Checking for npm test script..."
              if npm run | grep -E -q " test\\b"; then
                echo "[Test] Running tests..."
                npm test
              else
                echo "[Test] No test script defined; skipping."
              fi
            '''
          } else if (env.PROJECT_TYPE == 'python') {
            sh '''
              set -eu
              echo "[Deps] Python project detected."
              python3 -V || true
              python -V || true

              echo "[Deps] Using virtual environment at ${VENV_DIR}"
              python3 -m venv "${VENV_DIR}" || python -m venv "${VENV_DIR}"
              . "${VENV_DIR}/bin/activate"

              if [ -f requirements.txt ]; then
                pip install --upgrade pip
                pip install -r requirements.txt
              elif [ -f pyproject.toml ]; then
                pip install --upgrade pip
                if grep -qi "\\[tool.poetry\\]" pyproject.toml; then
                  pip install poetry && poetry install --no-interaction --no-ansi
                else
                  pip install build && python -m build
                fi
              else
                echo "[Deps] No recognized Python dependency file found."
              fi
            '''
          } else if (env.PROJECT_TYPE == 'java-maven') {
            sh '''
              set -eu
              echo "[Deps] Maven project detected."
              mvn -v
              mvn -B -e -U clean package
            '''
          } else if (env.PROJECT_TYPE == 'java-gradle') {
            sh '''
              set -eu
              echo "[Deps] Gradle project detected."
              if [ -x "./gradlew" ]; then
                ./gradlew clean build --no-daemon
              else
                gradle clean build --no-daemon
              fi
            '''
          } else if (env.PROJECT_TYPE == 'go') {
            sh '''
              set -eu
              echo "[Deps] Go project detected."
              go version
              go mod download
              go build ./...
            '''
          } else {
            error "Unknown project type. No recognized dependency file found."
          }
        }
      }
    }

    stage('Deploy Application') {
      steps {
        script {
          if (env.PROJECT_TYPE == 'node') {
            sh '''
              set -eu

              echo "[Deploy] Ensuring logs directory exists..."
              mkdir -p "${LOG_DIR}"
              rm -f "${LOG_FILE}"

              echo "[Deploy] Killing existing process on port ${APP_PORT} (if any)..."
              if command -v fuser >/dev/null 2>&1; then
                fuser -k "${APP_PORT}"/tcp || true
              fi
              if command -v lsof >/dev/null 2>&1; then
                lsof -ti :"${APP_PORT}" | xargs -r kill -9 || true
              fi

              echo "[Deploy] Starting application with nohup on port ${APP_PORT}..."
              # Ensure nvm context for Node commands if needed
              if ! command -v node >/dev/null 2>&1 || ! command -v npm >/dev/null 2>&1; then
                . "${NVM_DIR}/nvm.sh"
                nvm use ${NODE_VERSION} >/dev/null
              fi

              nohup env PORT="${APP_PORT}" npm start > "${LOG_FILE}" 2>&1 & echo $! > "${LOG_DIR}/app.pid"
              sleep 2

              echo "[Deploy] Verifying application startup..."
              STARTED=0
              for i in $(seq 1 30); do
                if command -v curl >/dev/null 2>&1; then
                  if curl -fsS "http://127.0.0.1:${APP_PORT}/ready" >/dev/null 2>&1 || curl -fsS "http://127.0.0.1:${APP_PORT}/" >/dev/null 2>&1; then
                    STARTED=1
                    break
                  fi
                elif command -v wget >/dev/null 2>&1; then
                  if wget -qO- "http://127.0.0.1:${APP_PORT}/ready" >/dev/null 2>&1 || wget -qO- "http://127.0.0.1:${APP_PORT}/" >/dev/null 2>&1; then
                    STARTED=1
                    break
                  fi
                else
                  echo "[Deploy] Neither curl nor wget available to perform healthcheck."
                  break
                fi
                sleep 1
              done

              if [ "$STARTED" -ne 1 ]; then
                echo "[Deploy] Application did not become healthy in time."
                echo "----- Last 200 log lines -----"
                tail -n 200 "${LOG_FILE}" || true
                exit 1
              fi

              echo "[Deploy] Application is up on port ${APP_PORT}."
              echo "----- Last 50 log lines -----"
              tail -n 50 "${LOG_FILE}" || true
            '''
          } else if (env.PROJECT_TYPE == 'python') {
            sh '''
              set -eu
              echo "[Deploy] Python app deployment selected."
              mkdir -p "${LOG_DIR}"
              rm -f "${LOG_FILE}"

              echo "[Deploy] Killing existing process on port ${APP_PORT} (if any)..."
              if command -v fuser >/dev/null 2>&1; then
                fuser -k "${APP_PORT}"/tcp || true
              fi
              if command -v lsof >/dev/null 2>&1; then
                lsof -ti :"${APP_PORT}" | xargs -r kill -9 || true
              fi

              . "${VENV_DIR}/bin/activate" || true
              if [ -f app.py ]; then
                APP_ENTRY="app.py"
              elif [ -f server.py ]; then
                APP_ENTRY="server.py"
              else
                echo "[Deploy] No obvious Python entrypoint found (app.py/server.py)."
                exit 1
              fi

              nohup env PORT="${APP_PORT}" python "$APP_ENTRY" > "${LOG_FILE}" 2>&1 & echo $! > "${LOG_DIR}/app.pid"
              sleep 2

              echo "[Deploy] Verifying application startup..."
              STARTED=0
              for i in $(seq 1 30); do
                if command -v curl >/dev/null 2>&1; then
                  if curl -fsS "http://127.0.0.1:${APP_PORT}/" >/dev/null 2>&1; then
                    STARTED=1
                    break
                  fi
                elif command -v wget >/dev/null 2>&1; then
                  if wget -qO- "http://127.0.0.1:${APP_PORT}/" >/dev/null 2>&1; then
                    STARTED=1
                    break
                  fi
                else
                  echo "[Deploy] Neither curl nor wget available to perform healthcheck."
                  break
                fi
                sleep 1
              done
              if [ "$STARTED" -ne 1 ]; then
                echo "[Deploy] Application did not become healthy in time."
                tail -n 200 "${LOG_FILE}" || true
                exit 1
              fi
              echo "[Deploy] Python app is up on port ${APP_PORT}."
            '''
          } else if (env.PROJECT_TYPE == 'java-maven' || env.PROJECT_TYPE == 'java-gradle') {
            sh '''
              set -eu
              echo "[Deploy] Java app deployment selected."
              mkdir -p "${LOG_DIR}"
              rm -f "${LOG_FILE}"

              echo "[Deploy] Killing existing process on port ${APP_PORT} (if any)..."
              if command -v fuser >/dev/null 2>&1; then
                fuser -k "${APP_PORT}"/tcp || true
              fi
              if command -v lsof >/dev/null 2>&1; then
                lsof -ti :"${APP_PORT}" | xargs -r kill -9 || true
              fi

              JAR_FILE="$(ls -1 target/*.jar 2>/dev/null || true)"
              if [ -z "$JAR_FILE" ]; then
                JAR_FILE="$(ls -1 build/libs/*.jar 2>/dev/null || true)"
              fi
              if [ -z "$JAR_FILE" ]; then
                echo "[Deploy] No JAR artifact found to run."
                exit 1
              fi

              nohup env SERVER_PORT="${APP_PORT}" java -jar "$JAR_FILE" > "${LOG_FILE}" 2>&1 & echo $! > "${LOG_DIR}/app.pid"
              sleep 2

              echo "[Deploy] Verifying application startup..."
              STARTED=0
              for i in $(seq 1 30); do
                if command -v curl >/dev/null 2>&1; then
                  if curl -fsS "http://127.0.0.1:${APP_PORT}/" >/dev/null 2>&1; then
                    STARTED=1
                    break
                  fi
                elif command -v wget >/dev/null 2>&1; then
                  if wget -qO- "http://127.0.0.1:${APP_PORT}/" >/dev/null 2>&1; then
                    STARTED=1
                    break
                  fi
                else
                  echo "[Deploy] Neither curl nor wget available to perform healthcheck."
                  break
                fi
                sleep 1
              done
              if [ "$STARTED" -ne 1 ]; then
                echo "[Deploy] Application did not become healthy in time."
                tail -n 200 "${LOG_FILE}" || true
                exit 1
              fi
              echo "[Deploy] Java app is up on port ${APP_PORT}."
            '''
          } else if (env.PROJECT_TYPE == 'go') {
            sh '''
              set -eu
              echo "[Deploy] Go app deployment selected."
              mkdir -p "${LOG_DIR}"
              rm -f "${LOG_FILE}"

              echo "[Deploy] Killing existing process on port ${APP_PORT} (if any)..."
              if command -v fuser >/dev/null 2>&1; then
                fuser -k "${APP_PORT}"/tcp || true
              fi
              if command -v lsof >/dev/null 2>&1; then
                lsof -ti :"${APP_PORT}" | xargs -r kill -9 || true
              fi

              APP_BIN="./app"
              if [ -f main.go ]; then
                go build -o "${APP_BIN}" .
              elif ls -1 *.go >/dev/null 2>&1; then
                go build -o "${APP_BIN}" ./...
              fi

              nohup env PORT="${APP_PORT}" "${APP_BIN}" > "${LOG_FILE}" 2>&1 & echo $! > "${LOG_DIR}/app.pid"
              sleep 2

              echo "[Deploy] Verifying application startup..."
              STARTED=0
              for i in $(seq 1 30); do
                if command -v curl >/dev/null 2>&1; then
                  if curl -fsS "http://127.0.0.1:${APP_PORT}/" >/dev/null 2>&1; then
                    STARTED=1
                    break
                  fi
                elif command -v wget >/dev/null 2>&1; then
                  if wget -qO- "http://127.0.0.1:${APP_PORT}/" >/dev/null 2>&1; then
                    STARTED=1
                    break
                  fi
                else
                  echo "[Deploy] Neither curl nor wget available to perform healthcheck."
                  break
                fi
                sleep 1
              done
              if [ "$STARTED" -ne 1 ]; then
                echo "[Deploy] Application did not become healthy in time."
                tail -n 200 "${LOG_FILE}" || true
                exit 1
              fi
              echo "[Deploy] Go app is up on port ${APP_PORT}."
            '''
          } else {
            error "Cannot deploy unknown project type."
          }
        }
      }
    }
  }

  post {
    always {
      script {
        echo "Archiving deployment logs (if any)..."
      }
      archiveArtifacts artifacts: 'logs/*.log', onlyIfSuccessful: false, allowEmptyArchive: true
      script {
        if (fileExists('logs/app.pid')) {
          echo "Current app PID: " + readFile('logs/app.pid').trim()
        }
      }
    }
    failure {
      script {
        if (fileExists("${env.LOG_FILE}")) {
          echo "----- Deployment Log Tail -----"
          sh "tail -n 200 '${env.LOG_FILE}' || true"
        }
      }
    }
  }
}