import groovy.transform.Field

// =====================================================
// DEMO Jenkinsfile - Discord Webhook (HARDCODE)
// =====================================================

// ⚠️ DEMO ONLY — webhook boleh dimatiin setelah training
@Field String DISCORD_WEBHOOK = 'https://discord.com/api/webhooks/1459736436520521788/2SchBPBcOMIbZ9MQC8a7iKLaTxR4w7Q45KaZd0Aq1TWS61E-i68H84kmLvRpC9HHFRKM'

// =========================
// Discord Notifier (OPS)
// =========================
def notifyDiscord(String status) {
  // Guard biar gak nge-crash post kalau webhook kosong / typo
  if (!DISCORD_WEBHOOK?.trim()) {
    echo "[WARN] DISCORD_WEBHOOK empty, skip notification."
    return
  }

  def tag   = (env.RESOLVED_TAG ?: env.TAG_NAME ?: '-')
  def envNm = (env.DEPLOY_ENV ?: 'Staging')   // kalau gak ada, fallback Staging
  def stage = (env.LAST_STAGE ?: '-')
  def url   = (env.BUILD_URL ?: '-')
  def job   = (env.JOB_NAME ?: '-')
  def build = (env.BUILD_NUMBER ?: '-')

  def color =
    (status == 'SUCCESS') ? 3066993 :
    (status == 'FAILED')  ? 15158332 :
    (status == 'ABORTED') ? 16705372 :
                            9807270

  // Mention only when FAILED
  def mention = (status == 'FAILED') ? '@here ' : ''

  // Escape minimal biar gak ancur karena karakter aneh
  def safeJob   = job.replace('"', '\\"')
  def safeUrl   = url.replace('"', '\\"')
  def safeTag   = tag.replace('"', '\\"')
  def safeStage = stage.replace('"', '\\"')
  def safeEnv   = envNm.replace('"', '\\"')

  sh """
    set -e
    payload=\$(cat <<'JSON'
{
  "content": "${mention}",
  "embeds": [
    {
      "title": "${status} · ${safeEnv}",
      "color": ${color},
      "fields": [
        { "name": "Service", "value": "${safeJob}", "inline": true },
        { "name": "Build", "value": "#${build}", "inline": true },
        { "name": "Version", "value": "${safeTag}", "inline": true },
        { "name": "Stage", "value": "${safeStage}", "inline": true },
        { "name": "Jenkins URL", "value": "${safeUrl}", "inline": false }
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
    // Label env buat notif
    DEPLOY_ENV = 'Staging'

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
    // Ini paling aman: selalu kirim notif sekali aja berdasarkan result final
    always {
      script {
        def r = (currentBuild.currentResult ?: 'UNKNOWN')
        def status =
          (r == 'SUCCESS') ? 'SUCCESS' :
          (r == 'FAILURE') ? 'FAILED'  :
          (r == 'ABORTED') ? 'ABORTED' : r

        echo "📣 Notify Discord: ${status} ${env.RESOLVED_TAG ?: ''}"
        notifyDiscord(status)
      }

      cleanWs(deleteDirs: true, disableDeferredWipeout: true)
    }
  }
}
