// pipeline {
  
//   agent any

//    environment {
//     // Add Homebrew bin/sbin *before* the inherited PATH
//     PATH = "/opt/homebrew/bin:/opt/homebrew/sbin:${PATH}"
//   }
  
//   stages {
    
//     stage('Verify gcloud in Jenkins') {
//       steps {
//         sh '''
//           echo "Shell: $SHELL"
//           echo "PATH: $PATH"
//           which gcloud || { echo "gcloud not found"; exit 1; }
//           gcloud --version
//         '''
//       }
//     }

//     stage('gcloud auth test') {
//       steps {
//         withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
//           sh '''
//             gcloud auth activate-service-account --key-file="$GOOGLE_APPLICATION_CREDENTIALS"
//             gcloud auth list
//           '''
//         }
//       }
//     }
//   }
// }


pipeline {
  agent any

  environment {
    //set these to your project/region/repo
    PROJECT_ID    = 'bright-feat-471605-t7'
    REGION        = 'us-central1'
    REPO          = 'app-images'
    IMAGE_NAME    = 'demo-api'
    IMAGE_URI     = "us-central1-docker.pkg.dev/${PROJECT_ID}/${REPO}/${IMAGE_NAME}"
    PATH = "/opt/homebrew/bin:/opt/homebrew/sbin:${PATH}"
  }

  stages {
    
    stage('Checkout') {
      steps { checkout scm }
    }

    // stage('Check gcloud') {
    //   steps {
    //     sh '''
    //       echo "PATH = $PATH"
    //       which gcloud || echo "gcloud not found"
    //       gcloud --version || true
    //     '''
    //   }
    // }

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
          '''
        }
      }
    }

    stage('Build & Push Image') {
      steps {
        sh '''
          docker build -t ${IMAGE_URI}:${APP_TAG} -t ${IMAGE_URI}:latest .
          docker push ${IMAGE_URI}:${APP_TAG}
          docker push ${IMAGE_URI}:latest
        '''
      }
    }

    stage('Deploy to GKE') {
      steps {
        sh '''
          # get cluster credentials for kubectl
          gcloud container clusters get-credentials autopilot-cluster-1 --region ${REGION}

          # update k8s manifests to use the new tag
          sed -i "s#${REGION}-docker.pkg.dev/.*/${IMAGE_NAME}:.*#${IMAGE_URI}:${APP_TAG}#g" k8s/deployment.yaml

          # create namespace if missing, then apply
          kubectl create namespace demo --dry-run=client -o yaml | kubectl apply -f -
          kubectl apply -f k8s/deployment.yaml
          kubectl apply -f k8s/service.yaml

          # (optional) roll out status
          kubectl rollout status deploy/demo-api -n demo --timeout=120s
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
