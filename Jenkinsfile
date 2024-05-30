pipeline {
    agent any
    
    tools {
        jdk 'jdk-17'
        maven 'maven3'
    }
    
    environment {
        SCANNER_HOME= tool 'sonar-scanner'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/devopsenthusiastH/Mission-Webapp.git'
                echo 'Git Connected'
            }
        }
        
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('Test') {
            steps {
                sh "mvn test -DskipTests=true"
            }
        }
        
        stage('Trivy Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }
        
        stage('Sonar Qube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=Mission -Dsonar.projectName=Mission \
                            -Dsonar.java.binaries=. '''
                }
            }
        }
        
        stage('Build') {
            steps {
                sh "mvn package -DskipTests=true"
            }
        }
        
        stage('Deploy Artifects to nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'maven-settings', jdk: 'jdk-17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh "mvn deploy -DskipTests=true"
                }
            }
        }
        
        stage('Docker build image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker build -t aakashhandibar/mission:v1 ."
                        
                    }
                }
            }
        }
        
        stage('Trivy Scan image') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html aakashhandibar/mission:v1"
            }
        }
        
        stage('Docker push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker push aakashhandibar/mission:v1"
                        
                    }
                }
            }
        }
        stage('Deploy to K8s') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'DS-EKS', contextName: '', credentialsId: 'kube-secret', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://9879829F7F8EAF7683EDFDC04CE547C7.gr7.ap-south-1.eks.amazonaws.com') {
                    sh "kubectl apply -f ds.yml -n webapps"
                }
            }
        }
        stage('Verify to Deploy') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'DS-EKS', contextName: '', credentialsId: 'kube-secret', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://9879829F7F8EAF7683EDFDC04CE547C7.gr7.ap-south-1.eks.amazonaws.com') {
                    sh "kubectl get pods -n webapps"
                    sh "kubectl get svc -n webapps"
                }
            }
        }
    }
    
    
    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'
                
                def body = """
                    <html>
                    <body>
                    <div style="border: 4px solid ${bannerColor}; padding:10px;">
                    <h2>${jobName} - Build ${buildNumber}</h2>
                    <div style="background-color: ${bannerColor}; padding:10px;">
                    <h3 style="color: white;">Pipeline Status:${pipelineStatus.toUpperCase()}</h3>
                    </div>
                    <p>Check the <a href="${BUILD_URL}">consoleoutput</a>.</p>
                    </div>
                    </body>
                    </html>
                """
                
                emailext (
                    subject: "${jobName} - Build ${buildNumber} -${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'jaiswaladi246@gmail.com',
                    from: 'jenkins@example.com',
                    replyTo: 'jenkins@example.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-image-report.html'
                )
            }
        }
    }
}

