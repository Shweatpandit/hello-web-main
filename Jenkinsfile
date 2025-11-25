pipeline {
 agent any


 environment {
   REGISTRY        = "192.168.1.138:8082"
   IMAGE_NAME      = "hello-web"
   IMAGE_TAG       = "1.0.1"
   NEXUS_CREDS_ID  = "nexus-docker"
   KUBECONFIG      = '/home/jenkins/.kube/config'
 }


 stages {


   stage('Checkout') {
     steps {
       checkout scm
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
       withCredentials([usernamePassword(credentialsId: env.NEXUS_CREDS_ID, usernameVariable: 'NUSER', passwordVariable: 'NPASS')]) {
         sh """
           echo "${NPASS}" | docker login ${REGISTRY} -u "${NUSER}" --password-stdin
           docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
           docker push ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
         """
       }
     }
   }


   stage('Create imagePullSecret') {
     steps {
       withCredentials([usernamePassword(credentialsId: env.NEXUS_CREDS_ID, usernameVariable: 'NUSER', passwordVariable: 'NPASS')]) {
         sh """
           kubectl delete secret regcred --ignore-not-found -n hello
           kubectl create secret docker-registry regcred \
             --docker-server=${REGISTRY} \
             --docker-username="${NUSER}" \
             --docker-password="${NPASS}" \
             -n hello
         """
       }
     }
   }


   stage('Deploy to Minikube') {
     steps {
       sh """
         # Ensure namespace exists
         kubectl create namespace hello --dry-run=client -o yaml | kubectl apply -f -
         
         # Apply manifests - direct files use karo kyunki root directory mein hain
         sed "s|REPLACE_REGISTRY|${REGISTRY}|g" deployment.yaml | kubectl apply -n hello -f -
         kubectl apply -n hello -f service.yaml
         
         # Wait for rollout
         kubectl rollout status deployment/hello-web -n hello --timeout=120s
       """
     }
   }


 }


 post {
   always {
     sh '''
       echo "=== Pods Status ==="
       kubectl get pods -n hello -o wide
       echo "=== Services Status ==="  
       kubectl get svc -n hello -o wide
     '''
   }
 }
}
