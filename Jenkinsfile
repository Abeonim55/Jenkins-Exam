pipeline {
  agent any
  environment {
    DOCKER_ID    = "abeonim"            // Docker Hub user
    MOVIE_IMAGE  = "movies-api"
    CAST_IMAGE   = "casts-api"
    BUILD_TAG    = "v${BUILD_NUMBER}.0" // tag unique à chaque build
  }

  stages {
    stage('Checkout info') {
      steps { sh 'git log -1 --oneline || true' }
    }

    stage('Docker Build') {
      steps {
        sh '''
          set -e
          docker rm -f test-movie test-cast || true
          docker build -t $DOCKER_ID/$MOVIE_IMAGE:$BUILD_TAG ./movie-service
          docker build -t $DOCKER_ID/$CAST_IMAGE:$BUILD_TAG  ./cast-service
        '''
      }
    }

    stage('Smoke test containers') {
      steps {
        script {
          catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
            sh '''
              set -e
              docker run -d --name test-movie -p 8081:8000 $DOCKER_ID/$MOVIE_IMAGE:$BUILD_TAG
              docker run -d --name test-cast  -p 8082:8000 $DOCKER_ID/$CAST_IMAGE:$BUILD_TAG
              sleep 8
              curl -sf http://localhost:8081/docs >/dev/null
              curl -sf http://localhost:8082/docs >/dev/null
            '''
          }
        }
        sh 'docker rm -f test-movie test-cast || true'
      }
    }



    stage('Docker Push') {
      environment { DOCKER_PASS = credentials('DOCKER_HUB_PASS') } // Secret text
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

          # 1) DBs d'abord
          kubectl -n dev apply -f k8s/postgres-movie.yaml
          kubectl -n dev apply -f k8s/postgres-cast.yaml
          kubectl -n dev rollout status deploy/movie-db -n dev --timeout=120s
          kubectl -n dev rollout status deploy/cast-db  -n dev --timeout=120s

          # 2) Apps (Helm) — conteneurs écoutent 8000, service 80 -> 8000
          helm upgrade --install movie-api charts \
            --set image.repository=$DOCKER_ID/$MOVIE_IMAGE \
            --set image.tag=$BUILD_TAG \
            --set service.port=80 \
            --namespace dev --create-namespace --wait --atomic

          helm upgrade --install cast-api charts \
            --set image.repository=$DOCKER_ID/$CAST_IMAGE \
            --set image.tag=$BUILD_TAG \
            --set service.port=80 \
            --namespace dev --create-namespace --wait --atomic

          # 3) Injecter l'ENV (si le chart ne le supporte pas nativement) + restart
          kubectl -n dev set env deploy/movie-api-fastapiapp \
            DATABASE_URI=postgresql://movie:moviepass@movie-db:5432/moviedb
          kubectl -n dev set env deploy/cast-api-fastapiapp \
            DATABASE_URI=postgresql://cast:castpass@cast-db:5432/castdb

          kubectl -n dev rollout restart deploy movie-api-fastapiapp cast-api-fastapiapp
          kubectl -n dev rollout status deploy/movie-api-fastapiapp --timeout=180s
          kubectl -n dev rollout status deploy/cast-api-fastapiapp  --timeout=180s

          echo -n "movie image (dev): ";  kubectl -n dev get deploy movie-api-fastapiapp -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
          echo -n "cast  image (dev): ";  kubectl -n dev get deploy cast-api-fastapiapp  -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
        '''
      }
    }

    stage('Deploy qa') {
      environment { KUBECONFIG = credentials('config') }
      steps {
        sh '''
          set -e
          mkdir -p .kube && cat $KUBECONFIG > .kube/config
          helm upgrade --install movie-api charts \
            --set image.repository=$DOCKER_ID/$MOVIE_IMAGE --set image.tag=$BUILD_TAG --set service.port=80 \
            --namespace qa --create-namespace --wait --atomic

          helm upgrade --install cast-api charts \
            --set image.repository=$DOCKER_ID/$CAST_IMAGE --set image.tag=$BUILD_TAG --set service.port=80 \
            --namespace qa --create-namespace --wait --atomic
        '''
      }
    }

    stage('Deploy staging') {
      environment { KUBECONFIG = credentials('config') }
      steps {
        sh '''
          set -e
          mkdir -p .kube && cat $KUBECONFIG > .kube/config
          helm upgrade --install movie-api charts \
            --set image.repository=$DOCKER_ID/$MOVIE_IMAGE --set image.tag=$BUILD_TAG --set service.port=80 \
            --namespace staging --create-namespace --wait --atomic

          helm upgrade --install cast-api charts \
            --set image.repository=$DOCKER_ID/$CAST_IMAGE --set image.tag=$BUILD_TAG --set service.port=80 \
            --namespace staging --create-namespace --wait --atomic
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
          helm upgrade --install movie-api charts \
            --set image.repository=$DOCKER_ID/$MOVIE_IMAGE --set image.tag=$BUILD_TAG --set service.port=80 \
            --namespace prod --create-namespace --wait --atomic

          helm upgrade --install cast-api charts \
            --set image.repository=$DOCKER_ID/$CAST_IMAGE --set image.tag=$BUILD_TAG --set service.port=80 \
            --namespace prod --create-namespace --wait --atomic

          echo -n "movie image (prod): "; kubectl -n prod get deploy movie-api-fastapi -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
          echo -n "cast  image (prod): "; kubectl -n prod get deploy cast-api-fastapi  -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
        '''
      }
    }
  }

  post {
    success { echo "OK: $BUILD_TAG déployé" }
    failure { echo "FAILED build $BUILD_NUMBER" }
  }
}
