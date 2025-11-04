pipeline {
  agent any

  environment {
    IMAGE = 'huytm1996/app'
    NAMESPACE = 'prod'
    DOCKER_CRED = 'dockerhub-cred'
    SMOKE_URL = '' // nếu cần set URL trực tiếp
    DOCKERHUB_CREDENTIALS = credentials('dockerhub-creds')
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build & Tag') {
      steps {
        script {
         // tag theo build number or sha
          env.NEW_TAG = "canary-${env.BUILD_NUMBER}"
          sh "docker build -t ${IMAGE}:${env.NEW_TAG} ."
        }
      }
    }

    stage('Push') {
      steps {
                 sh '''
                   echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin
                     docker push ${IMAGE}:${env.NEW_TAG}
                    '''
       
      }
    }

    stage('Deploy Canary') {
      steps {
        script {
          // đảm bảo namespace tồn tại
          sh "kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -"

          // apply stable (nếu chưa có) - optional
          sh "kubectl apply -f app-stable.yaml -n ${NAMESPACE} || true"

          // render canary manifest tạm (thay tag) và apply
          sh """
            sed 's|CANARY_TAG|${NEW_TAG}|g' app-canary.yaml > /tmp/app-canary-${NEW_TAG}.yaml
            kubectl apply -f /tmp/app-canary-${NEW_TAG}.yaml -n ${NAMESPACE}
          """

          // chờ canary ready (timeout)
          sh "kubectl rollout status deployment/app-canary -n ${NAMESPACE} --timeout=120s"
        }
      }
    }

    stage('Smoke Test Canary') {
      steps {
        script {
          // Lấy NodePort hoặc ClusterIP+port để test
          // Giả sử service NodePort:
          NODEPORT = sh(script: "kubectl get svc app-service -n ${NAMESPACE} -o jsonpath='{.spec.ports[0].nodePort}'", returnStdout: true).trim()
          NODEIP = sh(script: "kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type==\"InternalIP\")].address}'", returnStdout: true).trim()
          echo "Testing ${NODEIP}:${NODEPORT}"

          // thử 5 lần, yêu cầu 3 lần pass
          int success = 0
          for (i=0;i<5;i++) {
            def out = sh(script: "curl -sS --max-time 5 http://${NODEIP}:${NODEPORT}/health || true", returnStdout: true).trim()
            if (out.contains('OK') || out == 'ok') { success++ }
            sleep 3
          }
          if (success < 3) {
            error "Canary smoke test failed: ${success}/5"
          } else {
            echo "Canary smoke test passed: ${success}/5"
          }
        }
      }
    }

    stage('Promote Canary') {
      steps {
        script {
          // Tăng canary và giảm stable dần (strategy: here 3->0)
          sh """
            kubectl scale deployment app-canary --replicas=3 -n ${NAMESPACE}
            sleep 5
            kubectl scale deployment app-stable --replicas=0 -n ${NAMESPACE}
          """
        }
      }
    }
  }

  post {
    failure {
      script {
        // rollback: delete canary
        sh "kubectl delete deployment app-canary -n ${NAMESPACE} || true"
        // ensure stable exists and scale lại
        sh "kubectl scale deployment app-stable --replicas=9 -n ${NAMESPACE} || kubectl apply -f app-stable.yaml -n ${NAMESPACE} || true"
      }
      echo "Deployment failed — rolled back canary."
    }
    success {
      echo "Canary promoted to production."
    }
  }
}
