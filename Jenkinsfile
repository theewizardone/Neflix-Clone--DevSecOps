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
            dependencyCheck additionalArguments: "--scan ./ --disableYarnAudit --disableNodeAudit --nvdApiKey=$NVD_API_KEY", odcInstallation: 'DP-Check'
            dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
        }
    }
}
        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }

        stage('Docker Build & Push') {
    steps {
        script {
            withCredentials([string(credentialsId: 'TMDB_V3_API_KEY', variable: 'TMDB_API_KEY')]) {
                withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                    sh '''
                        docker build --build-arg TMDB_V3_API_KEY=$TMDB_API_KEY -t netflix .
                        docker tag netflix theewizardorne/netflix:latest
                        docker push theewizardorne/netflix:latest
                    '''
                }
            }
        }
    }
}
        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image theewizardorne/netflix:latest > trivyimage.txt'
            }
        }

        stage('Deploy to Container') {
            steps {
                sh 'docker run -d -p 8086:80 theewizardorne/netflix:latest'
            }
        }
    }
        stage('Deploy to kubernets'){
            steps{
                script{
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
      dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}<br/>" +
                "Build Number: ${env.BUILD_NUMBER}<br/>" +
                "URL: ${env.BUILD_URL}<br/>",
            to: 'alfoncemorara412@gmail.com',                                //change mail here
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
           echo 'Pipeline completed.'
        }
    }
}
