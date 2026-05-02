pipeline {
    agent any

    environment {
        IMAGE_TAG = "${BUILD_NUMBER}"
        IMAGE_NAME = "wallet-app:${IMAGE_TAG}"
    }

    stages {

        stage('Setup Docker') {
            steps {
                sh '''
                    echo "Скачиваем Docker CLI..."
                    curl -fsSL https://download.docker.com/linux/static/stable/x86_64/docker-27.3.1.tgz -o /tmp/d.tgz
                    cd /tmp && tar xzf d.tgz
                    chmod +x docker/docker
                    /tmp/docker/docker version
                '''
            }
        }

        stage('Maven Build') {
            steps {
                sh '''
                    /tmp/docker/docker run --rm \
                      -v "$(pwd)":/app -w /app \
                      maven:3.9.9-eclipse-temurin-17-alpine \
                      mvn clean package -DskipTests
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '/tmp/docker/docker build -t ${IMAGE_NAME} .'
            }
        }

        stage('Deploy PostgreSQL') {
            steps {
                sh '''
                    kubectl apply -f - <<'EOF'
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
                    kubectl wait --for=condition=ready pod -l app=wallet-db --timeout=120s
                '''
            }
        }

        stage('Deploy WalletApp') {
            steps {
                sh '''
                    cat <<EOF | kubectl apply -f -
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
        imagePullPolicy: IfNotPresent
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
                    kubectl rollout status deployment/wallet-app --timeout=180s
                '''
            }
        }
    }

    post {
        success {
            echo "ГОТОВО: kubectl port-forward svc/wallet-app 8080:8080"
        }
        failure {
            echo "ОШИБКА"
        }
    }
}