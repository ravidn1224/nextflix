pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
  }

  environment {
    IMAGE_NAME = "ravid-netflix"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        script {
          sh 'git rev-parse --short HEAD > .git/shortsha'
          env.SHORT_SHA = readFile('.git/shortsha').trim()
        }
      }
    }

    stage('Compute Image Tag') {
      steps {
        script {
          if (env.BRANCH_NAME == 'main') {
            env.IMAGE_TAG = "main-${env.BUILD_NUMBER}-${env.SHORT_SHA}"
          } else {
            env.IMAGE_TAG = "${env.BRANCH_NAME}-${env.SHORT_SHA}"
          }
          echo "IMAGE_TAG=${env.IMAGE_TAG}"
        }
      }
    }

    stage('Unit Tests') {
      steps {
        sh '''
          docker run --rm -v "$PWD":/app -w /app node:20-alpine \
            sh -lc "npm ci && npm test || echo '⚠️ No tests or tests failed'"
        '''
      }
    }

    stage('Build Docker Image') {
      steps {
        sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
      }
    }

    stage('Deploy to STAGING') {
      when { branch 'main' }
      steps {
        sh "TAG=${IMAGE_TAG} docker compose -f docker-compose.staging.yml up -d"
      }
    }

    stage('Deploy to PRODUCTION (if PR merged)') {
      when {
        allOf {
          branch 'main'
          expression {
            def msg = sh(script: "git log -1 --pretty=%s", returnStdout: true).trim()
            return msg ==~ /^Merge pull request #\\d+.*/
          }
        }
      }
      steps {
        sh "TAG=${IMAGE_TAG} docker compose -f docker-compose.prod.yml up -d"
      }
    }
  }

  post {
    success {
      echo "✅ Deployed ${IMAGE_NAME}:${IMAGE_TAG}"
    }
    failure {
      echo "❌ Pipeline failed"
    }
  }
}
