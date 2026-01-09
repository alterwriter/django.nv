pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '10'))
    timeout(time: 30, unit: 'MINUTES')
  }

  environment {
    // === GitHub access (PRIVATE) ===
    githubCreds  = credentials('ananda-github-access') // expected: usernamePassword
    APP_NAME     = 'django.nv'
    GITHUB_REPO  = 'django.nv' // repo name tanpa .git

    // === Tag policy (DEFINE TAG YANG BOLEH) ===
    // contoh allow:
    // - v1.2.3
    // - release-2026.01.09
    // - hotfix-auth-001
    TAG_REGEX = '^(v\\d+\\.\\d+\\.\\d+|release-\\d{4}\\.\\d{2}\\.\\d{2}|hotfix-[a-z0-9-]+)$'

    // === SonarQube config ===
    SONAR_SERVER_NAME   = 'Sonarqube-William'
    SONAR_PROJECT_KEY   = 'github_djangonv'
    SONAR_PROJECT_NAME  = 'github_djangonv'
    SONAR_SOURCES       = '.'
  }

  stages {

    stage('Resolve Tag (TAG ONLY)') {
      steps {
        script {
          def tag = (env.TAG_NAME ?: '').trim()

          // fallback kalau TAG_NAME kosong
          if (!tag) {
            def gb = (env.GIT_BRANCH ?: env.BRANCH_NAME ?: '').trim()
            tag = gb.replaceFirst(/^refs\\/tags\\//, '')
                    .replaceFirst(/^origin\\/tags\\//, '')
                    .trim()
          }

          env.RESOLVED_TAG = tag ?: ''

          echo "BRANCH_NAME   = ${env.BRANCH_NAME}"
          echo "GIT_BRANCH    = ${env.GIT_BRANCH}"
          echo "TAG_NAME      = ${env.TAG_NAME}"
          echo "RESOLVED_TAG  = ${env.RESOLVED_TAG}"

          if (!env.RESOLVED_TAG) {
            error("TAG-only pipeline: build ini bukan dari TAG. Buat TAG di GitHub dulu (misal v1.2.3).")
          }

          if (!(env.RESOLVED_TAG ==~ /${env.TAG_REGEX}/)) {
            error("TAG '${env.RESOLVED_TAG}' ditolak. Harus match regex: ${env.TAG_REGEX}")
          }

          currentBuild.displayName = "#${env.BUILD_NUMBER} ${env.RESOLVED_TAG}"
        }
      }
    }

    stage('Checkout (TAG)') {
      steps {
        sh '''
          set -e
          rm -rf "$APP_NAME" || true

          echo "▶ Clone repo..."
          git clone https://$githubCreds_USR:$githubCreds_PSW@github.com/$githubCreds_USR/$GITHUB_REPO.git "$APP_NAME"

          cd "$APP_NAME"
          echo "▶ Fetch tags..."
          git fetch --tags --force

          echo "▶ Checkout tag: $RESOLVED_TAG"
          git checkout "tags/$RESOLVED_TAG"

          echo "▶ Current commit:"
          git rev-parse HEAD
        '''
      }
    }

    stage('Sonar - Scan (Docker, TAG version)') {
      steps {
        withSonarQubeEnv("${SONAR_SERVER_NAME}") {
          sh '''
            set -e
            cd "$APP_NAME"

            echo "[INFO] Sonar scan for TAG: $RESOLVED_TAG"
            echo "[DEBUG] SONAR_HOST_URL = $SONAR_HOST_URL"

            docker run --rm \
              -e SONAR_HOST_URL="$SONAR_HOST_URL" \
              -e SONAR_TOKEN="$SONAR_AUTH_TOKEN" \
              -v "$PWD:/usr/src" \
              sonarsource/sonar-scanner-cli \
              -Dsonar.projectKey="$SONAR_PROJECT_KEY" \
              -Dsonar.projectName="$SONAR_PROJECT_NAME" \
              -Dsonar.projectVersion="$RESOLVED_TAG" \
              -Dsonar.sources="$SONAR_SOURCES"
          '''
        }
      }
    }

    // Optional: aktifin kalau Sonar webhook + plugin sudah bener
    stage('Quality Gate (optional)') {
      when { expression { return false } } // ubah ke true kalau mau aktif
      steps {
        timeout(time: 10, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

  } // end stages

  post {
    success { echo "✅ SUCCESS: TAG ${env.RESOLVED_TAG}" }
    failure { echo "❌ FAILED: TAG ${env.RESOLVED_TAG}" }
    aborted { echo "⚠️ ABORTED: TAG ${env.RESOLVED_TAG}" }
    always  {
      sh 'echo "Done"'
      cleanWs(deleteDirs: true, disableDeferredWipeout: true)
    }
  }
}
