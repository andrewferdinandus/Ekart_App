pipeline {
    agent any
    
    tools {
        jdk 'jdk' 
        maven 'maven'
    }
    
    environment {
        K8S_CRED_ID = 'k8s-config-file' 
        DEPLOY_NAME = 'ekart-k8-deployment'
        NAMESPACE   = 'project1'
    }
    
    stages {
        stage('Code Checkout') {
            steps {
                git branch: 'agent-test', changelog: false, poll: false, url: 'https://github.com/andrewferdinandus/Ekart_App.git'
            }
        }

        stage('Compile & Test') {
            steps {
                sh "mvn clean verify"
            }
        }
  
        stage('Sonarqube Analysis') {
            tools {
                jdk 'jdk11' 
            }
            steps {
                withSonarQubeEnv(installationName: 'sonar-server', credentialsId: '53be6002-0fe8-438e-8aa4-8eee3e568cda') {
                    sh 'mvn org.sonarsource.scanner.maven:sonar-maven-plugin:4.0.0.4121:sonar \
                        -Dsonar.projectKey=Ekart \
                        -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                        -Dsonar.jacoco.reportPath=target/jacoco.exec'
                }
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests=true'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: '17b9b283-b3a9-434e-a078-1ada27fbcec3') {
                        sh "docker build -t andrewferdi/ekart-app:latest -f docker/Dockerfile ."
                    }
                }
            }
        }
        
        stage('Image Repo Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: '17b9b283-b3a9-434e-a078-1ada27fbcec3') {
                        sh "docker push andrewferdi/ekart-app:latest"
                    }
                }
            }
        }
        
       
        stage('Kubernetes Deployment') {
            steps {
                withKubeConfig([credentialsId: "${K8S_CRED_ID}"]) {
                    sh """
                        echo "Connecting to Cluster..."
                        kubectl apply -f k8s-deployment.yaml -n ${NAMESPACE}
                        
                        echo "Restarting pods to pull latest image..."
                        kubectl rollout restart deployment/${DEPLOY_NAME} -n ${NAMESPACE}
                        
                        echo "Waiting for rollout to complete..."
                        kubectl rollout status deployment/${DEPLOY_NAME} -n ${NAMESPACE} --timeout=60s
                    """
                }
            }
        }
        
        stage('Verification') {
            steps {
                withKubeConfig([credentialsId: "${K8S_CRED_ID}"]) {
                    sh """
                        echo "Final Pod Status:"
                        kubectl get pods -n ${NAMESPACE} -o wide
                    """
                }
            }
        }
    }
}