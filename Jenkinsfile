pipeline {
    agent any

    environment {
        IMAGE_TAG = "${BUILD_NUMBER}"
        REGISTRY = "registry:5000"
        IMAGE_NAME = "${REGISTRY}/wallet-app:${IMAGE_TAG}"
    }

    stages {

        stage('Setup kubectl') {
            steps {
                sh '''
                    if [ ! -f ./kubectl ]; then
                        curl -LO "https://dl.k8s.io/release/v1.29.0/bin/linux/amd64/kubectl"
                        chmod +x kubectl
                    fi

                    TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
                    CA_CERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

                    ./kubectl config set-cluster docker-desktop --server=https://kubernetes.default.svc --certificate-authority=$CA_CERT
                    ./kubectl config set-credentials sa-user --token=$TOKEN
                    ./kubectl config set-context docker-desktop --cluster=docker-desktop --user=sa-user
                    ./kubectl config use-context docker-desktop
                '''
            }
        }

        stage('Build with Kaniko') {
            steps {
                sh '''
                    echo "Сборка образа: ${IMAGE_NAME}"

                    ./kubectl delete pod kaniko-build --ignore-not-found 2>/dev/null

                    cat > /tmp/kaniko.yaml <<KANIKO
apiVersion: v1
kind: Pod
metadata:
  name: kaniko-build
spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:latest
    args:
    - "--context=git://github.com/IvanJavDev/WalletApp.git#master"
    - "--destination=${IMAGE_NAME}"
    - "--insecure"
    - "--insecure-pull"
  restartPolicy: Never
KANIKO

                    ./kubectl apply -f /tmp/kaniko.yaml
                    sleep 10

                    ./kubectl logs -f kaniko-build &
                    LOGS=$!

                    for i in $(seq 1 120); do
                        STATUS=$(./kubectl get pod kaniko-build -o jsonpath='{.status.phase}' 2>/dev/null)
                        [ "$STATUS" = "Succeeded" ] && echo "✅ OK" && break
                        [ "$STATUS" = "Failed" ] && echo "❌ FAIL" && kill $LOGS 2>/dev/null && ./kubectl delete pod kaniko-build --ignore-not-found 2>/dev/null && exit 1
                        sleep 5
                    done

                    kill $LOGS 2>/dev/null
                    ./kubectl delete pod kaniko-build --ignore-not-found 2>/dev/null
                '''
            }
        }

        stage('Deploy PostgreSQL') {
            steps {
                sh '''
                    cat <<'EOF' | ./kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources: { requests: { storage: 1Gi } }
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
data:
  POSTGRES_DB: wallet
  POSTGRES_USER: postgres
---
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
stringData:
  POSTGRES_PASSWORD: "1923"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wallet-db
spec:
  replicas: 1
  selector:
    matchLabels: { app: wallet-db }
  template:
    metadata:
      labels: { app: wallet-db }
    spec:
      containers:
      - name: postgres
        image: postgres:latest
        ports: [{ containerPort: 5432 }]
        envFrom:
        - configMapRef: { name: postgres-config }
        - secretRef: { name: postgres-secret }
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: data
        persistentVolumeClaim: { claimName: postgres-pvc }
---
apiVersion: v1
kind: Service
metadata:
  name: wallet-db
spec:
  ports: [{ port: 5432 }]
  selector: { app: wallet-db }
EOF

                    ./kubectl wait --for=condition=ready pod -l app=wallet-db --timeout=120s
                '''
            }
        }

        stage('Deploy WalletApp') {
            steps {
                sh '''
                    cat <<EOF | ./kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wallet-app
spec:
  replicas: 1
  selector:
    matchLabels: { app: wallet-app }
  template:
    metadata:
      labels: { app: wallet-app }
    spec:
      containers:
      - name: wallet-app
        image: ${IMAGE_NAME}
        imagePullPolicy: Always
        ports: [{ containerPort: 8080 }]
        env:
        - name: SPRING_DATASOURCE_URL
          value: jdbc:postgresql://wallet-db:5432/wallet
        - name: SPRING_DATASOURCE_USERNAME
          valueFrom:
            configMapKeyRef: { name: postgres-config, key: POSTGRES_USER }
        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom:
            secretKeyRef: { name: postgres-secret, key: POSTGRES_PASSWORD }
---
apiVersion: v1
kind: Service
metadata:
  name: wallet-app
spec:
  ports: [{ port: 8080 }]
  selector: { app: wallet-app }
EOF

                    ./kubectl rollout status deployment/wallet-app --timeout=180s
                '''
            }
        }
    }

    post {
        success {
            echo "✅ ГОТОВО: kubectl port-forward svc/wallet-app 8080:8080"
        }
        failure {
            echo "❌ ОШИБКА"
        }
    }
}