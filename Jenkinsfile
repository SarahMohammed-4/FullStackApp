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

        // 🔹 Stage 3: بناء مشروع الفرونت (محلي باستخدام Node المثبت)
        stage('Frontend Build - NodeJS') {
            steps {
                dir('frontend') {
                    sh '''
                        echo "🚀 Starting frontend build (Node.js local)..."
                        npm install
                        npm run build
                        echo "✅ Frontend build completed successfully!"
                    '''
                }
            }
        }

        // 🔹 Stage 4: تحليل الجودة عبر SonarQube للفرونت
        stage('SonarQube Frontend Analysis') {
            steps {
                withSonarQubeEnv('Frontend') {
                    script {
                        def scannerHome = tool 'sonar-scanner'
                        dir('frontend') {
                            sh """
                                echo "🔍 Starting SonarQube analysis for Frontend..."
                                ${scannerHome}/bin/sonar-scanner \
                                  -Dsonar.projectKey=frontend \
                                  -Dsonar.sources=. \
                                  -Dsonar.exclusions=node_modules/**,dist/** \
                                  -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info
                            """
                        }
                        timeout(time: 15, unit: 'MINUTES') {
                            waitForQualityGate abortPipeline: true
                        }
                    }
                }
            }
        }

        // 🔹 Stage 5: رفع الباك والفرونت إلى Nexus بشكل متوازي
        stage('Upload_To_Nexus') {
            parallel {

                stage('Upload_Backend') {
                    steps {
                        nexusArtifactUploader(
                            artifacts: [[
                                artifactId: 'demo',
                                classifier: '',
                                file: 'demo/target/demo-0.0.1-SNAPSHOT.jar',
                                type: 'jar'
                            ]],
                            credentialsId: 'Nexus',
                            groupId: 'com.example',
                            nexusUrl: '3.127.210.51:8081',
                            nexusVersion: 'nexus3',
                            protocol: 'http',
                            repository: 'backend',
                            version: "${BUILD_NUMBER}"
                        )
                    }
                }

                stage('Upload_Frontend') {
                    steps {
                        dir('frontend') {
                            sh "tar -czf frontend-${BUILD_NUMBER}.tgz -C dist ."
                        }
                        nexusArtifactUploader(
                            artifacts: [[
                                artifactId: 'frontend',
                                classifier: '',
                                file: "frontend/frontend-${BUILD_NUMBER}.tgz",
                                type: 'tgz'
                            ]],
                            credentialsId: 'Nexus',
                            groupId: 'com.example.frontend',
                            nexusUrl: '3.127.210.51:8081',
                            nexusVersion: 'nexus3',
                            protocol: 'http',
                            repository: 'frontend',
                            version: "${BUILD_NUMBER}"
                        )
                    }
                }
            }
        }

        // 🔹 Stage 6: بناء ودفع صور Docker
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
                            echo "🐳 Building and pushing Docker images..."

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

        // 🔹 Stage 7: تحديث تاغات الصور في ملفات K8s
        stage('Update image tags in K8s manifests') {
            steps {
                sh """
                    echo "📝 Updating image tags in deployment files..."
                    sed -i "s|sarah1mo/backend-demo:.*|sarah1mo/backend-demo:${BUILD_NUMBER}|g" k8s/backend-deployment.yaml
                    sed -i "s|sarah1mo/frontend-app:.*|sarah1mo/frontend-app:${BUILD_NUMBER}|g" k8s/frontend-deployment.yaml
                """
            }
        }

        // 🔹 Stage 8: النشر على Kubernetes باستخدام Ansible
        stage('Deploy to Kubernetes (Ansible)') {
            steps {
                sh 'ansible-playbook -i ansible/inventory.ini ansible/deploy.yml'
            }
        }
    }

    post {
        success {
            echo '🎉 Pipeline finished SUCCESSFULLY ✅'
        }
        failure {
            echo '💥 Pipeline failed ❌ - check logs'
        }
    }
}
