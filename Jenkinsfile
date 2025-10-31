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
        script {
          // Prefer a Jenkins-managed NodeJS tool if available to avoid network installs
          def nodeToolCandidates = [
            'NodeJS_18','Node18','node18','NodeJS','nodejs','Node 18','Node'
          ]
          for (def t : nodeToolCandidates) {
            try {
              def home = tool t
              if (home) {
                env.PATH = "${home}/bin:${env.PATH}"
                echo "[Env] Using Jenkins NodeJS tool '${t}' at ${home}"
                break
              }
            } catch (ignored) {
              // Try next candidate
            }
          }
        }
        script {
          if (isUnix()) {
            sh '''
              set -eu

              echo "[Env] Workspace: $WORKSPACE"
              echo "[Env] Preparing directories..."
              mkdir -p "$LOG_DIR"

              echo "[Env] Detecting Node..."
              if command -v node >/dev/null 2>&1 && command -v npm >/dev/null 2>&1; then
                echo "[Env] Node and npm already installed."
                node -v || true
                npm -v || true
              else
                echo "[Env] Node/npm not found. Attempting local nvm install without sudo..."
                mkdir -p "${NVM_DIR}"
                if [ ! -s "${NVM_DIR}/nvm.sh" ]; then
                  echo "[Env] nvm not found at ${NVM_DIR}/nvm.sh; attempting to install..."
                  if command -v git >/dev/null 2>&1; then
                    rm -rf "${NVM_DIR}.tmp" || true
                    git clone https://github.com/nvm-sh/nvm.git "${NVM_DIR}.tmp" || true
                    if [ -d "${NVM_DIR}.tmp/.git" ]; then
                      (cd "${NVM_DIR}.tmp" && git checkout "v0.39.7") || true
                      rm -rf "${NVM_DIR}" || true
                      mv "${NVM_DIR}.tmp" "${NVM_DIR}" || true
                    else
                      rm -rf "${NVM_DIR}.tmp" || true
                    fi
                  elif command -v curl >/dev/null 2>&1; then
                    export NVM_DIR="${NVM_DIR}"
                    (curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash) || true
                  elif command -v wget >/dev/null 2>&1; then
                    export NVM_DIR="${NVM_DIR}"
                    (wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash) || true
                  else
                    echo "[Env] No git/curl/wget available to install nvm; skipping nvm installation."
                  fi
                fi

                if [ -s "${NVM_DIR}/nvm.sh" ]; then
                  . "${NVM_DIR}/nvm.sh" || true
                  if command -v nvm >/dev/null 2>&1; then
                    nvm install ${NODE_VERSION} || true
                    nvm use ${NODE_VERSION} || true
                  else
                    echo "[Env] nvm command not available after sourcing; continuing without nvm."
                  fi
                else
                  echo "[Env] nvm.sh not found at ${NVM_DIR}/nvm.sh; continuing without nvm."
                fi

                if command -v node >/dev/null 2>&1 && command -v npm >/dev/null 2>&1; then
                  echo "[Env] Node/npm available after setup."
                  node -v || true
                  npm -v || true
                else
                  echo "[Env][WARN] Node/npm still not available; subsequent steps may fail."
                fi
              fi

              echo "[Env] Final PATH: $PATH"
            '''
          } else {
            bat '''
@echo off
setlocal EnableExtensions EnableDelayedExpansion

echo [Env] Workspace: %WORKSPACE%
echo [Env] Preparing directories...
if not exist "%LOG_DIR%" mkdir "%LOG_DIR%" 2>nul

echo [Env] Detecting Node...
where node >nul 2>nul && where npm >nul 2>nul
if errorlevel 1 (
  echo [Env][WARN] Node/npm not found. Skipping local install on Windows.
) else (
  node -v
  npm -v
)

echo [Env] Final PATH: %PATH%
endlocal
'''
          }
        }
      }
    }

    stage('Install Dependencies') {
      steps {
        script {
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
          } else {
            env.PROJECT_TYPE = 'unknown'
          }

          echo "Detected project type: ${env.PROJECT_TYPE}"

          if (env.PROJECT_TYPE == 'node') {
            if (isUnix()) {
              sh '''
                set -eu
                if ! command -v node >/dev/null 2>&1 || ! command -v npm >/dev/null 2>&1; then
                  if [ -s "${NVM_DIR}/nvm.sh" ]; then
                    . "${NVM_DIR}/nvm.sh" || true
                    command -v nvm >/dev/null 2>&1 && { nvm install ${NODE_VERSION} || true; nvm use ${NODE_VERSION} || true; }
                  fi
                fi

                if ! command -v node >/dev/null 2>&1 || ! command -v npm >/dev/null 2>&1; then
                  echo "[Deps][ERROR] Node.js and npm are required but not available."
                  exit 1
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
              sh '''
                set -eu
                echo "[Test] Checking for npm test script..."
                if node -e "process.exit(!((require('./package.json').scripts||{}).test))"; then
                  echo "[Test] Running tests..."
                  npm test || npm run test
                else
                  echo "[Test] No test script defined; skipping."
                fi
              '''
            } else {
              bat '''
@echo off
setlocal EnableExtensions EnableDelayedExpansion

where node >nul 2>nul && where npm >nul 2>nul
if errorlevel 1 (
  echo [Deps][ERROR] Node.js and npm are required but not available.
  exit /b 1
)

echo [Deps] Installing Node dependencies...
if exist package-lock.json (
  npm ci
) else (
  npm install --no-audit --no-fund
)
if errorlevel 1 (
  echo [Deps][ERROR] npm install failed.
  exit /b 1
)

echo [Deps] Verifying installation...
if exist node_modules (
  echo [Deps] OK
) else (
  echo [Deps][ERROR] node_modules not found
  exit /b 1
)

echo [Test] Checking for npm test script...
findstr /I /C:"\"test\":" package.json >nul 2>&1
if errorlevel 1 (
  echo [Test] No test script defined; skipping.
) else (
  echo [Test] Running tests...
  npm test
  if errorlevel 1 exit /b 1
)

endlocal
'''
            }
          } else if (env.PROJECT_TYPE == 'python') {
            sh '''
              set -eu
              echo "[Deps] Python project detected."
              python3 -V || true
              python -V || true

              echo "[Deps] Using virtual environment at ${VENV_DIR}"
              python3 -m venv "${VENV_DIR}" || python -m venv "${VENV_DIR}" || true
              if [ -f "${VENV_DIR}/bin/activate" ]; then
                . "${VENV_DIR}/bin/activate"
              fi

              if [ -f requirements.txt ]; then
                pip install --upgrade pip || true
                pip install -r requirements.txt || true
              elif [ -f pyproject.toml ]; then
                pip install --upgrade pip || true
                if grep -qi "\\[tool.poetry\\]" pyproject.toml; then
                  pip install poetry && poetry install --no-interaction --no-ansi || true
                else
                  pip install build && python -m build || true
                fi
              else
                echo "[Deps] No recognized Python dependency file found."
              fi
            '''
          } else if (env.PROJECT_TYPE == 'java-maven') {
            sh '''
              set -eu
              echo "[Deps] Maven project detected."
              mvn -v || true
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
              go version || true
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
            if (isUnix()) {
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
                  PIDS="$(lsof -ti :"${APP_PORT}" || true)"
                  if [ -n "${PIDS:-}" ]; then
                    echo "$PIDS" | xargs kill -9 || true
                  fi
                fi

                echo "[Deploy] Starting application with nohup on port ${APP_PORT}..."
                if ! command -v node >/dev/null 2>&1 || ! command -v npm >/dev/null 2>&1; then
                  if [ -s "${NVM_DIR}/nvm.sh" ]; then
                    . "${NVM_DIR}/nvm.sh" || true
                    command -v nvm >/dev/null 2>&1 && nvm use ${NODE_VERSION} >/dev/null || true
                  fi
                fi

                nohup env PORT="${APP_PORT}" npm start > "${LOG_FILE}" 2>&1 & echo $! > "${LOG_DIR}/app.pid" || true
                sleep 2

                echo "[Deploy] Verifying application startup..."
                STARTED=0
                i=1
                while [ "$i" -le 30 ]; do
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
                  i=$((i+1))
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
            } else {
              bat '''
@echo off
setlocal EnableExtensions EnableDelayedExpansion

echo [Deploy] Ensuring logs directory exists...
if not exist "%LOG_DIR%" mkdir "%LOG_DIR%" 2>nul
if exist "%LOG_FILE%" del /f /q "%LOG_FILE%" 2>nul

echo [Deploy] Killing existing process on port %APP_PORT% (if any)...
for /f "tokens=5" %%p in ('netstat -aon ^| findstr /R /C:":%APP_PORT% .*LISTENING"') do (
  echo [Deploy] Killing PID %%p
  taskkill /F /PID %%p >nul 2>&1
)

echo [Deploy] Starting application in background on port %APP_PORT%...
start "" cmd /c "set PORT=%APP_PORT% && npm start >> ""%LOG_FILE%"" 2>&1"

echo [Deploy] Verifying application startup...
set STARTED=0
for /L %%i in (1,1,30) do (
  powershell -NoProfile -Command "try { (Invoke-WebRequest -UseBasicParsing ""http://127.0.0.1:%APP_PORT%/ready"").StatusCode -eq 200 -or (Invoke-WebRequest -UseBasicParsing ""http://127.0.0.1:%APP_PORT%/"").StatusCode -eq 200 } catch { $false }" | findstr /I "True" >nul 2>&1
  if not errorlevel 1 (
    set STARTED=1
    goto :started
  )
  timeout /t 1 >nul
)
:started
if "%STARTED%"=="1" (
  echo [Deploy] Application is up on port %APP_PORT%.
  echo ----- Last 50 log lines -----
  powershell -NoProfile -Command "if (Test-Path '%LOG_FILE%') { Get-Content -Tail 50 -Path '%LOG_FILE%' }"
) else (
  echo [Deploy] Application did not become healthy in time.
  echo ----- Last 200 log lines -----
  powershell -NoProfile -Command "if (Test-Path '%LOG_FILE%') { Get-Content -Tail 200 -Path '%LOG_FILE%' }"
  exit /b 1
)

endlocal
'''
            }
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
                PIDS="$(lsof -ti :"${APP_PORT}" || true)"
                if [ -n "${PIDS:-}" ]; then
                  echo "$PIDS" | xargs kill -9 || true
                fi
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

              nohup env PORT="${APP_PORT}" python "$APP_ENTRY" > "${LOG_FILE}" 2>&1 & echo $! > "${LOG_DIR}/app.pid" || true
              sleep 2

              echo "[Deploy] Verifying application startup..."
              STARTED=0
              i=1
              while [ "$i" -le 30 ]; do
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
                i=$((i+1))
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
                PIDS="$(lsof -ti :"${APP_PORT}" || true)"
                if [ -n "${PIDS:-}" ]; then
                  echo "$PIDS" | xargs kill -9 || true
                fi
              fi

              JAR_FILE="$(ls -1 target/*.jar 2>/dev/null || true)"
              if [ -z "$JAR_FILE" ]; then
                JAR_FILE="$(ls -1 build/libs/*.jar 2>/dev/null || true)"
              fi
              if [ -z "$JAR_FILE" ]; then
                echo "[Deploy] No JAR artifact found to run."
                exit 1
              fi

              nohup env SERVER_PORT="${APP_PORT}" java -jar "$JAR_FILE" > "${LOG_FILE}" 2>&1 & echo $! > "${LOG_DIR}/app.pid" || true
              sleep 2

              echo "[Deploy] Verifying application startup..."
              STARTED=0
              i=1
              while [ "$i" -le 30 ]; do
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
                i=$((i+1))
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
                PIDS="$(lsof -ti :"${APP_PORT}" || true)"
                if [ -n "${PIDS:-}" ]; then
                  echo "$PIDS" | xargs kill -9 || true
                fi
              fi

              APP_BIN="./app"
              if [ -f main.go ]; then
                go build -o "${APP_BIN}" .
              elif ls -1 *.go >/dev/null 2>&1; then
                go build -o "${APP_BIN}" ./...
              fi

              nohup env PORT="${APP_PORT}" "${APP_BIN}" > "${LOG_FILE}" 2>&1 & echo $! > "${LOG_DIR}/app.pid" || true
              sleep 2

              echo "[Deploy] Verifying application startup..."
              STARTED=0
              i=1
              while [ "$i" -le 30 ]; do
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
                i=$((i+1))
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
          if (isUnix()) {
            sh "tail -n 200 '${env.LOG_FILE}' || true"
          } else {
            bat '''
@echo off
powershell -NoProfile -Command "if (Test-Path '%LOG_FILE%') { Get-Content -Tail 200 -Path '%LOG_FILE%' }"
'''
          }
        }
      }
    }
  }
}