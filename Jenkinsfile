pipeline {
    agent any

    environment {
        AWS_REGION = "ap-southeast-1"
        CLUSTER_NAME = "hello-cluster"
        PATH = "${env.HOME}/bin:${env.PATH}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Tools') {
            steps {
                sh '''
                set -e

                mkdir -p $HOME/bin

                echo "Installing kubectl..."

                curl -LO https://dl.k8s.io/release/v1.30.0/bin/linux/amd64/kubectl

                chmod +x kubectl

                mv kubectl $HOME/bin/

                echo "Installing AWS CLI..."

                curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

                python3 -m zipfile -e awscliv2.zip .

                ./aws/install -i $HOME/aws-cli -b $HOME/bin

                echo "Verifying installations..."

                kubectl version --client

                aws --version
                '''
            }
        }

        stage('Configure EKS Access') {
            steps {
                sh '''
                set -e

                echo "Configuring kubeconfig for EKS..."

                aws eks update-kubeconfig \
                  --region $AWS_REGION \
                  --name $CLUSTER_NAME

                echo "Connected nodes:"

                kubectl get nodes
                '''
            }
        }

        stage('Deploy Application') {
            steps {
                sh '''
                set -e

                echo "Deploying application..."

                kubectl apply -f k8s/deployment.yaml

                kubectl apply -f k8s/service.yaml
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                echo "Pods status:"

                kubectl get pods

                echo "Services status:"

                kubectl get svc
                '''
            }
        }
    }

    post {
        success {
            echo 'Deployment completed successfully 🚀'
        }

        failure {
            echo 'Pipeline failed ❌'
        }
    }
}