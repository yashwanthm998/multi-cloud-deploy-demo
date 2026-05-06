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
      image: lachlanevenson/k8s-kubectl:v1.30.0
      command:
        - cat
      tty: true
'''
) {

    node(POD_LABEL) {

        stage('Checkout') {
            checkout scm
        }

        stage('Install AWS CLI') {

            container('tools') {

                sh '''
                curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

                unzip awscliv2.zip

                ./aws/install --bin-dir /usr/local/bin --install-dir /usr/local/aws-cli --update

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
                    mkdir -p /home/jenkins/.kube

                    aws eks update-kubeconfig \
                      --region ap-southeast-1 \
                      --name hello-cluster \
                      --kubeconfig /home/jenkins/.kube/config

                    export KUBECONFIG=/home/jenkins/.kube/config

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
                    export KUBECONFIG=/home/jenkins/.kube/config

                    kubectl apply -f k8s/deployment.yaml

                    kubectl apply -f k8s/service.yaml

                    kubectl get pods

                    kubectl get svc
                    '''
                }
            }
        }
    }
}