// Jenkinsfile for user-service
// This file should be placed at the root of the user-service repository.

// Define global environment variables
def dockerRegistry = "sumitrajneesh" // e.g., "myusername"
def dockerCredentialsId = "3bdc9f350d0642d19dec3a60aa1875b4" // Jenkins credential ID for Docker Hub/GitLab Registry
def sonarqubeServerId = "sonarqube-server" // Jenkins SonarQube server configuration ID
def kubernetesCredentialsId = "kubernetes-credentials" // Jenkins credential ID for Kubernetes access (e.g., Kubeconfig)
def kubernetesContext = "minikube" 
def helmChartPath = "helm/user-service-chart"

pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
        skipDefaultCheckout(false)
    }

    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Build and Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        echo "Running Maven unit tests for Spring Boot user-service..."
                        sh "mvn clean test"
                    }
                    post {
                        always {
                            junit '**/target/surefire-reports/*.xml'
                        }
                    }
                }

                stage('Code Quality (SonarQube)') {
                    steps {
                        echo "Running SonarQube analysis for Spring Boot user-service..."
                        withSonarQubeEnv(sonarqubeServerId) {
                            sh "mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=${JOB_NAME} -Dsonar.host.url=${SONARQUBE_URL} -Dsonar.login=${SONARQUBE_TOKEN}"
                        }
                    }
                    post {
                        always {
                            timeout(time: 5, unit: 'MINUTES') {
                                waitForQualityGate abortPipeline: true
                            }
                        }
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def serviceName = env.JOB_NAME.toLowerCase().replaceAll('/', '-')
                    def imageTag = "${env.BRANCH_NAME == 'main' ? 'latest' : env.BRANCH_NAME}-${env.BUILD_NUMBER}".replaceAll('/', '-')
                    def dockerImageName = "${dockerRegistry}/${serviceName}:${imageTag}"

                    echo "Building Docker image: ${dockerImageName}"
                    sh "docker build -t ${dockerImageName} ."

                    withCredentials([usernamePassword(credentialsId: dockerCredentialsId, usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh "echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin ${dockerRegistry.split('/')[0]}"
                        echo "Pushing Docker image: ${dockerImageName}"
                        sh "docker push ${dockerImageName}"
                    }

                    env.DOCKER_IMAGE = dockerImageName
                }
            }
        }

        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            steps {
                script {
                    def serviceName = env.JOB_NAME.toLowerCase().replaceAll('/', '-')
                    def namespace = "staging"

                    echo "Deploying ${serviceName} to Kubernetes staging cluster (${kubernetesContext})..."

                    withKubeConfig(credentialsId: kubernetesCredentialsId, contextName: kubernetesContext) {
                        sh "kubectl create namespace ${namespace} --dry-run=client -o yaml | kubectl apply -f -"

                        // Database connection details for user-service's PostgreSQL
                        def dbHost = "user-service-postgresql" // K8s service name for user DB
                        def dbPort = "5432"
                        def dbName = "user_db"
                        def dbUser = "useruser"
                        def dbPassword = "userpassword" // Replace with Jenkins Secret Text variable

                        sh "helm upgrade --install ${serviceName} ${helmChartPath} --namespace ${namespace} " +
                           "--set image.repository=${dockerRegistry}/${serviceName} " +
                           "--set image.tag=${env.DOCKER_IMAGE.split(':')[-1]} " +
                           "--set db.host=${dbHost} " +
                           "--set db.port=${dbPort} " +
                           "--set db.name=${dbName} " +
                           "--set db.user=${dbUser} " +
                           "--set db.password=${dbPassword} " +
                           "--wait --timeout 5m"

                        echo "Deployment of ${serviceName} to staging completed."
                        echo "Check the status using: kubectl get pods -n ${namespace} -l app.kubernetes.io/instance=${serviceName}"
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        failure {
            echo "Pipeline for ${env.JOB_NAME} on branch ${env.BRANCH_NAME} failed. Check logs for details."
        }
        success {
            echo "Pipeline for ${env.JOB_NAME} on branch ${env.BRANCH_NAME} succeeded!"
        }
    }
}
