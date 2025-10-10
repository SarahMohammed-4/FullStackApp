def stageStatus = [:]

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

        // Stage 1: Checkout
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    extensions: [[$class: 'CleanBeforeCheckout']],
                    userRemoteConfigs: [[
                        url: 'git@github.com:SarahMohammed-4/FullStackApp.git',
                        credentialsId: 'github-creds'
                    ]]
                ])
            }
            post {
                success { script { stageStatus['Checkout'] = 'SUCCESS'; sendStageEmail('Checkout', 'SUCCESS') } }
                failure { script { stageStatus['Checkout'] = 'FAILURE'; sendStageEmail('Checkout', 'FAILURE') } }
            }
        }

        // Stage 2: Build & Test (Parallel)
        stage('Build & Test') {
            parallel {

                stage('Backend Build & Test') {
                    steps {
                        dir('demo') {
                            sh '''
                                echo "Building backend..."
                                mvn clean package
                                echo "Running backend tests..."
                                mvn test -Dspring.profiles.active=test-no-db
                            '''
                        }
                    }
                    post {
                        success { script { stageStatus['Backend Build & Test'] = 'SUCCESS'; sendStageEmail('Backend Build & Test', 'SUCCESS') } }
                        failure { script { stageStatus['Backend Build & Test'] = 'FAILURE'; sendStageEmail('Backend Build & Test', 'FAILURE') } }
                    }
                }

                stage('Frontend Build & Test') {
                    steps {
                        dir('frontend') {
                            sh '''
                                echo "Building frontend..."
                                npm install
                                npm run build
                                echo "Running frontend tests..."
                                export CHROME_BIN=$(which google-chrome)
                                npm test -- --watch=false --browsers=ChromeHeadless --code-coverage
                            '''
                        }
                    }
                    post {
                        success { script { stageStatus['Frontend Build & Test'] = 'SUCCESS'; sendStageEmail('Frontend Build & Test', 'SUCCESS') } }
                        failure { script { stageStatus['Frontend Build & Test'] = 'FAILURE'; sendStageEmail('Frontend Build & Test', 'FAILURE') } }
                    }
                }
            }
        }

        // Stage 3: SonarQube Analysis (Parallel)
        stage('SonarQube Analysis') {
            parallel {

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
                    post {
                        success { script { stageStatus['SonarQube Backend'] = 'SUCCESS'; sendStageEmail('SonarQube Backend', 'SUCCESS') } }
                        failure { script { stageStatus['SonarQube Backend'] = 'FAILURE'; sendStageEmail('SonarQube Backend', 'FAILURE') } }
                    }
                }

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
                    post {
                        success { script { stageStatus['SonarQube Frontend'] = 'SUCCESS'; sendStageEmail('SonarQube Frontend', 'SUCCESS') } }
                        failure { script { stageStatus['SonarQube Frontend'] = 'FAILURE'; sendStageEmail('SonarQube Frontend', 'FAILURE') } }
                    }
                }
            }
        }

        // Stage 4: Quality Gate
        stage('Quality Gate Check') {
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
            post {
                success { script { stageStatus['Quality Gate Check'] = 'SUCCESS'; sendStageEmail('Quality Gate Check', 'SUCCESS') } }
                failure { script { stageStatus['Quality Gate Check'] = 'FAILURE'; sendStageEmail('Quality Gate Check', 'FAILURE') } }
            }
        }

        // Stage 5: Upload to Nexus (Parallel)
        stage('Upload_To_Nexus') {
            parallel {

                stage('Upload_Backend') {
                    steps {
                        nexusArtifactUploader(
                            artifacts: [[
                                artifactId: 'demo',
                                file: 'demo/target/demo-0.0.1-SNAPSHOT.jar',
                                type: 'jar'
                            ]],
                            credentialsId: 'Nexus',
                            groupId: 'com.example',
                            nexusUrl: '52.57.155.74:8081',
                            nexusVersion: 'nexus3',
                            protocol: 'http',
                            repository: 'backend',
                            version: "${BUILD_NUMBER}"
                        )
                    }
                    post {
                        success { script { stageStatus['Upload_Backend'] = 'SUCCESS'; sendStageEmail('Upload_Backend', 'SUCCESS') } }
                        failure { script { stageStatus['Upload_Backend'] = 'FAILURE'; sendStageEmail('Upload_Backend', 'FAILURE') } }
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
                                file: "frontend/frontend-${BUILD_NUMBER}.tgz",
                                type: 'tgz'
                            ]],
                            credentialsId: 'Nexus',
                            groupId: 'com.example.frontend',
                            nexusUrl: '52.57.155.74:8081',
                            nexusVersion: 'nexus3',
                            protocol: 'http',
                            repository: 'frontend',
                            version: "${BUILD_NUMBER}"
                        )
                    }
                    post {
                        success { script { stageStatus['Upload_Frontend'] = 'SUCCESS'; sendStageEmail('Upload_Frontend', 'SUCCESS') } }
                        failure { script { stageStatus['Upload_Frontend'] = 'FAILURE'; sendStageEmail('Upload_Frontend', 'FAILURE') } }
                    }
                }
            }
        }

        // Stage 6: Build & Push Docker
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
                            echo "Building and pushing Docker images..."
                            docker build --no-cache -t ${BACKEND_IMAGE}:${BUILD_NUMBER} -f demo/Dockerfile demo
                            docker push ${BACKEND_IMAGE}:${BUILD_NUMBER}
                            docker build --no-cache -t ${FRONTEND_IMAGE}:${BUILD_NUMBER} -f frontend/Dockerfile frontend
                            docker push ${FRONTEND_IMAGE}:${BUILD_NUMBER}
                        """
                    }
                }
            }
            post {
                success { script { stageStatus['Build_And_Push_Docker'] = 'SUCCESS'; sendStageEmail('Build_And_Push_Docker', 'SUCCESS') } }
                failure { script { stageStatus['Build_And_Push_Docker'] = 'FAILURE'; sendStageEmail('Build_And_Push_Docker', 'FAILURE') } }
            }
        }

        // Stage 7: Update image tags
        stage('Update image tags in K8s manifests') {
            steps {
                sh """
                    echo "Updating image tags in deployment files..."
                    sed -i "s|sarah1mo/backend-demo:.*|sarah1mo/backend-demo:${BUILD_NUMBER}|g" k8s/backend-deployment.yaml
                    sed -i "s|sarah1mo/frontend-app:.*|sarah1mo/frontend-app:${BUILD_NUMBER}|g" k8s/frontend-deployment.yaml
                """
            }
            post {
                success { script { stageStatus['Update image tags'] = 'SUCCESS'; sendStageEmail('Update image tags', 'SUCCESS') } }
                failure { script { stageStatus['Update image tags'] = 'FAILURE'; sendStageEmail('Update image tags', 'FAILURE') } }
            }
        }

        // Stage 8: Deploy to Kubernetes
        stage('Deploy to Kubernetes (Ansible)') {
            steps {
                sh 'ansible-playbook -i ansible/inventory.ini ansible/deploy.yml'
            }
            post {
                success { script { stageStatus['Deploy to Kubernetes'] = 'SUCCESS'; sendStageEmail('Deploy to Kubernetes', 'SUCCESS') } }
                failure { script { stageStatus['Deploy to Kubernetes'] = 'FAILURE'; sendStageEmail('Deploy to Kubernetes', 'FAILURE') } }
            }
        }
    }

    post {
        always {
            script {
                def htmlTable = """
                <html>
                  <body>
                    <h2>Pipeline Stage Results</h2>
                    <table border="1" cellpadding="5" cellspacing="0" style="border-collapse: collapse;">
                      <tr><th>Stage</th><th>Status</th></tr>
                """
                stageStatus.each { stage, status ->
                    def color = (status == 'SUCCESS') ? 'green' : 'red'
                    htmlTable += "<tr><td>${stage}</td><td style='color:${color}; font-weight:bold;'>${status}</td></tr>"
                }
                htmlTable += "</table></body></html>"

                writeFile file: 'pipeline_report.html', text: htmlTable
                archiveArtifacts artifacts: 'pipeline_report.html', fingerprint: true

                emailext(
                    subject: "Pipeline Report - ${currentBuild.fullDisplayName}",
                    to: "sarah.mkhj22@gmail.com",
                    body: htmlTable,
                    mimeType: 'text/html'
                )
            }
        }
    }
}

def sendStageEmail(String stageName, String status) {
    def color = (status == 'SUCCESS') ? 'green' : 'red'
    def body = """
    <html>
      <body>
        <h3>Stage: ${stageName}</h3>
        <p>Status: <span style="color:${color}; font-weight:bold;">${status}</span></p>
      </body>
    </html>
    """
    emailext(
        subject: "Stage '${stageName}' Completed - ${status}",
        to: "sarah.mkhj22@gmail.com",
        body: body,
        mimeType: 'text/html'
    )
}
