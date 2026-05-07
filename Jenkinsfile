properties([
    parameters([

        choice(
            name: 'CLOUD_PROVIDER',
            choices: ['aws', 'gcp'],
            description: 'Choose cloud provider'
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
    
                // Common tools (always run)
                sh '''
                apt-get update
        
                apt-get install -y \
                  curl \
                  unzip \
                  git
        
                # Install kubectl
                curl -LO "https://dl.k8s.io/release/v1.30.0/bin/linux/amd64/kubectl"
                chmod +x kubectl
                mv kubectl /usr/local/bin/
        
                echo "===== Kubectl Version ====="
                kubectl version --client
                '''
        
                // AWS specific
                if (params.CLOUD_PROVIDER == "aws") {
                    sh '''
                    echo "===== Installing AWS CLI ====="
        
                    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" \
                      -o "awscliv2.zip"
        
                    unzip awscliv2.zip
                    ./aws/install || true
        
                    aws --version
                    '''
                }
        
                // GCP specific
                if (params.CLOUD_PROVIDER == "gcp") {
                    sh '''
                    echo "===== GCP Setup ====="
        
                    # Install GKE auth plugin (if needed)
                    gcloud components install gke-gcloud-auth-plugin -q || true
        
                    gcloud version
                    '''
                }
            }
        }

        stage('Configure Cluster Access') {
    
            container('tools') {
    
                script {
    
                    // ================= AWS =================
    
                    if (params.CLOUD_PROVIDER == "aws") {
    
                        withCredentials([[
                            $class: 'AmazonWebServicesCredentialsBinding',
                            credentialsId: 'aws-creds'
                        ]]) {
    
                            sh '''
                            mkdir -p /root/.kube
    
                            aws eks update-kubeconfig \
                              --region ap-southeast-1 \
                              --name hello-cluster
    
                            echo "===== EKS Nodes ====="
    
                            kubectl get nodes
                            '''
                        }
                    }
    
                    // ================= GCP =================
    
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
                              --project gke-qa2-36938 \
                              --internal-ip
    
                            echo "===== GKE Nodes ====="
    
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
    
                    // ================= AWS =================
    
                    if (params.CLOUD_PROVIDER == "aws") {
    
                        withCredentials([[
                            $class: 'AmazonWebServicesCredentialsBinding',
                            credentialsId: 'aws-creds'
                        ]]) {
    
                            if (params.ACTION == "deploy") {
    
                                sh """
                                echo "===== Deploying to AWS EKS ====="
    
                                kubectl apply -n ${params.NAMESPACE} \
                                  -f k8s/deployment.yaml
    
                                kubectl apply -n ${params.NAMESPACE} \
                                  -f k8s/service.yaml
    
                                echo "===== Pods ====="
    
                                kubectl get pods -n ${params.NAMESPACE}
    
                                echo "===== Services ====="
    
                                kubectl get svc -n ${params.NAMESPACE}
                                """
                            }
    
                            if (params.ACTION == "delete") {
    
                                sh """
                                echo "===== Deleting from AWS EKS ====="
    
                                kubectl delete -n ${params.NAMESPACE} \
                                  -f k8s/deployment.yaml || true
    
                                kubectl delete -n ${params.NAMESPACE} \
                                  -f k8s/service.yaml || true
                                """
                            }
                        }
                    }
    
                    // ================= GCP =================
    
                    if (params.CLOUD_PROVIDER == "gcp") {
    
                        withCredentials([
                            file(
                                credentialsId: 'gcp-sa-key',
                                variable: 'GCP_KEY'
                            )
                        ]) {
    
                            // sh '''
                            // export GOOGLE_APPLICATION_CREDENTIALS=$GCP_KEY
    
                            // gcloud auth activate-service-account \
                            //   --key-file=$GOOGLE_APPLICATION_CREDENTIALS
                            // '''
    
                            if (params.ACTION == "deploy") {
    
                                sh """
                                echo "===== Deploying to GKE ====="
    
                                kubectl apply -n ${params.NAMESPACE} \
                                  -f k8s/deployment.yaml
    
                                kubectl apply -n ${params.NAMESPACE} \
                                  -f k8s/service.yaml
    
                                echo "===== Pods ====="
    
                                kubectl get pods -n ${params.NAMESPACE}
    
                                echo "===== Services ====="
    
                                kubectl get svc -n ${params.NAMESPACE}
                                """
                            }
    
                            if (params.ACTION == "delete") {
    
                                sh """
                                echo "===== Deleting from GKE ====="
    
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
        stage('Deploy Router (Split Traffic)') {
            container('tools') {
                script {

                    if (params.ACTION == "deploy" && params.CLOUD_PROVIDER == "gcp") {

                        sh """
                        echo "===== Deploying NGINX Router ====="

                        kubectl apply -n ${params.NAMESPACE} \
                        -f k8s/nginx-router-config.yaml

                        kubectl apply -n ${params.NAMESPACE} \
                        -f k8s/nginx-router-deployment.yaml

                        kubectl rollout status deployment nginx-router \
                            -n ${params.NAMESPACE} --timeout=120s

                        kubectl apply -n ${params.NAMESPACE} \
                        -f k8s/nginx-router-service.yaml

                        echo "===== Waiting for Service Endpoints ====="

                        until kubectl get endpoints nginx-router-service -n ${params.NAMESPACE} \
                        -o jsonpath='{.subsets[0].addresses[0].ip}' 2>/dev/null; do
                        echo "Waiting for endpoints..."
                        sleep 5
                        done

                        echo "===== Router Service ====="

                        kubectl get svc nginx-router-service -n ${params.NAMESPACE}
                        """
                    }

                    if (params.ACTION == "delete" && params.CLOUD_PROVIDER == "gcp") {

                        sh """
                        echo "===== Deleting NGINX Router ====="

                        kubectl delete -n ${params.NAMESPACE} \
                        -f k8s/nginx-router-deployment.yaml || true

                        kubectl delete -n ${params.NAMESPACE} \
                        -f k8s/nginx-router-service.yaml || true

                        kubectl delete -n ${params.NAMESPACE} \
                        -f k8s/nginx-router-config.yaml || true
                        """
                    }
                }
            }
        }
    }
}
