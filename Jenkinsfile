pipeline {
  agent any
  environment {
    DOCKER_ID   = "abeonim"
    MOVIE_IMAGE = "movies-api"
    CAST_IMAGE  = "casts-api"
    BUILD_TAG   = "v${BUILD_NUMBER}.0"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Docker Build') {
      steps {
        sh '''
          set -e
          docker build -t $DOCKER_ID/$MOVIE_IMAGE:$BUILD_TAG ./movie-service
          docker build -t $DOCKER_ID/$CAST_IMAGE:$BUILD_TAG  ./cast-service
        '''
      }
    }

    stage('Smoke test (containers)') {
      steps {
        script {
          catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
            sh '''
              set -e
              docker rm -f test-movie test-cast || true
              docker run -d --name test-movie -p 8081:8000 $DOCKER_ID/$MOVIE_IMAGE:$BUILD_TAG
              docker run -d --name test-cast  -p 8082:8000 $DOCKER_ID/$CAST_IMAGE:$BUILD_TAG
              sleep 6
              # FastAPI expose TOUJOURS /openapi.json -> 200
              curl -sf http://localhost:8081/openapi.json >/dev/null
              curl -sf http://localhost:8082/openapi.json >/dev/null
            '''
          }
        }
        sh 'docker rm -f test-movie test-cast || true'
      }
    }

    stage('Docker Push') {
      environment { DOCKER_PASS = credentials('DOCKER_HUB_PASS') }
      steps {
        sh '''
          set -e
          docker login -u $DOCKER_ID -p $DOCKER_PASS
          docker push $DOCKER_ID/$MOVIE_IMAGE:$BUILD_TAG
          docker push $DOCKER_ID/$CAST_IMAGE:$BUILD_TAG
        '''
      }
    }

    stage('Deploy dev') {
      environment { KUBECONFIG = credentials('config') }
      steps {
        sh '''
          set -e
          mkdir -p .kube && cat $KUBECONFIG > .kube/config

          # DBs (tes manifests existants)
          kubectl -n dev apply -f k8s/postgres-movie.yaml
          kubectl -n dev apply -f k8s/postgres-cast.yaml
          kubectl -n dev rollout status deploy/movie-db -n dev --timeout=180s
          kubectl -n dev rollout status deploy/cast-db  -n dev --timeout=180s

          # Apps (on fixe le tag du build du jour via --set)
          helm upgrade --install movie-api charts -n dev -f charts/movie-values.yaml --set image.tag=$BUILD_TAG --wait --atomic
          helm upgrade --install cast-api  charts -n dev -f charts/cast-values.yaml  --set image.tag=$BUILD_TAG --wait --atomic

          echo -n "movie image (dev): "; kubectl -n dev get deploy movie-api-fastapiapp -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
          echo -n "cast  image (dev): "; kubectl -n dev get deploy cast-api-fastapiapp  -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
        '''
      }
    }

    stage('Deploy staging') {
      environment { KUBECONFIG = credentials('config') }
      steps {
        sh '''
          set -e
          mkdir -p .kube && cat $KUBECONFIG > .kube/config
          helm upgrade --install movie-api charts -n staging -f charts/movie-values.yaml --set image.tag=$BUILD_TAG --wait --atomic
          helm upgrade --install cast-api  charts -n staging -f charts/cast-values.yaml  --set image.tag=$BUILD_TAG --wait --atomic
        '''
      }
    }

    stage('Approval & Deploy prod') {
      when { branch 'main' }
      environment { KUBECONFIG = credentials('config') }
      steps {
        timeout(time: 15, unit: 'MINUTES') {
          input message: 'Déployer en production ?', ok: 'Oui'
        }
        sh '''
          set -e
          mkdir -p .kube && cat $KUBECONFIG > .kube/config
          helm upgrade --install movie-api charts -n prod -f charts/movie-values.yaml --set image.tag=$BUILD_TAG --wait --atomic
          helm upgrade --install cast-api  charts -n prod -f charts/cast-values.yaml  --set image.tag=$BUILD_TAG --wait --atomic

          echo -n "movie image (prod): "; kubectl -n prod get deploy movie-api-fastapiapp -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
          echo -n "cast  image (prod): "; kubectl -n prod get deploy cast-api-fastapiapp  -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
        '''
      }
    }
  }

  post {
    success { echo "OK: ${BUILD_TAG} déployé" }
    failure { echo "FAILED build ${BUILD_NUMBER}" }
  }
}
