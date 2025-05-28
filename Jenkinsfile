pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'Node16'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        NVD_API_KEY = credentials('nvd-api-key')
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/theewizardone/Neflix-Clone--DevSecOps.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh "$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix -Dsonar.projectKey=Netflix"
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('OWASP FS SCAN') {
            steps {
                withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                    dependencyCheck(
                        additionalArguments: '''--scan ./ --disableYarnAudit --disableNodeAudit --nvdApiKey=$NVD_API_KEY''',
                        odcInstallation: 'DP-Check'
                    )
                }
            }
        }

        stage('Trivy FS Scan') {
            steps {
                script {
                    try {
                        sh 'trivy fs . > trivyfs.txt || echo "Trivy FS scan failed or vulnerabilities found."'
                    } catch (err) {
                        echo "Trivy FS scan failed: ${err.getMessage()}"
                    }
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'TMDB_V3_API_KEY', variable: 'TMDB_API_KEY')]) {
                        withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                            try {
                                sh '''
                                    docker build --build-arg TMDB_V3_API_KEY=$TMDB_API_KEY -t netflix .
                                    docker tag netflix theewizardorne/netflix:latest
                                    docker push theewizardorne/netflix:latest
                                '''
                            } catch (err) {
                                echo "Docker build or push failed: ${err.getMessage()}"
                            }
                        }
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                script {
                    try {
                        sh 'trivy image theewizardorne/netflix:latest > trivyimage.txt || echo "Trivy image scan failed or vulnerabilities found."'
                    } catch (err) {
                        echo "Trivy image scan failed: ${err.getMessage()}"
                    }
                }
            }
        }

        stage('Prepare Python Environment') {
            steps {
                sh '''
                    sudo apt-get update
                    sudo apt-get install -y python3-venv
                '''
            }
        }

        stage('Whispers Secret Scan') {
            steps {
                script {
                    try {
                        sh '''
                            python3 --version
                            python3 -m venv venv
                            . venv/bin/activate
                            pip install --quiet whispers
                            whispers --output-format json . > whispers-secrets.json || echo "Whispers scan found secrets or failed"
                        '''
                    } catch (err) {
                        echo "Whispers scan failed with error: ${err}"
                    }
                }
            }
        }

        stage('Deploy to Container') {
            steps {
                script {
                    try {
                        sh 'docker run -d -p 8093:80 theewizardorne/netflix:latest'
                    } catch (err) {
                        echo "Deployment failed: ${err.getMessage()}"
                    }
                }
            }
        }

        stage('Deploy to kubernets') {
            steps {
                script {
                    dir('Kubernetes') {
                        withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                            sh 'kubectl apply -f deployment.yml'
                            sh 'kubectl apply -f service.yml'
                        }
                    }
                }
            }
        }
    } 
    
    post {
        always {
            archiveArtifacts artifacts: '**/dependency-check-report.xml', allowEmptyArchive: true
            archiveArtifacts artifacts: 'whispers-secrets.json', allowEmptyArchive: true
            archiveArtifacts artifacts: 'trivyfs.txt', allowEmptyArchive: true
            archiveArtifacts artifacts: 'trivyimage.txt', allowEmptyArchive: true
            emailext attachLog: true,
                subject: "'${currentBuild.result}'",
                body: "Project: ${env.JOB_NAME}<br/>" +
                      "Build Number: ${env.BUILD_NUMBER}<br/>" +
                      "URL: ${env.BUILD_URL}<br/>",
                to: 'alfoncemorara412@gmail.com',
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
            echo 'Pipeline completed.'
        }
    }
}

    
