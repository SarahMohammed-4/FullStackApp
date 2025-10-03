pipeline {
    agent any

    environment {
        DOCKERHUB_USER = 'sarah1mo'
        BACKEND_IMAGE = "${DOCKERHUB_USER}/backend-demo"
        FRONTEND_IMAGE = "${DOCKERHUB_USER}/frontend-app"
        KUBECONFIG = "/home/ubuntu/.kube/config"
        BUILD_PLAYBOOK = 'ansible/build.yml'    // ملف Playbook لبناء Docker
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'github-creds',
                    url: 'git@github.com:SarahMohammed-4/fullstack-app.git'
            }
        }

        stage('Backend Build - Maven') {
            steps {
                dir('demo') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Build_And_Push_Docker') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker',
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    script {
                        // 🔹 Docker Login آمن باستخدام stdin
                        sh '''
                            echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
                        '''

                        // 🔹 Backend Docker Image
                        sh "docker build -t ${BACKEND_IMAGE}:${BUILD_NUMBER} -f demo/Dockerfile demo"
                        sh "docker push ${BACKEND_IMAGE}:${BUILD_NUMBER}"

                        // 🔹 Frontend Docker Image
                        sh "docker build --no-cache -t ${FRONTEND_IMAGE}:${BUILD_NUMBER} ./frontend"
                        sh "docker push ${FRONTEND_IMAGE}:${BUILD_NUMBER}"
                    }
                }
            }
            post {
                success {
                    echo "✅ Docker Build & Push SUCCESS - #${BUILD_NUMBER}"
                }
                failure {
                    echo "❌ Docker Build & Push FAILED - #${BUILD_NUMBER}"
                }
            }
        }

        stage('Deploy to Kubernetes (Ansible)') {
            steps {
                sh 'ansible-playbook -i ansible/inventory.ini ansible/deploy.yml'
            }
        }
    }

    post {
        success {
            echo 'Pipeline finished SUCCESSFULLY ✅'
        }
        failure {
            echo 'Pipeline failed ❌ - check logs'
        }
    }
}
