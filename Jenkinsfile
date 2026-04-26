pipeline {
    agent any

    stages {
        stage('Hello') {
            steps {
                echo 'Hello from Jenkins!'
                echo 'Это мой первый пайплайн'
            }
        }

        stage('Date') {
            steps {
                // Показываем текущую дату и время
                sh 'date'
            }
        }

        stage('List Files') {
            steps {
                // Показываем содержимое текущей директории
                sh 'ls -la'
                sh 'pwd'
            }
        }

        stage('Environment') {
            steps {
                // Показываем переменные окружения
                sh 'printenv | sort'
            }
        }
    }

    post {
        success {
            echo '✅ ПАЙПЛАЙН УСПЕШНО ЗАВЕРШЁН!'
        }
        failure {
            echo '❌ ПАЙПЛАЙН ЗАВЕРШИЛСЯ С ОШИБКОЙ'
        }
    }
}