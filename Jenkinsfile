pipeline {
    agent any

    stages {
        stage('Install kubectl') {
            steps {
                sh '''
                    curl -LO "https://dl.k8s.io/release/v1.29.0/bin/linux/amd64/kubectl"
                    chmod +x kubectl
                    ./kubectl version --client
                '''
            }
        }

        stage('Configure and Test') {
            steps {
                sh '''
                    TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
                    CA_CERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
                    SERVER=https://kubernetes.default.svc

                    ./kubectl config set-cluster docker-desktop \
                      --server=$SERVER \
                      --certificate-authority=$CA_CERT

                    ./kubectl config set-credentials sa-user \
                      --token=$TOKEN

                    ./kubectl config set-context docker-desktop \
                      --cluster=docker-desktop \
                      --user=sa-user

                    ./kubectl config use-context docker-desktop

                    echo "=== Testing connection ==="
                    ./kubectl cluster-info
                    ./kubectl get nodes
                    ./kubectl get pods --all-namespaces
                '''
            }
        }
    }
}