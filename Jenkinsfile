pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node22'
        maven 'maven3'
    }
    environment {
        SONAR_TOKEN = credentials('Sonar-token')  // Securely injecting Sonar token
        DOCKER_IMAGE = 'srilalithac/mp:1.0'
    }
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/psslalitha/project-1-maven-jenkins-CICD-docker-eks-.git'
            }
        }
        stage('Maven compile') { 
            steps { 
                sh 'mvn clean compile'  // Ensure full compilation before SonarQube analysis
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {  // Use SonarQube environment configured in Jenkins
                    script {
                        def scannerHome = tool 'sonar-scanner'  // Optional if Sonar Scanner is pre-configured
                        sh """
                        mvn sonar:sonar \
                        -Dsonar.projectName=pro-maven \
                        -Dsonar.login=${SONAR_TOKEN} \
                        -Dsonar.host.url=http://13.233.123.156:9000/
                        """
                    }
                }
            }
        }
        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage("Docker Image Build & Push") {
            steps {
                script {
                    sh 'docker build -t "$DOCKER_IMAGE" .'
                }
            }
        }
        
        stage('Image Scanner') {
            steps {
                script {
                    // Run Trivy scan on the Docker image
                    sh "trivy image --format json --output trivy-report.json $DOCKER_IMAGE"
                }
            }
        }
        stage('Docker Push') {
            steps {
                withDockerRegistry(credentialsId: 'docker-id') {
                    sh 'docker push "$DOCKER_IMAGE"'
                }
            }
        }
        stage('Update Deployment file') {
            environment {
                GIT_REPO_NAME = "project-1-maven-jenkins-CICD-docker-eks-"
                GIT_USER_NAME = "psslalitha"
            }
            steps {
                dir('Kubernetes-Manifests-file') {
                    withCredentials([string(credentialsId: 'github', variable: 'git_token')]) {
                        sh '''
                        git config user.email "poojitha2527@gmail.com"
                        git config user.name "psslalitha"
                        # Update the deployment YAML file with the new image
                        sed -i "s#image:.*#image: $DOCKER_IMAGE#g" deploy_svc.yml
                        
                        # Stage all modified files and untracked files
                        git add .
                        
                        # Check if there are any changes to commit
                        if [[ $(git diff --stat) != '' ]]; then
                            git commit -m "Update deployment Image"
                            git push https://${git_token}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
                        else
                            echo "No changes to commit"
                        fi
                        '''
                    }
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    dir('Kubernetes-Manifests-file') {
                        withKubeConfig(caCertificate: '', clusterName: 'project-eks', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                            sh 'aws eks update-kubeconfig --name project-eks --region ap-south-1'
                            sh 'kubectl apply -f deploy_svc.yml'
                        }
                    }
                }
            }
        }
    }
}
