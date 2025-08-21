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
          for i in $(seq 1 30); do
            if kubectl get --raw='/readyz' >/dev/null 2>&1; then break; fi
            echo "K8s API not ready yet..."; sleep 3
          done

          kubectl -n staging apply -f k8s/postgres-movie.yaml
          kubectl -n staging apply -f k8s/postgres-cast.yaml
          kubectl -n staging rollout status deploy/movie-db --timeout=180s
          kubectl -n staging rollout status deploy/cast-db  --timeout=180s

          helm upgrade --install movie-api charts -n staging -f movie-values.yaml --set image.tag=$BUILD_TAG --wait --atomic
          helm upgrade --install cast-api  charts -n staging -f cast-values.yaml  --set image.tag=$BUILD_TAG --wait --atomic

          # inject DB env (chart-agnostic)
          kubectl -n staging set env deploy/movie-api-fastapiapp \
            DATABASE_URI=postgresql://movie:moviepass@movie-db:5432/moviedb \
            DATABASE_URL=postgresql://movie:moviepass@movie-db:5432/moviedb
          kubectl -n staging set env deploy/cast-api-fastapiapp \
            DATABASE_URI=postgresql://cast:castpass@cast-db:5432/castdb \
            DATABASE_URL=postgresql://cast:castpass@cast-db:5432/castdb

          kubectl -n staging rollout restart deploy movie-api-fastapiapp cast-api-fastapiapp
          kubectl -n staging rollout status  deploy/movie-api-fastapiapp --timeout=180s
          kubectl -n staging rollout status  deploy/cast-api-fastapiapp  --timeout=180s
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

          helm upgrade --install movie-api charts -n prod -f movie-values.yaml --set image.tag=$BUILD_TAG --wait --atomic
          helm upgrade --install cast-api  charts -n prod -f cast-values.yaml  --set image.tag=$BUILD_TAG --wait --atomic

          kubectl -n prod set env deploy/movie-api-fastapiapp \
            DATABASE_URI=postgresql://movie:moviepass@movie-db:5432/moviedb \
            DATABASE_URL=postgresql://movie:moviepass@movie-db:5432/moviedb
          kubectl -n prod set env deploy/cast-api-fastapiapp \
            DATABASE_URI=postgresql://cast:castpass@cast-db:5432/castdb \
            DATABASE_URL=postgresql://cast:castpass@cast-db:5432/castdb

          kubectl -n prod rollout restart deploy movie-api-fastapiapp cast-api-fastapiapp
          kubectl -n prod rollout status  deploy/movie-api-fastapiapp --timeout=180s
          kubectl -n prod rollout status  deploy/cast-api-fastapiapp  --timeout=180s

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
