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
          helm upgrade --install movie-api charts -n dev -f movie-values.yaml --set image.tag=$BUILD_TAG --wait --atomic
          helm upgrade --install cast-api  charts -n dev -f cast-values.yaml  --set image.tag=$BUILD_TAG --wait --atomic

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

          # 1) DBs d'abord (namespace staging)
          kubectl -n staging apply -f k8s/postgres-movie.yaml
          kubectl -n staging apply -f k8s/postgres-cast.yaml
          kubectl -n staging rollout status deploy/movie-db --timeout=120s
          kubectl -n staging rollout status deploy/cast-db  --timeout=120s

          # 2) Apps (Helm) - valeurs par défaut du chart déjà correctes (8000->80)
          helm upgrade --install movie-api charts \
            --set image.repository=$DOCKER_ID/$MOVIE_IMAGE \
            --set image.tag=$BUILD_TAG \
            --namespace staging --create-namespace --wait --atomic

          helm upgrade --install cast-api charts \
            --set image.repository=$DOCKER_ID/$CAST_IMAGE \
            --set image.tag=$BUILD_TAG \
            --namespace staging --create-namespace --wait --atomic

          # 3) Env vars (comme en dev)
          kubectl -n staging set env deploy/movie-api-fastapiapp \
            DATABASE_URI=postgresql://movie:moviepass@movie-db:5432/moviedb \
            DATABASE_URL=postgresql://movie:moviepass@movie-db:5432/moviedb

          kubectl -n staging set env deploy/cast-api-fastapiapp \
            DATABASE_URI=postgresql://cast:castpass@cast-db:5432/castdb \
            DATABASE_URL=postgresql://cast:castpass@cast-db:5432/castdb

          # 4) Restart + attente
          kubectl -n staging rollout restart deploy movie-api-fastapiapp cast-api-fastapiapp
          kubectl -n staging rollout status  deploy/movie-api-fastapiapp --timeout=180s
          kubectl -n staging rollout status  deploy/cast-api-fastapiapp  --timeout=180s
        '''
      }
    }

    stage('Approval & Deploy prod') {
      when {
        expression {
          // vrai si BRANCH_NAME vaut 'main' (multibranch) OU si on build la branche par défaut
          return (env.BRANCH_NAME == 'main') || (env.GIT_BRANCH?.endsWith('/main')) || (env.CHANGE_TARGET == 'main')
        }
      }
      environment { KUBECONFIG = credentials('config') }
      steps {
        timeout(time: 15, unit: 'MINUTES') {
          input message: 'Déployer en production ?', ok: 'Oui'
        }
        sh '''
          set -e
          mkdir -p .kube && cat $KUBECONFIG > .kube/config

          # 1) DBs d'abord (namespace prod)
          kubectl -n prod apply -f k8s/postgres-movie.yaml
          kubectl -n prod apply -f k8s/postgres-cast.yaml
          kubectl -n prod rollout status deploy/movie-db --timeout=120s
          kubectl -n prod rollout status deploy/cast-db  --timeout=120s

          # 2) Apps (Helm) — on laisse le chart gérer port: 80 -> targetPort: 8000
          #    (pas de nodePort fixe pour éviter les conflits cluster-wide)
          helm upgrade --install movie-api charts \
            --set image.repository=$DOCKER_ID/$MOVIE_IMAGE \
            --set image.tag=$BUILD_TAG \
            --namespace prod --create-namespace --wait --atomic

          helm upgrade --install cast-api charts \
            --set image.repository=$DOCKER_ID/$CAST_IMAGE \
            --set image.tag=$BUILD_TAG \
            --namespace prod --create-namespace --wait --atomic

          # 3) Variables d'env DB (comme en dev/staging)
          kubectl -n prod set env deploy/movie-api-fastapiapp \
            DATABASE_URI=postgresql://movie:moviepass@movie-db:5432/moviedb \
            DATABASE_URL=postgresql://movie:moviepass@movie-db:5432/moviedb

          kubectl -n prod set env deploy/cast-api-fastapiapp \
            DATABASE_URI=postgresql://cast:castpass@cast-db:5432/castdb \
            DATABASE_URL=postgresql://cast:castpass@cast-db:5432/castdb

          # 4) Restart + attente des rollouts
          kubectl -n prod rollout restart deploy movie-api-fastapiapp cast-api-fastapiapp
          kubectl -n prod rollout status  deploy/movie-api-fastapiapp --timeout=180s
          kubectl -n prod rollout status  deploy/cast-api-fastapiapp  --timeout=180s

          # 5) Infos pratiques
          echo -n "movie image (prod): "; kubectl -n prod get deploy movie-api-fastapiapp -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
          echo -n "cast  image (prod): "; kubectl -n prod get deploy cast-api-fastapiapp  -o jsonpath='{.spec.template.spec.containers[0].image}'; echo

          NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')
          MOVIE_PORT=$(kubectl -n prod get svc movie-api-fastapiapp -o jsonpath='{.spec.ports[0].nodePort}')
          CAST_PORT=$(kubectl  -n prod get svc cast-api-fastapiapp  -o jsonpath='{.spec.ports[0].nodePort}')

          echo "PROD movie list:  http://$NODE_IP:$MOVIE_PORT/api/v1/movies/"
          echo "PROD cast list:   http://$NODE_IP:$CAST_PORT/api/v1/casts/"
          echo "PROD cast docs:   http://$NODE_IP:$CAST_PORT/api/v1/casts/docs"
          echo "PROD movie redoc: http://$NODE_IP:$MOVIE_PORT/redoc"
        '''
      }
    }



  }

  post {
    success { echo "OK: ${BUILD_TAG} déployé" }
    failure { echo "FAILED build ${BUILD_NUMBER}" }
  }
}
