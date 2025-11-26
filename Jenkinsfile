pipeline {
    agent any

    environment {
        IMAGE_NAME = "hello-web"
        IMAGE_TAG  = "1.0.1"
        NEXUS_URL  = "http://172.17.0.2:8082"   // Use HTTP explicitly
        NEXUS_USER = "admin"
        NAMESPACE  = "hello"
    }

    stages {
        stage('Checkout SCM') {
            steps {
                git branch: 'main', url: 'https://github.com/Shweatpandit/hello-web-main.git'
            }
        }

        stage('Build JAR') {
            steps {
                sh 'mvn clean package -DskipTests -B'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                """
            }
        }

        stage('Push to Nexus') {
            steps {
                withCredentials([string(credentialsId: 'NEXUS_PASS', variable: 'NPASS')]) {
                    sh """
                        # Login using HTTP
                        echo \$NPASS | docker login ${NEXUS_URL} -u ${NEXUS_USER} --password-stdin
                        docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${NEXUS_URL}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${NEXUS_URL}/${IMAGE_NAME}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Create imagePullSecret in Kubernetes') {
            steps {
                withCredentials([string(credentialsId: 'NEXUS_PASS', variable: 'NPASS')]) {
                    sh """
                        # Create namespace if it doesn't exist
                        kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                        # Delete existing secret if any
                        kubectl delete secret regcred -n ${NAMESPACE} --ignore-not-found
                        # Create new docker-registry secret
                        kubectl create secret docker-registry regcred \
                            --docker-server=${NEXUS_URL} \
                            --docker-username=${NEXUS_USER} \
                            --docker-password=\$NPASS \
                            -n ${NAMESPACE}
                    """
                }
            }
        }

        stage('Deploy to Minikube') {
            steps {
                sh """
                    # Apply Deployment and Service manifests
                    kubectl apply -n ${NAMESPACE} -f k8s/deployment.yaml
                    kubectl apply -n ${NAMESPACE} -f k8s/service.yaml
                """
            }
        }

        stage('Post Actions: Check Status') {
            steps {
                sh """
                    echo === Pods Status ===
                    kubectl get pods -n ${NAMESPACE} -o wide
                    echo === Services Status ===
                    kubectl get svc -n ${NAMESPACE} -o wide
                """
            }
        }
    }

    post {
        failure {
            echo "Pipeline failed! Check the logs above."
        }
        success {
            echo "Pipeline completed successfully!"
        }
    }
}



