def stageStatus = [:]

pipeline {
    agent any

    environment {
        DOCKERHUB_USER = 'sarah1mo'
        BACKEND_IMAGE = "${DOCKERHUB_USER}/backend-demo"
        FRONTEND_IMAGE = "${DOCKERHUB_USER}/frontend-app"
        KUBECONFIG = "/home/ubuntu/.kube/config"
    }

    stages {

        // 🟩 1️⃣ Checkout SCM (اللي Jenkins يسويه تلقائي)
        stage('Checkout SCM') {
            steps {
                echo 'Checking out source code from SCM...'
                checkout scm
            }
        }

        // 🟩 2️⃣ Checkout (حقك اليدوي)
        stage('Checkout') {
            steps {
                cleanWs()
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[
                        credentialsId: 'git-credentials',
                        url: 'https://github.com/sarah1mo/FullStackApp.git'
                    ]]
                ])
            }
        }

        // 🟦 3️⃣ Build & Test (Frontend + Backend Parallel)
        stage('Build & Test') {
            parallel {
                stage('Build & Test Frontend') {
                    steps {
                        dir('frontend') {
                            sh '''
                                echo "Running frontend build and test..."
                                npm install
                                npm run build
                                export CHROME_BIN=$(which google-chrome)
                                npm test -- --watch=false --browsers=ChromeHeadless --code-coverage
                            '''
                        }
                    }
                }
                stage('Build & Test Backend') {
                    steps {
                        dir('demo') {
                            sh '''
                                echo "Running backend build and test..."
                                mvn clean package -DskipTests=false
                            '''
                        }
                    }
                }
            }
        }

        // 🟨 4️⃣ SonarQube Analysis (Frontend + Backend Parallel)
        stage('SonarQube Analysis') {
            parallel {
                stage('SonarQube Frontend') {
                    steps {
                        withSonarQubeEnv('Frontend') {
                            dir('frontend') {
                                script {
                                    def scannerHome = tool 'Scanner'
                                    sh """
                                        echo "Running SonarQube for Frontend..."
                                        ${scannerHome}/bin/sonar-scanner \
                                          -Dsonar.projectKey=frontend \
                                          -Dsonar.sources=. \
                                          -Dsonar.exclusions=node_modules/**,dist/** \
                                          -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info
                                    """
                                }
                            }
                        }
                    }
                }
                stage('SonarQube Backend') {
                    steps {
                        withSonarQubeEnv('Backend') {
                            dir('demo') {
                                sh '''
                                    echo "Running SonarQube for Backend..."
                                    mvn clean verify sonar:sonar -DskipTests \
                                      -Dsonar.projectKey=backend
                                '''
                            }
                        }
                    }
                }
            }
        }

        // 🟧 5️⃣ Upload to Nexus (Frontend + Backend Parallel)
        stage('Upload_To_Nexus') {
            parallel {
                stage('Upload_Backend') {
                    steps {
                        dir('demo') {
                            sh '''
                                echo "Uploading backend artifact to Nexus..."
                                mvn deploy -DskipTests
                            '''
                        }
                    }
                }
                stage('Upload_Frontend') {
                    steps {
                        dir('frontend') {
                            sh '''
                                echo "Uploading frontend build to Nexus..."
                                tar -czf frontend.tar.gz dist/
                                curl -v -u admin:admin123 --upload-file frontend.tar.gz \
                                  http://<your-nexus-ip>:8081/repository/npm-hosted/frontend.tar.gz
                            '''
                        }
                    }
                }
            }
        }

        // 🟥 6️⃣ Build and Push Docker Images
        stage('Build_And_Push_Docker') {
            steps {
                sh '''
                    echo "Building and pushing Docker images..."
                    docker build -t ${BACKEND_IMAGE}:latest ./demo
                    docker push ${BACKEND_IMAGE}:latest
                    docker build -t ${FRONTEND_IMAGE}:latest ./frontend
                    docker push ${FRONTEND_IMAGE}:latest
                '''
            }
        }

        // 🟪 7️⃣ Deploy to Kubernetes
        stage('Deploy_Kubernetes') {
            steps {
                sh '''
                    echo "Deploying to Kubernetes..."
                    kubectl apply -f k8s/
                '''
            }
        }

        // 🟫 8️⃣ Send Email Report
        stage('Send Email Report') {
            steps {
                script {
                    emailext(
                        subject: "Jenkins Pipeline Report - ${currentBuild.currentResult}",
                        body: """
                            <h2>Pipeline Execution Summary</h2>
                            <p>Status: ${currentBuild.currentResult}</p>
                            <p>Build URL: ${env.BUILD_URL}</p>
                        """,
                        recipientProviders: [[$class: 'DevelopersRecipientProvider']]
                    )
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
