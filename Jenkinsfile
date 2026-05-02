pipeline {
    agent any

    environment {
        IMAGE_TAG = "${BUILD_NUMBER}"
        IMAGE_NAME = "wallet-app:${IMAGE_TAG}"
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

        stage('Docker Build') {
            steps {
                sh '''
                    echo "=== Сборка Docker образа (Maven внутри) ==="
                    docker build -t ${IMAGE_NAME} .
                    echo "Образ собран: ${IMAGE_NAME}"
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
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
data:
  POSTGRES_DB: "wallet"
  POSTGRES_USER: "postgres"
---
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
type: Opaque
stringData:
  POSTGRES_PASSWORD: "1923"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wallet-db
  labels:
    app: wallet-db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wallet-db
  template:
    metadata:
      labels:
        app: wallet-db
    spec:
      containers:
      - name: postgres
        image: postgres:latest
        ports:
        - containerPort: 5432
        envFrom:
        - configMapRef:
            name: postgres-config
        - secretRef:
            name: postgres-secret
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: wallet-db
spec:
  type: ClusterIP
  ports:
  - port: 5432
    targetPort: 5432
  selector:
    app: wallet-db
EOF

                    echo "Ожидание PostgreSQL..."
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
    matchLabels:
      app: wallet-app
  template:
    metadata:
      labels:
        app: wallet-app
    spec:
      containers:
      - name: wallet-app
        image: ${IMAGE_NAME}
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_DATASOURCE_URL
          value: "jdbc:postgresql://wallet-db:5432/wallet"
        - name: SPRING_DATASOURCE_USERNAME
          valueFrom:
            configMapKeyRef:
              name: postgres-config
              key: POSTGRES_USER
        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: POSTGRES_PASSWORD
        - name: SPRING_LIQUIBASE_ENABLED
          value: "true"
---
apiVersion: v1
kind: Service
metadata:
  name: wallet-app
spec:
  type: ClusterIP
  ports:
  - port: 8080
    targetPort: 8080
  selector:
    app: wallet-app
EOF

                    echo "Ожидание WalletApp..."
                    ./kubectl rollout status deployment/wallet-app --timeout=180s
                '''
            }
        }

        stage('Проверка') {
            steps {
                sh '''
                    echo "=== Под ==="
                    ./kubectl get pods
                    echo ""
                    echo "=== Логи WalletApp ==="
                    ./kubectl logs -l app=wallet-app --tail=30
                '''
            }
        }
    }

    post {
        success {
            echo """
            ✅ ДЕПЛОЙ УСПЕШЕН!
            kubectl port-forward svc/wallet-app 8080:8080
            http://localhost:8080
            """
        }
        failure {
            echo "❌ ОШИБКА!"
        }
    }
}