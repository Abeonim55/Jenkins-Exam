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

          # API server readiness guard (prevents "context deadline exceeded")
          for i in $(seq 1 30); do
            if kubectl get --raw='/readyz' >/dev/null 2>&1; then break; fi
            echo "K8s API not ready yet..."; sleep 3
          done

          # DBs
          kubectl -n dev apply -f k8s/postgres-movie.yaml
          kubectl -n dev apply -f k8s/postgres-cast.yaml
          kubectl -n dev rollout status deploy/movie-db --timeout=180s
          kubectl -n dev rollout status deploy/cast-db  --timeout=180s

          # Helm cleanup (in case of previous failed deploys)
          helm -n dev uninstall movie-api  || true
          helm -n dev uninstall cast-api   || true

          # Apps (values files already set; only tag varies)
          helm upgrade --install movie-api charts -n dev -f movie-values.yaml --set image.tag=$BUILD_TAG --wait --atomic
          helm upgrade --install cast-api  charts -n dev -f cast-values.yaml  --set image.tag=$BUILD_TAG --wait --atomic

          # Quick info
          echo -n "movie image (dev): ";  kubectl -n dev get deploy movie-api-fastapiapp -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
          echo -n "cast  image (dev): ";  kubectl -n dev get deploy cast-api-fastapiapp  -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
        '''
      }
    }

    stage('Deploy staging') {
      environment { KUBECONFIG = credentials('config') }
      steps {
        sh '''
          set -e
          mkdir -p .kube && cat $KUBECONFIG > .kube/config

          # helper retry kubectl (tolère les "Timeout" / lenteurs API)
          retry_k() {
            n=0
            until [ $n -ge 5 ]; do
              kubectl --request-timeout=60s "$@" && return 0
              n=$((n+1))
              echo "kubectl retry $n/5: $*"
              sleep $((n*5))
            done
            return 1
          }

          # 1) DBs d'abord
          retry_k -n staging apply -f k8s/postgres-movie.yaml
          retry_k -n staging apply -f k8s/postgres-cast.yaml
          retry_k -n staging rollout status deploy/movie-db --timeout=180s
          retry_k -n staging rollout status deploy/cast-db  --timeout=180s

          # 2) Apps (Helm) — timeout helm augmenté, atomic
          helm upgrade --install movie-api charts \
            -n staging --create-namespace \
            -f movie-values.yaml \
            --set image.tag=$BUILD_TAG \
            --wait --atomic --timeout 10m0s

          helm upgrade --install cast-api charts \
            -n staging --create-namespace \
            -f cast-values.yaml  \
            --set image.tag=$BUILD_TAG \
            --wait --atomic --timeout 10m0s

          # 3) Env vars DB
          retry_k -n staging set env deploy/movie-api-fastapiapp \
            DATABASE_URI=postgresql://movie:moviepass@movie-db:5432/moviedb \
            DATABASE_URL=postgresql://movie:moviepass@movie-db:5432/moviedb

          retry_k -n staging set env deploy/cast-api-fastapiapp \
            DATABASE_URI=postgresql://cast:castpass@cast-db:5432/castdb \
            DATABASE_URL=postgresql://cast:castpass@cast-db:5432/castdb

          # 4) Restart + attente (avec retry)
          retry_k -n staging rollout restart deploy movie-api-fastapiapp cast-api-fastapiapp
          retry_k -n staging rollout status  deploy/movie-api-fastapiapp --timeout=600s
          retry_k -n staging rollout status  deploy/cast-api-fastapiapp  --timeout=600s

          # 5) (facultatif) petit récap pour debug rapide
          retry_k -n staging get deploy,svc -o wide
        '''
      }
    }


    stage('Approval & Deploy prod') {
      when {
        expression {
          (env.BRANCH_NAME == 'main') || (env.GIT_BRANCH?.endsWith('/main')) || (env.CHANGE_TARGET == 'main')
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
          for i in $(seq 1 30); do
            if kubectl get --raw='/readyz' >/dev/null 2>&1; then break; fi
            echo "K8s API not ready yet..."; sleep 3
          done

          kubectl -n prod apply -f k8s/postgres-movie.yaml
          kubectl -n prod apply -f k8s/postgres-cast.yaml
          kubectl -n prod rollout status deploy/movie-db --timeout=180s
          kubectl -n prod rollout status deploy/cast-db  --timeout=180s

          # Helm cleanup (in case of previous failed deploys)
          helm -n prod uninstall movie-api || true
          helm -n prod uninstall cast-api  || true

          helm upgrade --install movie-api charts -n prod -f movie-values.yaml --set image.tag=$BUILD_TAG --wait --atomic
          helm upgrade --install cast-api  charts -n prod -f cast-values.yaml  --set image.tag=$BUILD_TAG --wait --atomic

          kubectl -n prod set env deploy/movie-api-fastapiapp \
            DATABASE_URI=postgresql://movie:moviepass@movie-db:5432/moviedb \
            DATABASE_URL=postgresql://movie:moviepass@movie-db:5432/moviedb
          kubectl -n prod set env deploy/cast-api-fastapiapp \
            DATABASE_URI=postgresql://cast:castpass@cast-db:5432/castdb \
            DATABASE_URL=postgresql://cast:castpass@cast-db:5432/castdb


          # 4) Rollout avec délai + tolérance puis health-check HTTP
          set +e
          kubectl -n prod rollout status deploy/movie-api-fastapiapp  --timeout=600s
          RM=$?
          kubectl -n prod rollout status deploy/cast-api-fastapiapp   --timeout=600s
          RC=$?
          set -e

          # 5) Health checks "réels" (sur les NodePorts)
          NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')
          MOVIE_PORT=$(kubectl -n prod get svc movie-api-fastapiapp -o jsonpath='{.spec.ports[0].nodePort}')
          CAST_PORT=$(kubectl  -n prod get svc cast-api-fastapiapp  -o jsonpath='{.spec.ports[0].nodePort}')

          # movie: redoc OK ?  cast: docs OK ?
          if curl -sf "http://$NODE_IP:$MOVIE_PORT/redoc" >/dev/null && \
            curl -sf "http://$NODE_IP:$CAST_PORT/api/v1/casts/docs" >/dev/null ; then
            echo "Health checks OK (movie=$MOVIE_PORT, cast=$CAST_PORT) ; rollout exitcodes: movie=$RM cast=$RC"
          else
            echo "Health checks FAILED (movie=$MOVIE_PORT, cast=$CAST_PORT)"
            kubectl -n prod get deploy,rs,pods -o wide
            exit 1
          fi

          echo -n "movie image (prod): "; kubectl -n prod get deploy movie-api-fastapiapp -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
          echo -n "cast  image (prod): "; kubectl -n prod get deploy cast-api-fastapiapp  -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
          echo "PROD movie redoc: http://$NODE_IP:$MOVIE_PORT/redoc"
          echo "PROD cast  docs: http://$NODE_IP:$CAST_PORT/api/v1/casts/docs"

        '''
      }
    }
  }

  post {
    success { echo "OK: ${BUILD_TAG} déployé" }
    failure { echo "FAILED build ${BUILD_NUMBER}" }
  }
}
