pipeline {
    agent any
    tools {
        jdk "jdk"
        maven "maven"
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    
    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/elyjam/pfsdevops.git'
            }
        }
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        stage('Trivy FS') {
            steps {
                sh "trivy fs . --format table -o fs.html"
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqubeServer') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=pfs_2024 -Dsonar.projectKey=pfs_2024 -Dsonar.java.binaries=target'''
                }
            }
        }
        stage('Build') {
            steps {
                sh "mvn package"
            }
        }
        stage('Publish Artifacts') {
            steps {
                withMaven(globalMavenSettingsConfig: 'maven-settings', jdk: 'jdk', maven: 'maven', mavenSettingsConfig: '', traceability: true) {
                        sh "mvn deploy"
                }
            }
        }
        stage('Docker Build & Tag') {
            steps {
                script{
                withDockerRegistry(credentialsId: 'dockerhub-credentials', url: 'https://index.docker.io/v1/') {
                sh "docker build -t hajarzouhair/pfs_2024 ."
                }
                }
            }
        }
        stage('Trivy Image Scan') {
            steps {
                sh "trivy image --format table -o image.html  hajarzouhair/pfs_2024:latest"
            }
        }
        stage('Docker Push Image') {
            steps {
                script{
                withDockerRegistry(credentialsId: 'dockerhub-credentials', url: 'https://index.docker.io/v1/') {
                    sh "docker push hajarzouhair/pfs_2024"
                }
                }
            }
        }
        stage('K8s Deploy') {
            steps {
               withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: ' devopsshack-cluster', contextName: '', credentialsId: 'k8s-token', namespace: 'webapps', serverUrl: 'https://72241905C35CB6C4F6D55139060613FC.gr7.us-east-1.eks.amazonaws.com']]) {
                    sh "kubectl apply -f deployment-service.yml"
                    sleep 20
                }
            }
        }
        stage('Verify Deployment') {
            steps {
               withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: ' devopsshack-cluster', contextName: '', credentialsId: 'k8s-token', namespace: 'webapps', serverUrl: 'https://72241905C35CB6C4F6D55139060613FC.gr7.us-east-1.eks.amazonaws.com']]) {
                    sh "kubectl get pods"
                    sh "kubectl get service"
                }
            }
        }
        
    }  // Closing stages
}  // Closing pipeline
post {
    always {
        script {
            // Get job name, build number, and pipeline status
            def jobName = env.JOB_NAME
            def buildNumber = env.BUILD_NUMBER
            def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
            pipelineStatus = pipelineStatus.toUpperCase()
            
            // Set the banner color based on the status
            def bannerColor = pipelineStatus == 'SUCCESS' ? 'green' : 'red'

            // HTML body for the email
            def body = """
            <body>
                <div style="border: 2px solid ${bannerColor}; padding: 10px;">
                    <h3 style="color: ${bannerColor};">
                        Pipeline Status: ${pipelineStatus}
                    </h3>
                    <p>Job: ${jobName}</p>
                    <p>Build Number: ${buildNumber}</p>
                    <p>Status: ${pipelineStatus}</p>
                </div>
            </body>
            """

            // Send email notification
            emailext(
                subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus}",
                body: body,
                to: 'hajarzouhair001@gmail.com',
                mimeType: 'text/html'
            )
        }
    }
    
}

