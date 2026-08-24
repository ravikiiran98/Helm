pipeline {
    agent any

    environment {
        IMAGE_NAME = "ravikiiran/hello-world"
        IMAGE_TAG = "5.0"
        DOCKER_CREDENTIALS = "dockerhub-creds"
        HELM_RELEASE = "hello-world"
        HELM_CHART = "./helm/hello-world"
        KUBE_CONTEXT = "kind-myapp-cluster"
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

        stage('Deploy with Helm') {
            steps {
                sh '''
		    export KUBECONFIG=/var/lib/jenkins/.kube/config
	
                    kubectl config use-context ${KUBE_CONTEXT}

                    helm upgrade --install ${HELM_RELEASE} ${HELM_CHART} \
                        --set image.repository=${IMAGE_NAME} \
                        --set image.tag=${IMAGE_TAG} \
                        --set image.pullPolicy=Always \
                      #  --set replicaCount=4 \
                        --wait \
                        --timeout 5m
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    kubectl config use-context ${KUBE_CONTEXT}

                    echo "===== Helm Release ====="
                    helm list

                    echo "===== Pods ====="
                    kubectl get pods -o wide

                    echo "===== Services ====="
                    kubectl get svc

                    echo "===== Deployment ====="
                    kubectl rollout status deployment/${HELM_RELEASE}-hello-world
                '''
            }
        }
    }
}
