pipeline {
    agent any

    environment {
        REGISTRY = "localhost:8082"       // Nexus Docker registry
        IMAGE_NAME = "hello-app"          // Your app image name
        KUBECONFIG = "/root/.kube/config" // So kubectl works inside Jenkins container
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                url: 'https://github.com/Shweatpandit/hello-web-main.git'
            }
        }

        stage('Maven Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Docker Build') {
            steps {
                sh """
                    docker build -t ${REGISTRY}/${IMAGE_NAME}:latest .
                """
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus-docker',
                                                 usernameVariable: 'USER',
                                                 passwordVariable: 'PASS')]) {   // <-- CHANGED HERE
                    sh """
                        echo "$PASS" | docker login ${REGISTRY} -u "$USER" --password-stdin
                    """
                }
            }
        }

        stage('Docker Push') {
            steps {
                sh "docker push ${REGISTRY}/${IMAGE_NAME}:latest"
            }
        }

        stage('K8s Deploy') {
            steps {
                sh """
                    kubectl set image deployment/hello-app hello-app=${REGISTRY}/${IMAGE_NAME}:latest --namespace default || \
                    kubectl apply -f k8s/deployment.yaml
                """
            }
        }
    }

    post {
        success {
            echo "🎉 Deployment completed successfully!"
        }
        failure {
            echo "❌ Deployment failed!"
        }
    }
}
