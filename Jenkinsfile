podTemplate(
    containers: [

        containerTemplate(
            name: 'aws',
            image: 'amazon/aws-cli',
            command: 'sleep',
            args: '99d',
            ttyEnabled: true
        ),

        containerTemplate(
            name: 'kubectl',
            image: 'bitnami/kubectl:latest',
            command: 'sleep',
            args: '99d',
            ttyEnabled: true
        )
    ]
) {

    node(POD_LABEL) {

        stage('Checkout') {
            checkout scm
        }

        stage('Configure EKS Access') {

            container('aws') {

                sh '''
                aws eks update-kubeconfig \
                  --region ap-southeast-1 \
                  --name hello-cluster
                '''
            }
        }

        stage('Deploy Application') {

            container('kubectl') {

                sh '''
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