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
                        docker rm -f health-care-app || true

                        # Run the container on port 3000
                        docker run -d --name health-care-app -p 3000:8081 ${IMAGE_NAME}:latest
                    """
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






