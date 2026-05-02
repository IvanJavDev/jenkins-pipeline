pipeline {
    agent any

    stages {
        stage('Install kubectl') {
            steps {
                sh '''
                    echo "Downloading kubectl..."
                    curl -LO "https://dl.k8s.io/release/v1.29.0/bin/linux/amd64/kubectl"
                    chmod +x kubectl

                    # Копируем в домашнюю директорию Jenkins (она точно writable)
                    mkdir -p $HOME/bin
                    cp kubectl $HOME/bin/kubectl
                    export PATH="$HOME/bin:$PATH"

                    echo "kubectl installed to $HOME/bin"
                    $HOME/bin/kubectl version --client
                '''
            }
        }

        stage('Configure kubectl and Test') {
            steps {
                sh '''
                    # Добавляем в PATH
                    export PATH="$HOME/bin:$PATH"

                    echo "=== Проверка ServiceAccount ==="
                    ls -la /var/run/secrets/kubernetes.io/serviceaccount/

                    echo ""
                    echo "=== Настройка kubeconfig ==="
                    TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
                    CA_CERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
                    SERVER=https://kubernetes.default.svc

                    kubectl config set-cluster docker-desktop \
                      --server=$SERVER \
                      --certificate-authority=$CA_CERT

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