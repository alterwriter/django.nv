pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
    disableConcurrentBuilds()
  }

  environment {
    // === GitHub access (PRIVATE) ===
    githubCreds  = credentials('ananda-github-access') // expected: usernamePassword
    APP_NAME     = 'django.nv'
    GITHUB_REPO  = 'django.nv'           // ganti sesuai repo, tanpa .git

    // === Tag policy (DEFINE TAG YANG BOLEH) ===
    // Contoh yang di-allow:
    // - v1.2.3
    // - release-2026.01.09
    // - hotfix-auth-001
    // Silakan ubah sesuai standar tim kamu.
    TAG_REGEX = '^(v\\d+\\.\\d+\\.\\d+|release-\\d{4}\\.\\d{2}\\.\\d{2}|hotfix-[a-z0-9-]+)$'

    // === SonarQube config ===
    SONAR_SERVER_NAME   = 'Sonarqube-William'
    SONAR_PROJECT_KEY   = 'github_djangonv'
    SONAR_PROJECT_NAME  = 'github_djangonv'
    SONAR_SOURCES       = '.'
  }

  stages {

    stage('Resolve Tag & Guard (TAG ONLY)') {
      steps {
        script {
          // Jenkins (Multibranch) biasanya ngasih TAG_NAME kalau build dari tag
          def tag = env.TAG_NAME

          // Fallback: kadang GIT_BRANCH isinya refs/tags/<tag> atau origin/tags/<tag>
          if (!tag?.trim()) {
            def gb = env.GIT_BRANCH ?: env.BRANCH_NAME ?: ''
            // contoh gb: "refs/tags/v1.2.3" atau "origin/tags/v1.2.3"
            tag = gb.replaceFirst(/^refs\\/tags\\//, '')
                    .replaceFirst(/^origin\\/tags\\//, '')
          }

          tag = tag?.trim()
          env.RESOLVED_TAG = tag ?: ''

          echo "BRANCH_NAME = ${env.BRANCH_NAME}"
          echo "GIT_BRANCH  = ${env.GIT_BRANCH}"
          echo "TAG_NAME    = ${env.TAG_NAME}"
          echo "RESOLVED_TAG= ${env.RESOLVED_TAG}"

          if (!env.RESOLVED_TAG) {
            error("TAG-only pipeline: build ini bukan dari TAG. Bikin TAG di GitHub dulu (misal v1.2.3), baru jalan.")
          }

          // Enforce tag format
          if (!(env.RESOLVED_TAG ==~ /${env.TAG_REGEX}/)) {
            error("TAG '${env.RESOLVED_TAG}' ditolak. Harus match regex: ${env.TAG_REGEX}")
          }

          currentBuild.displayName = "#${env.BUILD_NUMBER} ${env.RESOLVED_TAG}"
          currentBuild.description = "Tag-based run: ${env.RESOLVED_TAG}"
        }
      }
    }

    stage('Checkout (TAG)') {
      steps {
        sh '''
          set -euo pipefail
          rm -rf "$APP_NAME" || true

          echo "▶ Cloning repo..."
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
            set -euo pipefail
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

    // === OPTIONAL: kalau mau Quality Gate nahan pipeline ===
    // Pastikan plugin SonarQube + webhook Sonar ke Jenkins sudah bener
    stage('Quality Gate (optional)') {
      when { expression { return true } } // ubah ke false kalau belum siap
      steps {
        timeout(time: 10, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    // === OPTIONAL: Build / Test / Deploy berdasarkan TAG ===
    stage('Deploy (placeholder)') {
      steps {
        sh '''
          set -euo pipefail
          echo "🚀 Deploying TAG: $RESOLVED_TAG"
          echo "TODO: taro step deploy kamu di sini"
          # contoh:
          # docker build -t myapp:$RESOLVED_TAG .
          # docker push registry/myapp:$RESOLVED_TAG
          # ssh server "deploy myapp:$RESOLVED_TAG"
        '''
      }
    }

  } // end stages

  post {
    success {
      echo "✅ SUCCESS: TAG ${env.RESOLVED_TAG} scanned & processed"
    }
    failure {
      echo "❌ FAILED: TAG ${env.RESOLVED_TAG} - cek log stage yang error"
    }
    aborted {
      echo "⚠️ ABORTED: TAG ${env.RESOLVED_TAG}"
    }
    always {
      sh 'echo "Done"'
      cleanWs(deleteDirs: true, disableDeferredWipeout: true)
    }
  }
}
