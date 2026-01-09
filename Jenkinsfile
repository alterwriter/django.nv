pipeline {
  agent any

  environment {
    // === GitHub access (PRIVATE) ===
    githubCreds  = credentials('ananda-github-access')
    BRANCH_REPO  = 'main'
    APP_NAME     = 'django.nv'
    GITHUB_REPO  = 'django.nv'           // ganti sesuai repo

    // === SonarQube config (optional tetap) ===
    SONAR_PROJECT_KEY   = 'github_djangonv'
    SONAR_PROJECT_NAME  = 'github_djangonv'
    SONAR_SOURCES       = '.'
  }

  stages {
    stage('Git Pull (GitHub)') {
      steps {
        sh '''
          set -e
          rm -rf "$APP_NAME" || true

          # NOTE:
          # githubCreds_PSW = PAT token
          # githubCreds_USR = username
          git clone https://$githubCreds_USR:$githubCreds_PSW@github.com/$githubCreds_USR/$GITHUB_REPO.git "$APP_NAME"

          cd "$APP_NAME"
          git fetch --tags
          git switch "$BRANCH_REPO"
        '''
      }
    }

    stage('Sonar - Scan (Docker)') {
      steps {
        withSonarQubeEnv('Sonarqube-William') {
          sh '''
            set -e
            cd "$APP_NAME"

            echo "[INFO] Running sonar-scanner via Docker..."
            echo "[DEBUG] SONAR_HOST_URL = $SONAR_HOST_URL"

            docker run --rm \
              -e SONAR_HOST_URL="$SONAR_HOST_URL" \
              -e SONAR_TOKEN="$SONAR_AUTH_TOKEN" \
              -v "$PWD:/usr/src" \
              sonarsource/sonar-scanner-cli \
              -Dsonar.projectKey="$SONAR_PROJECT_KEY" \
              -Dsonar.projectName="$SONAR_PROJECT_NAME" \
              -Dsonar.projectVersion="$BRANCH_REPO-$BUILD_NUMBER" \
              -Dsonar.sources="$SONAR_SOURCES" \
              -Dsonar.branch.name="$BRANCH_REPO"
          '''
        }
      }
    }

  post {
    success {
      echo "✅ Pipeline Success (Sonar OK)"
    }
    failure {
      echo "❌ Something is wrong, pipeline failed..."
    }
    aborted {
      echo "⚠️ Pipeline Aborted..."
    }
    always {
      sh 'echo "Done"'
    }
  }
}
