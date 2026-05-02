pipeline {
    agent any

    environment {
        IMAGE_TAG = "${BUILD_NUMBER}"
        IMAGE_NAME = "wallet-app:${IMAGE_TAG}"
        DOCKER_HOST = "tcp://host.docker.internal:2375"
    }

    stages {

        stage('Setup Tools') {
            steps {
                sh '''
                    # kubectl
                    curl -LO "https://dl.k8s.io/release/v1.29.0/bin/linux/amd64/kubectl"
                    chmod +x kubectl
                    cp kubectl /tmp/kubectl

                    # Docker CLI
                    curl -fsSL https://download.docker.com/linux/static/stable/x86_64/docker-27.3.1.tgz -o /tmp/d.tgz
                    cd /tmp && tar xzf d.tgz
                    chmod +x docker/docker
                    cp docker/docker /tmp/docker
                '''
            }
        }

        stage('Checkout WalletApp') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/IvanJavDev/WalletApp.git'
            }
        }

        stage('Maven Build') {
            steps {
                sh '''
                    tar czf /tmp/project.tar.gz .
                    cat /tmp/project.tar.gz | /tmp/docker -H ${DOCKER_HOST} run --rm -i \
                      -w /app \
                      maven:3.9.9-eclipse-temurin-17-alpine \
                      sh -c "cd /app && tar xzf - && mvn clean package -DskipTests"
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '/tmp/docker -H ${DOCKER_HOST} build -t ${IMAGE_NAME} .'
            }
        }

        stage('Deploy WalletApp') {
            steps {
                sh '''
                    cat > /tmp/app.yaml <<EOF
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
        imagePullPolicy: Never
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
                    /tmp/kubectl apply -f /tmp/app.yaml
                    /tmp/kubectl rollout status deployment/wallet-app --timeout=180s
                '''
            }
        }
    }
}