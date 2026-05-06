podTemplate(
    yaml: '''
apiVersion: v1
kind: Pod
spec:
  tolerations:
    - key: "role"
      operator: "Exists"
      effect: "NoSchedule"

    - key: "CriticalAddonsOnly"
      operator: "Exists"

  containers:
    - name: tools
      image: ubuntu:22.04
      command:
        - sleep
      args:
        - "999999"
      tty: true
'''
) {

    node(POD_LABEL) {

        stage('Checkout') {
            checkout scm
        }

        stage('Install Tools') {

            container('tools') {

                sh '''
                apt-get update

                apt-get install -y \
                  curl \
                  unzip \
                  git

                curl -LO "https://dl.k8s.io/release/v1.30.0/bin/linux/amd64/kubectl"

                chmod +x kubectl

                mv kubectl /usr/local/bin/

                curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

                unzip awscliv2.zip

                ./aws/install

                aws --version

                kubectl version --client
                '''
            }
        }

        stage('Configure EKS Access') {

            container('tools') {

                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {

                    sh '''
                    mkdir -p /root/.kube

                    aws eks update-kubeconfig \
                      --region ap-southeast-1 \
                      --name hello-cluster

                    kubectl get nodes
                    '''
                }
            }
        }

        stage('Deploy Application') {

            container('tools') {

                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {

                    sh '''
                    kubectl apply -n default -f k8s/deployment.yaml

                    kubectl apply -n default -f k8s/service.yaml

                    kubectl get pods -n default

                    kubectl get svc -n default
                    '''
                }
            }
        }
    }
}