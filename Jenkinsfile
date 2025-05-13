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
                git branch: 'main', url: 'https://github.com/Ahmedrafat9/Blogging-app.git'
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
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Blogging-app \
                        -Dsonar.projectKey=Blogging-app \
                        -Dsonar.java.binaries=target'''
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
                sh "docker build -t ahmedrafat/blogging-app:latest ."
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh "trivy image --format table -o image.html ahmedrafat/blogging-app:latest"
                sh "trivy image --format json -o image.json ahmedrafat/blogging-app:latest"
            }
        }

        stage('Docker Push Image') {
            steps {
                withDockerRegistry(credentialsId: 'dockerhub', url: 'https://index.docker.io/v1/') {
                    sh "docker push ahmedrafat/blogging-app:latest"
                }
            }
        }

        stage('K8s Deploy') {
            steps {
                withKubeCredentials(kubectlCredentials: [[
                    caCertificate: '', 
                    clusterName: 'devopsshack-cluster', 
                    contextName: '', 
                    credentialsId: 'k8s-token', 
                    namespace: 'webapps', 
                    serverUrl: 'https://0FC3C0371E5919CF7870F6EB7CB7AD82.sk1.us-east-1.eks.amazonaws.com'
                ]]) {
                    sh "kubectl apply -f deployment-service.yml"
                    sleep 20
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withKubeCredentials(kubectlCredentials: [[
                    caCertificate: '', 
                    clusterName: 'devopsshack-cluster', 
                    contextName: '', 
                    credentialsId: 'k8s-token', 
                    namespace: 'webapps', 
                    serverUrl: 'https://0FC3C0371E5919CF7870F6EB7CB7AD82.sk1.us-east-1.eks.amazonaws.com'
                ]]) {
                    sh "kubectl get pods"
                    sh "kubectl get service"
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'image.html', allowEmptyArchive: true

            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
                pipelineStatus = pipelineStatus.toUpperCase()
                def bannerColor = pipelineStatus == 'SUCCESS' ? 'green' : 'red'

                def body = """
                <body>
                    <div style="border: 2px solid ${bannerColor}; padding: 10px;">
                        <h3 style="color: ${bannerColor};">
                            Pipeline Status: ${pipelineStatus}
                        </h3>
                        <p>Job: ${jobName}</p>
                        <p>Build Number: ${buildNumber}</p>
                        <p>Status: ${pipelineStatus}</p>
                        <p><a href="${env.BUILD_URL}">View Build</a></p>
                    </div>
                </body>
                """

                emailext(
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus}",
                    body: body,
                    to: 'ahmedrafat456@gmail.com',
                    from: 'jenkins@example.com',
                    replyTo: 'jenkins@example.com',
                    mimeType: 'text/html'
                )
            }
        }
    } // <-- Closing brace for the 'post' block
} // <-- Closing brace for the pipeline block
