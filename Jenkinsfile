pipeline {
    agent any

    stages {
        stage('Install kubectl') {
            steps {
                sh '''
                    echo "Downloading kubectl..."
                    curl -LO "https://dl.k8s.io/release/v1.29.0/bin/linux/amd64/kubectl"
                    chmod +x kubectl
                    mv kubectl /usr/local/bin/kubectl
                    echo "kubectl installed"
                '''
            }
        }

        stage('Check ServiceAccount Access') {
            steps {
                sh '''
                    echo "=== Проверка ServiceAccount ==="
                    ls -la /var/run/secrets/kubernetes.io/serviceaccount/

                    echo ""
                    echo "=== Пробуем curl к API ==="
                    TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
                    CA_CERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
                    SERVER=https://kubernetes.default.svc

                    curl -s --cacert $CA_CERT -H "Authorization: Bearer $TOKEN" \
                      $SERVER/api/v1/namespaces/default/pods

                    echo ""
                    echo "=== Пробуем kubectl с ServiceAccount ==="
                    kubectl config set-cluster docker-desktop \
                      --server=https://kubernetes.default.svc \
                      --certificate-authority=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

                    kubectl config set-credentials sa-user \
                      --token=$TOKEN

                    kubectl config set-context docker-desktop \
                      --cluster=docker-desktop \
                      --user=sa-user

                    kubectl config use-context docker-desktop

                    echo ""
                    echo "=== Проверка подключения ==="
                    kubectl cluster-info
                    kubectl get nodes
                    kubectl get pods --all-namespaces
                '''
            }
        }
    }
}