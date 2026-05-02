pipeline {
    agent any

    environment {
        K8S_NAMESPACE = 'default'
        IMAGE_TAG = "${BUILD_NUMBER}"
        IMAGE_NAME = "wallet-app:${IMAGE_TAG}"
    }

    stages {

        // =============================================
        // Этап 1: Настройка kubectl
        // =============================================
        stage('Setup kubectl') {
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

        // =============================================
        // Этап 2: Сборка Maven
        // =============================================
        stage('Maven Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        // =============================================
        // Этап 3: Тесты
        // =============================================
        stage('Run Tests') {
            steps {
                sh 'mvn test'
            }
        }

        // =============================================
        // Этап 4: Сборка Docker образа
        // =============================================
        stage('Docker Build') {
            steps {
                sh '''
                    # Используем твой существующий Dockerfile
                    docker build -t ${IMAGE_NAME} .
                    echo "Image built: ${IMAGE_NAME}"
                    docker images | grep wallet-app
                '''
            }
        }

        // =============================================
        // Этап 5: Деплой всего стека в Kubernetes
        // =============================================
        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    echo "=== Deploying PostgreSQL ==="

                    # --- PostgreSQL Deployment ---
                    cat <<'EOF' | ./kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: ${K8S_NAMESPACE}
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
  namespace: ${K8S_NAMESPACE}
data:
  POSTGRES_DB: "wallet"
  POSTGRES_USER: "postgres"
---
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: ${K8S_NAMESPACE}
type: Opaque
stringData:
  POSTGRES_PASSWORD: "1923"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wallet-db
  namespace: ${K8S_NAMESPACE}
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
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
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
  namespace: ${K8S_NAMESPACE}
  labels:
    app: wallet-db
spec:
  type: ClusterIP
  ports:
  - port: 5432
    targetPort: 5432
  selector:
    app: wallet-db
EOF

                    echo ""
                    echo "=== Waiting for PostgreSQL ==="
                    ./kubectl wait --for=condition=ready pod \
                      -l app=wallet-db \
                      -n ${K8S_NAMESPACE} \
                      --timeout=120s

                    echo ""
                    echo "=== Deploying WalletApp ==="

                    # --- WalletApp Deployment ---
                    cat <<EOF | ./kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wallet-app
  namespace: ${K8S_NAMESPACE}
  labels:
    app: wallet-app
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
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 20
---
apiVersion: v1
kind: Service
metadata:
  name: wallet-app
  namespace: ${K8S_NAMESPACE}
  labels:
    app: wallet-app
spec:
  type: ClusterIP
  ports:
  - port: 8080
    targetPort: 8080
  selector:
    app: wallet-app
EOF

                    echo ""
                    echo "=== Waiting for WalletApp ==="
                    ./kubectl rollout status deployment/wallet-app \
                      -n ${K8S_NAMESPACE} \
                      --timeout=180s
                '''
            }
        }

        // =============================================
        // Этап 6: Проверка
        // =============================================
        stage('Verify') {
            steps {
                sh '''
                    echo "=== All Pods ==="
                    ./kubectl get pods -n ${K8S_NAMESPACE}

                    echo ""
                    echo "=== Services ==="
                    ./kubectl get svc -n ${K8S_NAMESPACE}

                    echo ""
                    echo "=== WalletApp Logs ==="
                    ./kubectl logs -l app=wallet-app -n ${K8S_NAMESPACE} --tail=30
                '''
            }
        }
    }

    post {
        success {
            echo '''
            ================================================
             ДЕПЛОЙ УСПЕШНО ЗАВЕРШЕН!
            ================================================

             Компоненты в namespace default:
               - wallet-db (PostgreSQL)
               - wallet-app (Spring Boot)

             Для доступа к приложению выполни на хосте:
               kubectl port-forward svc/wallet-app 8080:8080

             Затем открой: http://localhost:8080
            ================================================
            '''
        }
        failure {
            echo 'ОШИБКА! Смотри логи выше.'
        }
    }
}