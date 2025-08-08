``` bash


sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```



``` jenkinsfile
pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-creds') // Docker Hub username/password
        DOCKERHUB_USERNAME = "${DOCKERHUB_USR}"
        DOCKERHUB_PASSWORD = "${DOCKERHUB_PSW}"
        IMAGE_NAME = 'subkamble/star-agile-insurance-project'
    }

    stages {
        stage('Checkout from GitHub') {
            steps {
                git branch: 'master', url: 'https://github.com/longhippo/star-agile-insurance-project.git'
            }
        }

        stage('Maven Build & Test') {
            steps {
                sh 'mvn clean test'
                sh 'mvn package -DskipTests=false' // Or mvn install
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    
                    dockerImage = docker.build("${IMAGE_NAME}:${BUILD_NUMBER}")
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-creds') {
                        dockerImage.push()
                        dockerImage.push('latest')
                    }
                }
            }
        }
        stage('Deploy Docker Image on Port 3000') {
            steps {
                script {
                    sh """
                        
                        docker pull ${IMAGE_NAME}:latest

                        # Stop and remove any existing container with the same name
                        docker rm -f ${IMAGE_NAME} || true
                        docker rm -f star-agile-insurance-project
                        # Run the container on port 3000
                        docker run -d --name star-agile-insurance-project -p 3000:8081 subkamble/star-agile-insurance-project:latest

                    """
                }
            }
        }
        stage('Deploy to Kubernetes') {
    steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
            sh '''
                echo "Using kubeconfig: $KUBECONFIG"
                kubectl apply -f deployment.yaml
                kubectl apply -f service.yaml

                echo "app is running on port <ip>:30080"
                
                
            '''
        }
    }
}


        
    }

    post {
        success {
            echo "Deployment successful!"
        }
        failure {
            echo "Deployment failed!"
        }
    }
}
````


Use Kubeconfig in Jenkins (for local clusters or remote K8s access)
On a machine where kubectl works, run:

``` bash
cat ~/.kube/config
```
Copy the contents and store them in Jenkins as a Secret file:

Go to: Jenkins → Manage Jenkins → Credentials

ID: kubeconfig

Update your Jenkinsfile to use it:

``` groovy
stage('Deploy to Kubernetes') {
    steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
            sh '''
                echo "Using kubeconfig: $KUBECONFIG"
                kubectl apply -f k8s/deployment.yaml
                kubectl apply -f k8s/service.yaml
            '''
        }
    }
}
```


``` bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm version
```

### prometheus and grafana 
``` bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add stable https://charts.helm.sh/stable
helm repo update
kubectl create namespace monitoring
helm install kind-prometheus prometheus-community/kube-prometheus-stack --namespace monitoring --set prometheus.service.nodePort=30000 --set prometheus.service.type=NodePort --set grafana.service.nodePort=31000 --set grafana.service.type=NodePort --set alertmanager.service.nodePort=32000 --set alertmanager.service.type=NodePort --set prometheus-node-exporter.service.nodePort=32001 --set prometheus-node-exporter.service.type=NodePort
kubectl get svc -n monitoring
kubectl get namespace
```

``` bash
kubectl port-forward svc/kind-prometheus-kube-prome-prometheus -n monitoring 9090:9090 --address=0.0.0.0 &
kubectl port-forward svc/kind-prometheus-grafana -n monitoring 31000:80 --address=0.0.0.0 &
```


``` pomql
sum (rate (container_cpu_usage_seconds_total{namespace="default"}[1m])) / sum (machine_cpu_cores) * 100

sum (container_memory_usage_bytes{namespace="default"}) by (pod)


sum(rate(container_network_receive_bytes_total{namespace="default"}[5m])) by (pod)
sum(rate(container_network_transmit_bytes_total{namespace="default"}[5m])) by (pod)

```



