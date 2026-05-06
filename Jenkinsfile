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

        stage('Install kubectl') {
            steps {
                sh '''
                set -e

                mkdir -p $HOME/bin

                curl -LO https://dl.k8s.io/release/v1.30.0/bin/linux/amd64/kubectl

                chmod +x kubectl

                mv kubectl $HOME/bin/

                kubectl version --client
                '''
            }
        }

        stage('Configure EKS Access') {
            steps {
                sh '''
                set -e

                docker run --rm \
                  -v $HOME/.kube:/root/.kube \
                  -v $(pwd):/workdir \
                  -w /workdir \
                  amazon/aws-cli \
                  eks update-kubeconfig \
                  --region $AWS_REGION \
                  --name $CLUSTER_NAME

                kubectl get nodes
                '''
            }
        }

        stage('Deploy Application') {
            steps {
                sh '''
                set -e

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

    post {
        success {
            echo 'Deployment completed successfully 🚀'
        }

        failure {
            echo 'Pipeline failed ❌'
        }
    }
}