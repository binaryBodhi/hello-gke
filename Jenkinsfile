pipeline {
  agent any

  parameters {
    choice(name: 'ENV', choices: ['dev', 'sit', 'uat'], description: 'Target environment/namespace')
  }

  environment {
    // # set these to your project/region/repo
    PROJECT_ID    = 'test-project-472205'
    REGION        = 'us-central1'
    REPO          = 'app-images'
    IMAGE_NAME    = 'demo-api'
    IMAGE_URI     = "us-central1-docker.pkg.dev/${PROJECT_ID}/${REPO}/${IMAGE_NAME}"
    // PATH = "/opt/homebrew/bin:/opt/homebrew/sbin:/opt/homebrew/Caskroom/google-cloud-sdk/latest/google-cloud-sdk/bin:${PATH}"
    K8S_NAMESPACE = "${params.ENV}"
  }


  stages {
    
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Set Version') {
      steps {
        script {
          GIT_SHA = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
          env.APP_TAG = "${GIT_SHA}"
          echo "Version tag: ${APP_TAG}"
        }
      }
    }

    stage('Unit Tests') {
      steps {
        sh '''
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
            gcloud auth activate-service-account --key-file="$GOOGLE_APPLICATION_CREDENTIALS"
            gcloud config set project ${PROJECT_ID}
            gcloud auth configure-docker ${REGION}-docker.pkg.dev --quiet

            # Direct Docker login using service account JSON key
            # cat "$GOOGLE_APPLICATION_CREDENTIALS" | docker login -u _json_key --password-stdin https://us-central1-docker.pkg.dev

            
          '''
        }
      }
    }

    stage('Build & Push Image') {
      steps {
        sh '''
          # Enable buildx (safe to run multiple times)
          docker buildx create --use || true

          # Build for amd64 (Intel/AMD) only and push directly
          docker buildx build \
            --platform linux/amd64 \
            -t ${IMAGE_URI}:${APP_TAG} \
            -t ${IMAGE_URI}:${K8S_NAMESPACE} \
            --push .
        '''
      }
    } 

    // stage('Build & Push Image') {
    //   steps {
    //     sh '''
    //       # echo 'Logging in to Artifact Registry...'
    //       # gcloud auth print-access-token | docker login -u oauth2accesstoken --password-stdin https://${REGION}-docker.pkg.dev

    //       docker build -t ${IMAGE_URI}:${APP_TAG} -t ${IMAGE_URI}:${APP_TAG} .
    //       docker push ${IMAGE_URI}:${APP_TAG}
    //       docker push ${IMAGE_URI}:latest
    //     '''
    //   }
    // }

    stage('Deploy to GKE') {
      steps {
        sh '''
          # get cluster credentials for kubectl
          gcloud container clusters get-credentials autopilot-cluster-1 --region ${REGION}

          # update k8s manifests to use the new tag
          sed -i'' -e "s#${REGION}-docker.pkg.dev/.*/${IMAGE_NAME}:.*#${IMAGE_URI}:${K8S_NAMESPACE}#g" k8s/deployment.yaml
          # sed -i'' -e "s#${REGION}-docker.pkg.dev/.*/${IMAGE_NAME}:.*#${IMAGE_URI}:${APP_TAG}#g" k8s/deployment.yaml

          # create namespace if missing, then apply
          kubectl create namespace ${K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
          kubectl apply -n ${K8S_NAMESPACE} -f k8s/deployment.yaml
          kubectl apply  -n ${K8S_NAMESPACE} -f k8s/service.yaml

          # (optional) roll out status
          # kubectl rollout status deploy/demo-api -n demo --timeout=120s
        '''
      }
    }
  }

  post {
    success {
      echo "Deployed ${IMAGE_URI}:${APP_TAG} to GKE namespace 'demo'."
    }
    failure {
      echo "Pipeline failed."
    }
  }
}
