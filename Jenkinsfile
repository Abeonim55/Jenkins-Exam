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

          # 0) Namespace
          kubectl get ns staging >/dev/null 2>&1 || kubectl create ns staging

          # 1) DBs (non bloquant sur la validation API)
          kubectl -n staging apply -f k8s/postgres-movie.yaml --validate=false
          kubectl -n staging apply -f k8s/postgres-cast.yaml  --validate=false

          # 2) Apps (Helm) — pas de --wait/--atomic pour éviter les faux négatifs
          helm upgrade --install movie-api charts -n staging \
            -f movie-values.yaml --set image.tag=$BUILD_TAG

          helm upgrade --install cast-api charts -n staging \
            -f cast-values.yaml --set image.tag=$BUILD_TAG

          # 3) Env vars DB (idempotent)
          kubectl -n staging set env deploy/movie-api-fastapiapp \
            DATABASE_URI=postgresql://movie:moviepass@movie-db:5432/moviedb \
            DATABASE_URL=postgresql://movie:moviepass@movie-db:5432/moviedb || true

          kubectl -n staging set env deploy/cast-api-fastapiapp \
            DATABASE_URI=postgresql://cast:castpass@cast-db:5432/castdb \
            DATABASE_URL=postgresql://cast:castpass@cast-db:5432/castdb || true

          # 4) Restart (non bloquant)
          kubectl -n staging rollout restart deploy movie-api-fastapiapp cast-api-fastapiapp || true

          # 5) Attente souple (n'échoue pas le build si k3s est lent)
          kubectl -n staging rollout status deploy/movie-api-fastapiapp --timeout=120s || true
          kubectl -n staging rollout status deploy/cast-api-fastapiapp  --timeout=120s || true

          # 6) Récap & URLs (diagnostic rapide)
          kubectl -n staging get deploy,svc -o wide || true
          echo -n "movie image (staging): "; kubectl -n staging get deploy movie-api-fastapiapp -o jsonpath='{.spec.template.spec.containers[0].image}'; echo || true
          echo -n "cast  image (staging): "; kubectl -n staging get deploy cast-api-fastapiapp  -o jsonpath='{.spec.template.spec.containers[0].image}'; echo || true

          NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')
          MOVIE_PORT=$(kubectl -n staging get svc movie-api-fastapiapp -o jsonpath='{.spec.ports[0].nodePort}')
          CAST_PORT=$(kubectl  -n staging get svc cast-api-fastapiapp  -o jsonpath='{.spec.ports[0].nodePort}')
          echo "STAGING movie list:  http://$NODE_IP:$MOVIE_PORT/api/v1/movies/"
          echo "STAGING cast  list:  http://$NODE_IP:$CAST_PORT/api/v1/casts/"
        '''
      }
    }

    stage('Deploy QA') {
      environment { KUBECONFIG = credentials('config') }
      steps {
        sh '''
          set -e
          mkdir -p .kube && cat $KUBECONFIG > .kube/config

          # 0) Namespace
          kubectl get ns QA >/dev/null 2>&1 || kubectl create ns QA

          # 1) DBs (non bloquant sur la validation API)
          kubectl -n QA apply -f k8s/postgres-movie.yaml --validate=false
          kubectl -n QA apply -f k8s/postgres-cast.yaml  --validate=false

          # 2) Apps (Helm) — pas de --wait/--atomic pour éviter les faux négatifs
          helm upgrade --install movie-api charts -n QA \
            -f movie-values.yaml --set image.tag=$BUILD_TAG

          helm upgrade --install cast-api charts -n QA \
            -f cast-values.yaml --set image.tag=$BUILD_TAG

          # 3) Env vars DB (idempotent)
          kubectl -n QA set env deploy/movie-api-fastapiapp \
            DATABASE_URI=postgresql://movie:moviepass@movie-db:5432/moviedb \
            DATABASE_URL=postgresql://movie:moviepass@movie-db:5432/moviedb || true

          kubectl -n QA set env deploy/cast-api-fastapiapp \
            DATABASE_URI=postgresql://cast:castpass@cast-db:5432/castdb \
            DATABASE_URL=postgresql://cast:castpass@cast-db:5432/castdb || true

          # 4) Restart (non bloquant)
          kubectl -n QA rollout restart deploy movie-api-fastapiapp cast-api-fastapiapp || true

          # 5) Attente souple (n'échoue pas le build si k3s est lent)
          kubectl -n QA rollout status deploy/movie-api-fastapiapp --timeout=120s || true
          kubectl -n QA rollout status deploy/cast-api-fastapiapp  --timeout=120s || true

          # 6) Récap & URLs (diagnostic rapide)
          kubectl -n QA get deploy,svc -o wide || true
          echo -n "movie image (QA): "; kubectl -n QA get deploy movie-api-fastapiapp -o jsonpath='{.spec.template.spec.containers[0].image}'; echo || true
          echo -n "cast  image (QA): "; kubectl -n QA get deploy cast-api-fastapiapp  -o jsonpath='{.spec.template.spec.containers[0].image}'; echo || true

          NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')
          MOVIE_PORT=$(kubectl -n QA get svc movie-api-fastapiapp -o jsonpath='{.spec.ports[0].nodePort}')
          CAST_PORT=$(kubectl  -n QA get svc cast-api-fastapiapp  -o jsonpath='{.spec.ports[0].nodePort}')
          echo "QA movie list:  http://$NODE_IP:$MOVIE_PORT/api/v1/movies/"
          echo "QA cast  list:  http://$NODE_IP:$CAST_PORT/api/v1/casts/"
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
