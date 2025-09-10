// 

pipeline {
  agent any

  environment {
    // Set these to your project/region/repo
    PROJECT_ID = 'bright-feat-471605-t7'
    REGION     = 'us-central1'
    REPO       = 'app-images'
    IMAGE_NAME = 'demo-api'
    IMAGE_URI  = "us-central1-docker.pkg.dev/${PROJECT_ID}/${REPO}/${IMAGE_NAME}"

    // Optional: set your target GKE cluster/namespace
    CLUSTER    = 'dev-cluster'
    NAMESPACE  = 'demo'
  }

  stages {

    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Set Version') {
      steps {
        script {
          def GIT_SHA = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
          env.APP_TAG = "${GIT_SHA}"
          echo "Version tag: ${env.APP_TAG}"
        }
      }
    }

    stage('Unit Tests') {
      steps {
        sh '''
          set -euxo pipefail
          python3 -m venv .venv
          . .venv/bin/activate
          pip install -r requirements.txt
          python -c "import flask; print('ok')"
        '''
      }
    }

    stage('Auth to GCP') {
      steps {
        withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
          sh '''
            set -euxo pipefail
            # Activate SA & set project
            gcloud auth activate-service-account --key-file="$GOOGLE_APPLICATION_CREDENTIALS"
            gcloud config set project ${PROJECT_ID}

            # Ensure Docker uses GCP creds for Artifact Registry host
            gcloud auth configure-docker ${REGION}-docker.pkg.dev --quiet

            # (Optional) quick sanity checks
            gcloud auth list
            gcloud auth print-access-token | head -c 20; echo "... token ok"
          '''
        }
      }
    }

    stage('Build & Push Image') {
      steps {
        sh '''
          set -euxo pipefail

          # Build with both commit tag and "latest"
          docker build -t ${IMAGE_URI}:${APP_TAG} -t ${IMAGE_URI}:latest .

          # Push to Artifact Registry (hostname/region/project MUST match)
          docker push ${IMAGE_URI}:${APP_TAG}
          docker push ${IMAGE_URI}:latest
        '''
      }
    }

    stage('Deploy to GKE') {
      steps {
        sh '''
          set -euxo pipefail

          # Get cluster credentials for kubectl
          gcloud container clusters get-credentials ${CLUSTER} --region ${REGION} --project ${PROJECT_ID}

          # Update the image tag inside the manifest
          # Assumes your deployment.yaml has an image starting with "us-central1-docker.pkg.dev/"
          sed -i "s#${REGION}-docker.pkg.dev/.*/${IMAGE_NAME}:.*#${IMAGE_URI}:${APP_TAG}#g" k8s/deployment.yaml

          # Ensure namespace exists
          kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

          # Apply manifests to that namespace (so you don't rely on 'namespace:' fields)
          kubectl -n ${NAMESPACE} apply -f k8s/deployment.yaml
          kubectl -n ${NAMESPACE} apply -f k8s/service.yaml

          # Wait for rollout
          kubectl -n ${NAMESPACE} rollout status deploy/${IMAGE_NAME} --timeout=120s
        '''
      }
    }
  }

  post {
    success {
      echo "Deployed ${IMAGE_URI}:${APP_TAG} to GKE namespace '${NAMESPACE}'."
    }
    failure {
      echo "Pipeline failed."
    }
  }
}
