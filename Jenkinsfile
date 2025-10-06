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

        // 🟦 Stage 1: Checkout
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

        // 🟦 Stage 2: Backend Build
        stage('Backend Build - Maven') {
            steps {
                dir('demo') {
                    sh 'mvn clean package -DskipTests'
                }
            }
            post {
                success { script { stageStatus['Backend Build - Maven'] = 'SUCCESS'; sendStageEmail('Backend Build - Maven', 'SUCCESS') } }
                failure { script { stageStatus['Backend Build - Maven'] = 'FAILURE'; sendStageEmail('Backend Build - Maven', 'FAILURE') } }
            }
        }

        // 🧪 Stage 3: Backend Test
        stage('Backend Test - Maven') {
            steps {
                dir('demo') {
                    sh '''
                        echo "🧪 Running backend tests with test-no-db profile..."
                        mvn test -Dspring.profiles.active=test-no-db
                    '''
                }
            }
            post {
                success { script { stageStatus['Backend Test - Maven'] = 'SUCCESS'; sendStageEmail('Backend Test - Maven', 'SUCCESS') } }
                failure { script { stageStatus['Backend Test - Maven'] = 'FAILURE'; sendStageEmail('Backend Test - Maven', 'FAILURE') } }
            }
        }

        // 🟦 Stage 4: SonarQube Backend Analysis
        stage('SonarQube Backend Analysis') {
            steps {
                withSonarQubeEnv('Backend') {
                    dir('demo') {
                        sh '''
                            echo "🔍 Starting SonarQube analysis for Backend..."
                            mvn clean verify sonar:sonar -DskipTests \
                              -Dsonar.projectKey=backend
                        '''
                    }
                }
            }
            post {
                success { script { stageStatus['SonarQube Backend Analysis'] = 'SUCCESS'; sendStageEmail('SonarQube Backend Analysis', 'SUCCESS') } }
                failure { script { stageStatus['SonarQube Backend Analysis'] = 'FAILURE'; sendStageEmail('SonarQube Backend Analysis', 'FAILURE') } }
            }
        }

        // 🟦 Stage 5: Frontend Build
        stage('Frontend Build - NodeJS') {
            steps {
                dir('frontend') {
                    sh '''
                        echo "🚀 Starting frontend build..."
                        npm install
                        npm run build
                        echo "✅ Frontend build completed successfully!"
                    '''
                }
            }
            post {
                success { script { stageStatus['Frontend Build - NodeJS'] = 'SUCCESS'; sendStageEmail('Frontend Build - NodeJS', 'SUCCESS') } }
                failure { script { stageStatus['Frontend Build - NodeJS'] = 'FAILURE'; sendStageEmail('Frontend Build - NodeJS', 'FAILURE') } }
            }
        }

        // 🧪 Stage 6: Frontend Test (Chrome Fixed)
        stage('Frontend Test - Angular') {
            steps {
                dir('frontend') {
                    sh '''
                        echo "🧪 Running frontend tests with coverage..."
                        npm install
                        export CHROME_BIN=$(which google-chrome)
                        npm test -- --watch=false --browsers=ChromeHeadless --code-coverage
                    '''
                }
            }
            post {
                success { script { stageStatus['Frontend Test - Angular'] = 'SUCCESS'; sendStageEmail('Frontend Test - Angular', 'SUCCESS') } }
                failure { script { stageStatus['Frontend Test - Angular'] = 'FAILURE'; sendStageEmail('Frontend Test - Angular', 'FAILURE') } }
            }
        }

        // 🟦 Stage 7: SonarQube Frontend Analysis
        stage('SonarQube Frontend Analysis') {
            steps {
                withSonarQubeEnv('Frontend') {
                    dir('frontend') {
                        script {
                            def scannerHome = tool 'Scanner'
                            sh """
                                echo "🔍 Starting SonarQube analysis for Frontend..."
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
                success { script { stageStatus['SonarQube Frontend Analysis'] = 'SUCCESS'; sendStageEmail('SonarQube Frontend Analysis', 'SUCCESS') } }
                failure { script { stageStatus['SonarQube Frontend Analysis'] = 'FAILURE'; sendStageEmail('SonarQube Frontend Analysis', 'FAILURE') } }
            }
        }

        // 🟦 Stage 8: Quality Gate
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

        // باقي الستيجز نفسها بدون تغيير ...
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
