// Lab 5: app-server-podman-deploy
// Deploys an image built by Lab 4 (docker-build-push-ocir) to the app server as a rootless
// Podman container managed by a Quadlet systemd user service.
//
//   deploy   = deploy IMAGE_DIGEST (or, if empty, redeploy the live image)
//   rollback = redeploy the newest healthy release older than the live one
//
// The registry token is passed to the server only for the pull and deleted straight after.
// The release to verify is read from the image's own label, not from a parameter.

pipeline {
  agent any

  options {
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '30'))
    timeout(time: 15, unit: 'MINUTES')
  }

  triggers {
    githubPush()
  }

  parameters {
    string(name: 'IMAGE_DIGEST', defaultValue: '', trim: true,
           description: 'Image to deploy, e.g. <registry>/<namespace>/ci-labs/flask-api@sha256:... (from Lab 4 image.json). Empty = redeploy the live image.')
    choice(name: 'ACTION', choices: ['deploy', 'rollback'],
           description: 'deploy = deploy IMAGE_DIGEST; rollback = return to the previous healthy release')
  }

  environment {
    HEALTH_URL = 'http://127.0.0.1:8086/health'   // through nginx, as users reach it
    SSH_OPTS   = '-o BatchMode=yes -o ConnectTimeout=10'
  }

  stages {
    stage('Preflight') {
      steps {
        script {
          ['APP_HOST', 'OCIR_HOST', 'OCIR_NAMESPACE'].each { v ->
            if (!env."${v}"?.trim()) {
              error("${v} is not set: Manage Jenkins > System > Global properties > Environment variables")
            }
          }
          env.REPO = "${env.OCIR_HOST}/${env.OCIR_NAMESPACE}/ci-labs/flask-api"
        }
        sshagent(credentials: ['app-server-deploy']) {
          sh '''ssh $SSH_OPTS deploy@$APP_HOST 'echo "connected to $(hostname -s) as $(whoami), $(podman --version)"' '''
        }
      }
    }

    stage('Resolve image') {
      steps {
        sshagent(credentials: ['app-server-deploy']) {
          script {
            def action = params.ACTION ?: 'deploy'
            def target = ''
            if (action == 'rollback') {
              def line = sh(script: 'ssh $SSH_OPTS deploy@$APP_HOST bash -s -- rollback-target < scripts/release.sh',
                            returnStdout: true).trim()
              if (!line) { error('No earlier healthy release to roll back to') }
              target = line.tokenize(' ')[1]
              echo "rollback target: ${line}"
            } else if (params.IMAGE_DIGEST?.trim()) {
              target = params.IMAGE_DIGEST.trim()
            } else {
              def line = sh(script: 'ssh $SSH_OPTS deploy@$APP_HOST bash -s -- current < scripts/release.sh',
                            returnStdout: true).trim()
              if (!line) { error('IMAGE_DIGEST is empty and nothing is deployed yet: paste the digest from Lab 4 image.json') }
              target = line.tokenize(' ')[1]
              echo "no IMAGE_DIGEST given: redeploying the live image"
            }
            // Only immutable references to OUR repository are accepted
            def prefix = env.REPO + '@sha256:'
            if (!(target.startsWith(prefix) && target.substring(prefix.length()) ==~ /[0-9a-f]{64}/)) {
              error("Refusing image '${target}': must be ${env.REPO}@sha256:<64 hex>")
            }
            env.TARGET_IMAGE = target
            currentBuild.displayName = "#${env.BUILD_NUMBER} ${action} ${target.substring(target.length() - 12)}"
          }
        }
      }
    }

    stage('Pull on server') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'ocir-push',
                                          usernameVariable: 'OCIR_USER', passwordVariable: 'OCIR_TOKEN')]) {
          sshagent(credentials: ['app-server-deploy']) {
            sh '''
              # The token travels on stdin into a temporary auth file that is deleted after the pull
              printf '%s' "$OCIR_TOKEN" | ssh $SSH_OPTS deploy@$APP_HOST \
                "umask 077; A=\\$(mktemp); trap 'rm -f \\$A' EXIT; podman login --authfile \\$A -u '$OCIR_USER' --password-stdin $OCIR_HOST >/dev/null && podman pull --quiet --authfile \\$A '$TARGET_IMAGE' >/dev/null && echo 'pulled: $TARGET_IMAGE (temporary login removed)'"
            '''
          }
        }
      }
    }

    stage('Deploy') {
      steps {
        sshagent(credentials: ['app-server-deploy']) {
          sh 'ssh $SSH_OPTS deploy@$APP_HOST bash -s -- deploy "$TARGET_IMAGE" "$HEALTH_URL" < scripts/release.sh'
        }
      }
    }

    stage('Smoke test') {
      steps {
        sshagent(credentials: ['app-server-deploy']) {
          sh '''
            ssh $SSH_OPTS deploy@$APP_HOST "curl -fsS 'http://127.0.0.1:8086/api/items?category=network'" | tee smoke.json
            echo
            grep -q '"count":' smoke.json || { echo "smoke test failed"; exit 1; }
            ssh $SSH_OPTS deploy@$APP_HOST 'podman ps --filter name=inventory-api --format "container: {{.Names}} | {{.Status}} | {{.Image}}"'
            echo "smoke test passed"
          '''
        }
      }
    }
  }

  post {
    success {
      script {
        def t = env.TARGET_IMAGE ?: ''
        currentBuild.description = "${params.ACTION ?: 'deploy'} ok: ...${t.length() > 12 ? t.substring(t.length() - 12) : t}"
      }
    }
    always {
      sh 'rm -f smoke.json'
    }
  }
}
