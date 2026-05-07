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

    // ============================================================
    // Checkout
    // ============================================================

    stage('Checkout') {

        checkout scm
    }

    // ============================================================
    // Install Tools
    // ============================================================

    stage('Install Tools') {

        container('tools') {

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

            kubectl version --client
            '''

            // ================= AWS Tools =================

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

            // ================= GCP Tools =================

            if (params.CLOUD_PROVIDER == "gcp") {

                sh '''
                echo "===== Installing GKE Auth Plugin ====="

                gcloud components install \
                  gke-gcloud-auth-plugin -q || true

                gcloud version
                '''
            }
        }
    }

    // ============================================================
    // Configure Cluster Access
    // ============================================================

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

    // ============================================================
    // Deploy Application In Selected Cloud
    // ============================================================

    stage('Deploy Application In Selected Cloud') {

        container('tools') {

            script {

                // ====================================================
                // Deploy To AWS
                // ====================================================

                if (params.CLOUD_PROVIDER == "aws" &&
                    params.ACTION == "deploy") {

                    withCredentials([[
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-creds'
                    ]]) {

                        sh '''
                        echo "===== Switching Context To AWS ====="

                        aws eks update-kubeconfig \
                          --region ap-southeast-1 \
                          --name hello-cluster

                        echo "===== Current Context ====="

                        kubectl config current-context
                        '''

                        sh """
                        echo "===== Deploying Application To AWS ====="

                        kubectl apply -n ${params.NAMESPACE} \
                          -f k8s/deployment.yaml

                        kubectl apply -n ${params.NAMESPACE} \
                          -f k8s/service.yaml

                        kubectl scale deployment hello-app \
                          --replicas=1 \
                          -n ${params.NAMESPACE}

                        echo "===== AWS Pods ====="

                        kubectl get pods -n ${params.NAMESPACE}

                        echo "===== AWS Services ====="

                        kubectl get svc -n ${params.NAMESPACE}
                        """
                    }
                }

                // ====================================================
                // Deploy To GCP
                // ====================================================

                if (params.CLOUD_PROVIDER == "gcp" &&
                    params.ACTION == "deploy") {

                    withCredentials([
                        file(
                            credentialsId: 'gcp-sa-key',
                            variable: 'GCP_KEY'
                        )
                    ]) {

                        sh '''
                        echo "===== Switching Context To GKE ====="

                        export GOOGLE_APPLICATION_CREDENTIALS=$GCP_KEY

                        gcloud auth activate-service-account \
                          --key-file=$GOOGLE_APPLICATION_CREDENTIALS

                        gcloud config set project gke-qa2-36938

                        gcloud container clusters get-credentials \
                          gke-qa2-sg1 \
                          --zone asia-southeast1 \
                          --project gke-qa2-36938 \
                          --internal-ip

                        echo "===== Current Context ====="

                        kubectl config current-context
                        '''

                        sh """
                        echo "===== Deploying Application To GKE ====="

                        kubectl apply -n ${params.NAMESPACE} \
                          -f k8s/deployment.yaml

                        kubectl apply -n ${params.NAMESPACE} \
                          -f k8s/service.yaml

                        kubectl scale deployment hello-app \
                          --replicas=1 \
                          -n ${params.NAMESPACE}

                        echo "===== GKE Pods ====="

                        kubectl get pods -n ${params.NAMESPACE}

                        echo "===== GKE Services ====="

                        kubectl get svc -n ${params.NAMESPACE}
                        """
                    }
                }

                // ====================================================
                // Delete From AWS
                // ====================================================

                if (params.CLOUD_PROVIDER == "aws" &&
                    params.ACTION == "delete") {

                    withCredentials([[
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-creds'
                    ]]) {

                        sh '''
                        aws eks update-kubeconfig \
                          --region ap-southeast-1 \
                          --name hello-cluster
                        '''

                        sh """
                        echo "===== Deleting From AWS ====="

                        kubectl delete -n ${params.NAMESPACE} \
                          -f k8s/deployment.yaml || true

                        kubectl delete -n ${params.NAMESPACE} \
                          -f k8s/service.yaml || true
                        """
                    }
                }

                // ====================================================
                // Delete From GCP
                // ====================================================

                if (params.CLOUD_PROVIDER == "gcp" &&
                    params.ACTION == "delete") {

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
                        '''

                        sh """
                        echo "===== Deleting From GKE ====="

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

    // ============================================================
    // Scale Down Deployment In Other Cloud
    // ============================================================

    stage('Scale Down Deployment In Other Cloud') {

        container('tools') {

            script {

                // ====================================================
                // Deploying to AWS → Scale down GKE
                // ====================================================

                if (params.CLOUD_PROVIDER == "aws" &&
                    params.ACTION == "deploy") {

                    echo "===== Checking GKE Deployment ====="

                    withCredentials([
                        file(
                            credentialsId: 'gcp-sa-key',
                            variable: 'GCP_KEY'
                        )
                    ]) {

                        sh """
                        export GOOGLE_APPLICATION_CREDENTIALS=\$GCP_KEY

                        gcloud auth activate-service-account \
                          --key-file=\$GOOGLE_APPLICATION_CREDENTIALS

                        gcloud config set project gke-qa2-36938

                        gcloud container clusters get-credentials \
                          gke-qa2-sg1 \
                          --zone asia-southeast1 \
                          --project gke-qa2-36938 \
                          --internal-ip

                        if kubectl get deployment hello-app \
                          -n ${params.NAMESPACE} >/dev/null 2>&1; then

                          CURRENT_REPLICAS=\$(kubectl get deployment hello-app \
                            -n ${params.NAMESPACE} \
                            -o jsonpath='{.spec.replicas}')

                          echo "Current GKE replicas: \$CURRENT_REPLICAS"

                          if [ "\$CURRENT_REPLICAS" != "0" ]; then

                            echo "Scaling down GKE deployment"

                            kubectl scale deployment hello-app \
                              --replicas=0 \
                              -n ${params.NAMESPACE}
                          fi

                        else
                          echo "No deployment found in GKE"
                        fi
                        """
                    }
                }

                // ====================================================
                // Deploying to GCP → Scale down AWS
                // ====================================================

                if (params.CLOUD_PROVIDER == "gcp" &&
                    params.ACTION == "deploy") {

                    echo "===== Checking EKS Deployment ====="

                    withCredentials([[
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-creds'
                    ]]) {

                        sh """
                        aws eks update-kubeconfig \
                          --region ap-southeast-1 \
                          --name hello-cluster

                        if kubectl get deployment hello-app \
                          -n ${params.NAMESPACE} >/dev/null 2>&1; then

                          CURRENT_REPLICAS=\$(kubectl get deployment hello-app \
                            -n ${params.NAMESPACE} \
                            -o jsonpath='{.spec.replicas}')

                          echo "Current EKS replicas: \$CURRENT_REPLICAS"

                          if [ "\$CURRENT_REPLICAS" != "0" ]; then

                            echo "Scaling down EKS deployment"

                            kubectl scale deployment hello-app \
                              --replicas=0 \
                              -n ${params.NAMESPACE}
                          fi

                        else
                          echo "No deployment found in EKS"
                        fi
                        """
                    }
                }
            }
        }
    }

    // ============================================================
    // Deploy Router (Split Traffic)
    // ============================================================

    stage('Deploy Router (Split Traffic)') {

        container('tools') {

            script {

                // ====================================================
                // Deploy Router In GCP
                // ====================================================

                if (params.ACTION == "deploy" &&
                    params.CLOUD_PROVIDER == "gcp") {

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
                        '''

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

                        echo "===== Router Service ====="

                        kubectl get svc nginx-router-service \
                          -n ${params.NAMESPACE}
                        """
                    }
                }

                // ====================================================
                // Delete Router From GCP
                // ====================================================

                if (params.ACTION == "delete" &&
                    params.CLOUD_PROVIDER == "gcp") {

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
                        '''

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
}
