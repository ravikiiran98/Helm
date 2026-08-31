pipeline {
agent any

environment {
    IMAGE_NAME = "ravikiiran/hello-world"
    IMAGE_TAG = "${BUILD_NUMBER}"
    DOCKER_CREDENTIALS = "dockerhub-creds"

    HELM_RELEASE = "hello-world"
    HELM_CHART = "./helm/hello-world"

    KUBE_CONTEXT = "kind-myapp-cluster"
    KUBECONFIG_PATH = "/var/lib/jenkins/.kube/config"
}

stages {

    stage('Build') {
        steps {
            sh 'mvn clean package -DskipTests'
        }
    }


    stage('Docker Build') {
        steps {
            sh '''
                docker build \
                    -t ${IMAGE_NAME}:${IMAGE_TAG} \
                    -t ${IMAGE_NAME}:latest \
                    .
            '''
        }
    }


    stage('Push to Docker Hub') {
        steps {
            withCredentials([
                usernamePassword(
                    credentialsId: "${DOCKER_CREDENTIALS}",
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )
            ]) {
                sh '''
                    echo "$DOCKER_PASSWORD" | docker login \
                        -u "$DOCKER_USERNAME" \
                        --password-stdin

                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    docker push ${IMAGE_NAME}:latest

                    docker logout
                '''
            }
        }
    }


    stage('Setup Prometheus Monitoring') {
        steps {
            sh '''
                export KUBECONFIG=${KUBECONFIG_PATH}

                kubectl config use-context ${KUBE_CONTEXT}

                echo "===== Checking Monitoring Namespace ====="

                kubectl get namespace monitoring >/dev/null 2>&1 || \
                kubectl create namespace monitoring


                echo "===== Checking Prometheus ====="

                if ! kubectl get deployment prometheus \
                    -n monitoring >/dev/null 2>&1
                then

                    echo "Prometheus not installed."
                    echo "Installing Prometheus..."

                    kubectl apply \
                        -f monitoring/prometheus-rbac.yaml

                    kubectl apply \
                        -f monitoring/prometheus-config.yaml

                    kubectl apply \
                        -f monitoring/prometheus.yaml

                else

                    echo "Prometheus already installed."

                fi


                echo "===== Waiting for Prometheus ====="

                kubectl rollout status \
                    deployment/prometheus \
                    -n monitoring \
                    --timeout=5m
            '''
        }
    }


    stage('Deploy with Helm') {
        steps {
            sh '''
                export KUBECONFIG=${KUBECONFIG_PATH}

                kubectl config use-context ${KUBE_CONTEXT}

                helm upgrade --install ${HELM_RELEASE} ${HELM_CHART} \
                    --set image.repository=${IMAGE_NAME} \
                    --set image.tag=${IMAGE_TAG} \
                    --set image.pullPolicy=Always \
                    --wait \
                    --timeout 5m
            '''
        }
    }


    stage('Verify Deployment') {
        steps {
            sh '''
                export KUBECONFIG=${KUBECONFIG_PATH}

                kubectl config use-context ${KUBE_CONTEXT}

                echo "===== Helm Release ====="
                helm list

                echo "===== Pods ====="
                kubectl get pods -o wide

                echo "===== Services ====="
                kubectl get svc

                echo "===== Deployment ====="
                kubectl rollout status \
                    deployment/${HELM_RELEASE}-hello-world
            '''
        }
    }


    stage('Monitor Pod CPU Usage') {
    steps {
        sh '''
            export KUBECONFIG=/var/lib/jenkins/.kube/config

            echo "===== Checking Prometheus ====="

            kubectl get pods -n monitoring

            echo "===== Starting Prometheus Port Forward ====="

            LOG_FILE=$WORKSPACE/prometheus.log

            kubectl port-forward \
                -n monitoring \
                svc/prometheus \
                9090:9090 \
                > $LOG_FILE 2>&1 &

            PROM_PID=$!

            sleep 10

            echo "===== Pod CPU Usage ====="

            curl -sG \
                http://localhost:9090/api/v1/query \
                --data-urlencode \
                'query=sum(rate(container_cpu_usage_seconds_total{container!="",pod=~"hello-world-.*"}[5m])) by (pod)'

            echo ""

            echo "===== Stopping Prometheus Port Forward ====="

            kill $PROM_PID 2>/dev/null || true
        '''
    }
}
}
}

