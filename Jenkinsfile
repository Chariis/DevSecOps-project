pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
        DOCKER_IMAGE="chariis15/netflix" // Your Docker Hub username
        DOCKER_TAG="${DOCKER_IMAGE}:latest"
        TMDB_KEY="<your-tmdb-api-key-here>" // Replace with Jenkins secret/parameter
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: '[https://github.com/Chariis/DevSecOps-project.git](https://github.com/Chariis/DevSecOps-project.git)'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix '''
                }
            }
        }
        stage("Quality Gate Check"){
           steps {
                script {
                    // Uses credential ID 'Sonar-token'
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            } 
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP Dependency Check (FS SCAN)') {
            steps {
                // Uses tool named 'DP-Check'
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY Filesystem SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                    // Uses credential ID 'docker' (chariis15 credentials)
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){ 
                        // Build using the TMDB API Key from environment variable
                        sh "docker build --build-arg TMDB_V3_API_KEY=${TMDB_KEY} -t netflix ."
                        
                        // Tag with your Docker Hub username
                        sh "docker tag netflix ${DOCKER_TAG}" 
                        
                        // Push to your Docker Hub repository
                        sh "docker push ${DOCKER_TAG}" 
                    }
                }
            }
        }
        stage("TRIVY Image Scan"){
            steps{
                // Scan the newly pushed image
                sh "trivy image ${DOCKER_TAG} > trivyimage.txt" 
            }
        }
        stage('Deploy to container'){
            steps{
                // Run the container using the image from your Docker Hub
                sh "docker stop netflix || true" // Stop existing container if it exists
                sh "docker rm netflix || true" // Remove existing container
                sh "docker run -d --name netflix -p 8081:80 ${DOCKER_TAG}"
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
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}<br/>" +
                "Build Number: ${env.BUILD_NUMBER}<br/>" +
                "URL: ${env.BUILD_URL}<br/>",
            to: 'charisnnadi15@gmail.com',
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}