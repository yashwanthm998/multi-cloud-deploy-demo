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
    - name: aws
      image: amazon/aws-cli
      command:
        - sleep
      args:
        - "99d"
      tty: true

    - name: kubectl
      image: bitnami/kubectl:latest
      command:
        - sleep
      args:
        - "99d"
      tty: true
'''
) {

    node(POD_LABEL) {

        stage('Checkout') {
            checkout scm
        }

        stage('Configure EKS Access') {

            container('aws') {

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

                    cat /home/jenkins/.kube/config
                    '''
                }
            }
        }

        stage('Deploy Application') {

            container('kubectl') {

                sh '''
                export KUBECONFIG=/home/jenkins/.kube/config

                kubectl get nodes

                kubectl apply -f k8s/deployment.yaml

                kubectl apply -f k8s/service.yaml

                kubectl get pods

                kubectl get svc
                '''
            }
        }
    }
}