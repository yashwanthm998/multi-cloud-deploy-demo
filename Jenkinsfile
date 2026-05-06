properties([
    parameters([
        choice(
            name: 'CLOUD_PROVIDER',
            choices: ['aws', 'gcp'],
            description: 'Choose cloud'
        ),

        string(
            name: 'NAMESPACE',
            defaultValue: 'default',
            description: 'Kubernetes namespace'
        ),

        choice(
            name: 'ACTION',
            choices: ['deploy', 'delete'],
            description: 'Choose action'
        )
    ])
])

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
      image: google/cloud-sdk:latest
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

            ./aws/install || true

            gcloud components install gke-gcloud-auth-plugin -q || true

            kubectl version --client
            '''
        }
    }

    stage('Configure Cluster Access') {

        container('tools') {

            script {

                if (params.CLOUD_PROVIDER == "aws") {

                    withCredentials([[
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-creds'
                    ]]) {

                        sh '''
                        aws eks update-kubeconfig \
                          --region ap-southeast-1 \
                          --name hello-cluster

                        kubectl get nodes
                        '''
                    }
                }

                if (params.CLOUD_PROVIDER == "gcp") {

                    withCredentials([
                        file(
                            credentialsId: 'gcp-sa-key',
                            variable: 'GCP_KEY'
                        )
                    ]) {

                        sh '''
                        export GOOGLE_APPLICATION_CREDENTIALS=$GCP_KEY

                        gcloud auth activate-service-account \
                          --key-file=$GOOGLE_APPLICATION_CREDENTIALS

                        gcloud config set project gke-qa2-36938

                        gcloud container clusters get-credentials \
                          gke-qa2-sg1 \
                          --zone asia-southeast1 \
                          --project gke-qa2-36938

                        kubectl get nodes
                        '''
                    }
                }
            }
        }
    }

    stage('Deploy/Delete Application') {

        container('tools') {

            script {

                if (params.ACTION == "deploy") {

                    sh """
                    kubectl apply -n ${params.NAMESPACE} \
                      -f k8s/deployment.yaml

                    kubectl apply -n ${params.NAMESPACE} \
                      -f k8s/service.yaml

                    kubectl get pods -n ${params.NAMESPACE}

                    kubectl get svc -n ${params.NAMESPACE}
                    """
                }

                if (params.ACTION == "delete") {

                    sh """
                    kubectl delete -n ${params.NAMESPACE} \
                      -f k8s/deployment.yaml || true

                    kubectl delete -n ${params.NAMESPACE} \
                      -f k8s/service.yaml || true
                    """
                }
            }
        }
    }
}
}