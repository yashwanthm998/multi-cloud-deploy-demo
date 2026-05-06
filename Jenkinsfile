pipeline {
    agent {
        docker {
            image 'amazon/aws-cli'
            args '-u root'
        }
    }

    environment {
        AWS_REGION = "ap-southeast-1"
        CLUSTER_NAME = "hello-cluster"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install kubectl') {
            steps {
                sh '''
                curl -LO https://dl.k8s.io/release/v1.30.0/bin/linux/amd64/kubectl

                chmod +x kubectl

                mv kubectl /usr/local/bin/

                kubectl version --client
                '''
            }
        }

        stage('Configure EKS Access') {
            steps {
                sh '''
                aws eks update-kubeconfig \
                  --region $AWS_REGION \
                  --name $CLUSTER_NAME

                kubectl get nodes
                '''
            }
        }

        stage('Deploy Application') {
            steps {
                sh '''
                kubectl apply -f k8s/deployment.yaml

                kubectl apply -f k8s/service.yaml
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                kubectl get pods

                kubectl get svc
                '''
            }
        }
    }
}