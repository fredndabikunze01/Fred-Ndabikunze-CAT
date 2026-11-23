pipeline {
  agent any

  options {
    disableConcurrentBuilds()
  }

  environment {
    // Keep image and target branch settings in one place for easy adjustment.
    IMAGE_REPOSITORY = 'fredndabikunze01/fredndabikunze-juice-shop'
    RELEASE_TAG = 'v6.0'
    DEPLOY_BRANCH = 'master'
    DOCKERHUB_CREDENTIALS_ID = 'dockerhub-credentials'
    KUBECONFIG_CREDENTIALS_ID = 'minikube-kubeconfig'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        script {
          env.IMAGE_TAG = "${env.RELEASE_TAG}-${env.GIT_COMMIT.take(7)}"
          echo "Building ${env.IMAGE_REPOSITORY}:${env.IMAGE_TAG}"
        }
      }
    }

    stage('Install Dependencies') {
      steps {
        sh '''
          if [ -f package-lock.json ] || [ -f npm-shrinkwrap.json ]; then
            npm ci
          else
            echo 'No root npm lockfile found; installing with npm install.'
            npm install
          fi
        '''
      }
    }

    stage('Build Application') {
      steps {
        sh 'npm run build'
      }
    }

    stage('Automated Tests') {
      steps {
        sh 'npm test'
      }
    }

    stage('Build Docker Image') {
      steps {
        sh '''
          docker build --pull \
            --tag "$IMAGE_REPOSITORY:$IMAGE_TAG" \
            --tag "$IMAGE_REPOSITORY:$RELEASE_TAG" \
            .
        '''
      }
    }

    stage('Publish Docker Image') {
      when {
        expression {
          env.BRANCH_NAME == env.DEPLOY_BRANCH ||
            env.GIT_BRANCH == "origin/${env.DEPLOY_BRANCH}" ||
            env.GIT_BRANCH == env.DEPLOY_BRANCH
        }
      }
      steps {
        withCredentials([usernamePassword(
          credentialsId: env.DOCKERHUB_CREDENTIALS_ID,
          usernameVariable: 'DOCKERHUB_USERNAME',
          passwordVariable: 'DOCKERHUB_TOKEN'
        )]) {
          sh '''#!/usr/bin/env bash
            set -euo pipefail
            set +x
            # Keep Docker Hub login data in a temporary config, not the agent's home directory.
            export DOCKER_CONFIG="$(mktemp -d)"
            trap 'rm -rf "$DOCKER_CONFIG"' EXIT
            printf '%s' "$DOCKERHUB_TOKEN" |
              docker login --username "$DOCKERHUB_USERNAME" --password-stdin
            docker push "$IMAGE_REPOSITORY:$IMAGE_TAG"
            docker push "$IMAGE_REPOSITORY:$RELEASE_TAG"
          '''
        }
      }
    }

    stage('Deploy to Minikube') {
      when {
        expression {
          env.BRANCH_NAME == env.DEPLOY_BRANCH ||
            env.GIT_BRANCH == "origin/${env.DEPLOY_BRANCH}" ||
            env.GIT_BRANCH == env.DEPLOY_BRANCH
        }
      }
      steps {
        withCredentials([file(
          credentialsId: env.KUBECONFIG_CREDENTIALS_ID,
          variable: 'KUBECONFIG'
        )]) {
          sh '''#!/usr/bin/env bash
            set -euo pipefail
            minikube status
            kubectl config current-context
            # Apply generated resources so reruns update the Deployment and Service in place.
            kubectl create deployment juice-shop \
              --image="$IMAGE_REPOSITORY:$IMAGE_TAG" \
              --port=3000 \
              --dry-run=client -o yaml | kubectl apply -f -
            kubectl expose deployment juice-shop \
              --type=NodePort \
              --port=3000 \
              --target-port=3000 \
              --dry-run=client -o yaml | kubectl apply -f -
            kubectl rollout status deployment/juice-shop --timeout=180s
          '''
        }
      }
    }

    stage('Verify Deployment') {
      when {
        expression {
          env.BRANCH_NAME == env.DEPLOY_BRANCH ||
            env.GIT_BRANCH == "origin/${env.DEPLOY_BRANCH}" ||
            env.GIT_BRANCH == env.DEPLOY_BRANCH
        }
      }
      steps {
        withCredentials([file(
          credentialsId: env.KUBECONFIG_CREDENTIALS_ID,
          variable: 'KUBECONFIG'
        )]) {
          sh '''#!/usr/bin/env bash
            set -euo pipefail
            kubectl get deployment juice-shop
            kubectl get pods --selector=app=juice-shop
            kubectl get service juice-shop --output=wide
            kubectl get endpoints juice-shop
            MINIKUBE_IP="$(minikube ip)"
            NODE_PORT="$(kubectl get service juice-shop -o jsonpath='{.spec.ports[0].nodePort}')"
            echo "Application URL: http://${MINIKUBE_IP}:${NODE_PORT}"
          '''
        }
      }
    }
  }

  post {
    success {
      echo "Pipeline succeeded. Image: ${env.IMAGE_REPOSITORY}:${env.IMAGE_TAG}"
    }
    failure {
      echo 'Pipeline failed. Review the stage output above for the error.'
    }
    always {
      echo "Build result: ${currentBuild.currentResult}"
    }
  }
}
