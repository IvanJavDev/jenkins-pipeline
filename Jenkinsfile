pipeline {
    agent any

    stages {
        stage('Install kubectl') {
            steps {
                sh '''
                    if [ ! -f ./kubectl ]; then
                        curl -LO "https://dl.k8s.io/release/v1.29.0/bin/linux/amd64/kubectl"
                        chmod +x kubectl
                    fi

                    TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
                    CA_CERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

                    ./kubectl config set-cluster docker-desktop \
                      --server=https://kubernetes.default.svc \
                      --certificate-authority=$CA_CERT

                    ./kubectl config set-credentials sa-user --token=$TOKEN

                    ./kubectl config set-context docker-desktop \
                      --cluster=docker-desktop --user=sa-user

                    ./kubectl config use-context docker-desktop
                '''
            }
        }

        stage('Grant Admin Rights') {
            steps {
                sh '''
                    echo "=== Создаём ClusterRoleBinding ==="

                    cat <<EOF | ./kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: jenkins
  namespace: jenkins
EOF

                    echo ""
                    echo "=== Проверяем права ==="
                    ./kubectl auth can-i list pods --all-namespaces
                    ./kubectl auth can-i list nodes
                '''
            }
        }

        stage('Test Full Access') {
            steps {
                sh '''
                    echo "=== Проверка кластера ==="
                    ./kubectl cluster-info
                    ./kubectl get nodes
                    ./kubectl get pods --all-namespaces
                '''
            }
        }
    }
}