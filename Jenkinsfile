pipeline {
    agent any

    environment {
        DOCKERHUB_USER = 'sarah1mo'
        BACKEND_IMAGE = "${DOCKERHUB_USER}/backend-demo"
        FRONTEND_IMAGE = "${DOCKERHUB_USER}/frontend-app"
        KUBECONFIG = "/home/ubuntu/.kube/config"
        BUILD_PLAYBOOK = 'ansible/build.yml'
    }

    stages {
        // 🔹 Stage 1: سحب الكود من GitHub
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    doGenerateSubmoduleConfigurations: false,
                    extensions: [[$class: 'CleanBeforeCheckout']],
                    userRemoteConfigs: [[
                        url: 'git@github.com:SarahMohammed-4/FullStackApp.git',
                        credentialsId: 'github-creds'
                    ]]
                ])
            }
        }

        // 🔹 Stage 2: بناء المشروع (Backend)
        stage('Backend Build - Maven') {
            steps {
                dir('demo') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        // 🔹 Stage 3: بناء ودفع صور Docker
        stage('Build_And_Push_Docker') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker',
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    script {
                        sh """
                            echo "\$DOCKER_PASSWORD" | docker login -u "\$DOCKER_USERNAME" --password-stdin
                            echo " Building and pushing Docker images..."

                            # Build & Push Backend
                            docker build --no-cache -t ${BACKEND_IMAGE}:${BUILD_NUMBER} -f demo/Dockerfile demo
                            docker push ${BACKEND_IMAGE}:${BUILD_NUMBER}

                            # Build & Push Frontend
                            docker build --no-cache -t ${FRONTEND_IMAGE}:${BUILD_NUMBER} -f frontend/Dockerfile frontend
                            docker push ${FRONTEND_IMAGE}:${BUILD_NUMBER}
                        """
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

        // 🔹 Stage 4: تحديث تاغات الصور في ملفات الـ K8s
        stage('Update image tags in K8s manifests') {
            steps {
                sh """
                    echo " Updating image tags in deployment files..."
                    sed -i "s|sarah1mo/backend-demo:.*|sarah1mo/backend-demo:${BUILD_NUMBER}|g" k8s/backend-deployment.yaml
                    sed -i "s|sarah1mo/frontend-app:.*|sarah1mo/frontend-app:${BUILD_NUMBER}|g" k8s/frontend-deployment.yaml
                """
            }
        }

        // 🔹 Stage 5: النشر على Kubernetes باستخدام Ansible
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
