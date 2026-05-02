pipeline {
    agent any

     environment {
             IMAGE_TAG = "${BUILD_NUMBER}"
             REGISTRY = "registry:5000"
             IMAGE_NAME = "${REGISTRY}/wallet-app:${IMAGE_TAG}"
         }

    stages {

        stage('Checkout WalletApp') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/IvanJavDev/WalletApp.git'
            }
        }

                stage('Kaniko Build') {
                    steps {
                        sh '''
                            cat > /tmp/kaniko.yaml <<EOF
        apiVersion: v1
        kind: Pod
        metadata:
          name: kaniko-build
        spec:
          hostAliases:
          - ip: "10.96.30.113"
            hostnames:
            - "registry"
          containers:
          - name: kaniko
            image: gcr.io/kaniko-project/executor:latest
            args:
            - "--context=git://github.com/IvanJavDev/WalletApp.git#master"
            - "--destination=registry:5000/wallet-app:${IMAGE_TAG}"
            - "--insecure"
            - "--insecure-pull"
          restartPolicy: Never
        EOF
                            /tmp/kubectl delete pod kaniko-build --ignore-not-found 2>/dev/null
                            /tmp/kubectl apply -f /tmp/kaniko.yaml
                            sleep 10
                            /tmp/kubectl logs -f kaniko-build &
                            while true; do
                                STATUS=$(/tmp/kubectl get pod kaniko-build -o jsonpath='{.status.phase}' 2>/dev/null)
                                [ "$STATUS" = "Succeeded" ] && echo "BUILD OK" && break
                                [ "$STATUS" = "Failed" ] && echo "BUILD FAIL" && break
                                sleep 5
                            done
                        '''
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
                    /tmp/kubectl apply -f /tmp/app.yaml
                    /tmp/kubectl rollout status deployment/wallet-app --timeout=180s
                '''
            }
        }
    }
}