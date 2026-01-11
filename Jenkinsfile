// =====================================================
// DEMO Jenkinsfile - Discord Webhook (HARDCODE)
// =====================================================

// ⚠️ DEMO ONLY — webhook boleh dimatiin setelah training
def DISCORD_WEBHOOK = 'https://discord.com/api/webhooks/1459736436520521788/2SchBPBcOMIbZ9MQC8a7iKLaTxR4w7Q45KaZd0Aq1TWS61E-i68H84kmLvRpC9HHFRKM'

// =========================
// Discord Notifier (OPS)
// =========================
def notifyDiscord(String status) {
  def tag   = (env.RESOLVED_TAG ?: env.TAG_NAME ?: '-')
  def envNm = 'Staging'
  def stage = (env.LAST_STAGE ?: '-')
  def url   = (env.BUILD_URL ?: '-')
  def job   = (env.JOB_NAME ?: '-')
  def build = (env.BUILD_NUMBER ?: '-')

  def color =
    (status == 'SUCCESS') ? 3066993 :
    (status == 'FAILED')  ? 15158332 :
    (status == 'ABORTED') ? 16705372 :
                            9807270

  def mention = (status == 'FAILED') ? '@here ' : ''

  sh """
    set -e
    payload=\$(cat <<'JSON'
{
  "content": "${mention}",
  "embeds": [
    {
      "title": "${status} · ${envNm}",
      "color": ${color},
      "fields": [
        { "name": "Service", "value": "${job}", "inline": true },
        { "name": "Build", "value": "#${build}", "inline": true },
        { "name": "Version", "value": "${tag}", "inline": true },
        { "name": "Stage", "value": "${stage}", "inline": true },
        { "name": "Jenkins URL", "value": "${url}", "inline": false }
      ],
      "footer": { "text": "CI/CD Deployment Notification" }
    }
  ]
}
JSON
)
    curl -sS -X POST -H "Content-Type: application/json" --data "\$payload" "${DISCORD_WEBHOOK}" >/dev/null
  """
}

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
    githubCreds  = credentials('ananda-github-access')
    APP_NAME     = 'django.nv'
    GITHUB_REPO  = 'django.nv'

    // === Tag policy ===
    TAG_REGEX = '^(v\\d+\\.\\d+\\.\\d+|release-\\d{4}\\.\\d{2}\\.\\d{2}|hotfix-[a-z0-9-]+)$'

    // === SonarQube ===
    SONAR_SERVER_NAME   = 'Sonarqube-William'
    SONAR_PROJECT_KEY   = 'github_djangonv'
    SONAR_PROJECT_NAME  = 'github_djangonv'
    SONAR_SOURCES       = '.'
  }

  stages {

    stage('Resolve Tag (TAG ONLY)') {
      steps {
        script { env.LAST_STAGE = env.STAGE_NAME }

        script {
          def tag = (env.TAG_NAME ?: '').trim()

          if (!tag) {
            def gb = (env.GIT_BRANCH ?: env.BRANCH_NAME ?: '').trim()
            tag = gb.replaceFirst(/^refs\\/tags\\//, '')
                    .replaceFirst(/^origin\\/tags\\//, '')
                    .trim()
          }

          env.RESOLVED_TAG = tag ?: ''

          echo "TAG = ${env.RESOLVED_TAG}"

          if (!env.RESOLVED_TAG) {
            error("TAG-only pipeline. Buat TAG dulu (misal v1.2.3).")
          }

          if (!(env.RESOLVED_TAG ==~ /${env.TAG_REGEX}/)) {
            error("TAG '${env.RESOLVED_TAG}' tidak sesuai policy.")
          }

          currentBuild.displayName = "#${env.BUILD_NUMBER} ${env.RESOLVED_TAG}"
        }
      }
    }

    stage('Checkout (TAG)') {
      steps {
        script { env.LAST_STAGE = env.STAGE_NAME }

        sh '''
          set -e
          rm -rf "$APP_NAME" || true

          git clone https://$githubCreds_USR:$githubCreds_PSW@github.com/$githubCreds_USR/$GITHUB_REPO.git "$APP_NAME"
          cd "$APP_NAME"

          git fetch --tags --force
          git checkout "tags/$RESOLVED_TAG"
          git rev-parse HEAD
        '''
      }
    }

    stage('Sonar Scan (Docker)') {
      steps {
        script { env.LAST_STAGE = env.STAGE_NAME }

        withSonarQubeEnv("${SONAR_SERVER_NAME}") {
          sh '''
            set -e
            cd "$APP_NAME"

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
  }

  post {
    success {
      echo "✅ SUCCESS ${env.RESOLVED_TAG}"
      script { notifyDiscord('SUCCESS') }
    }
    failure {
      echo "❌ FAILED ${env.RESOLVED_TAG}"
      script { notifyDiscord('FAILED') }
    }
    aborted {
      echo "⚠️ ABORTED ${env.RESOLVED_TAG}"
      script { notifyDiscord('ABORTED') }
    }
    always {
      cleanWs(deleteDirs: true, disableDeferredWipeout: true)
    }
  }
}
