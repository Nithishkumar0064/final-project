pipeline {
    agent {label 'docker'}
    environment {
        repoName="threetier-app"
        dockerCred = credentials('dockerCred')
        dockerImage="$dockerCred_USR/$repoName"
        SCANNER_HOME= tool 'sonar-scanner'
    }
    stages {
        stage ('SCM Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Nithishkumar0064/final-project.git'
            }
        }

        stage ('SonarQube Analysis'){
            steps {
                dir('Application-Code/frontend') {
                    withSonarQubeEnv('sonar-server') {
                        sh '''$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=three-tier-frontend \
                        -Dsonar.projectKey=three-tier-frontend '''
                    }
                }
                dir('Application-Code/backend') {
                    withSonarQubeEnv('sonar-server') {
                        sh '''$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=three-tier-backend \
                        -Dsonar.projectKey=three-tier-backend '''
                    }
                }
            }
        }

        stage('Quality Check') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonarToken' 
                }
            }
        }

        stage ('Docker Build') {
            steps {
                dir('Application-Code/frontend') { 
                        sh 'docker build -t $dockerImage:threetier-frontend .'
                }
                dir('Application-Code/backend') { 
                        sh 'docker build -t $dockerImage:threetier-backend .'
                }
            }
        }

        stage("TRIVY Image Scan") {
            steps {
                sh 'trivy image -f json -o results-${BUILD_NUMBER}.json $dockerImage:threetier-frontend'
                sh 'trivy image -f json -o results-${BUILD_NUMBER}.json $dockerImage:threetier-backend' 
            }
        }

        stage ('Docker Push') {
            steps {
                sh '''
                    echo "$dockerCred_PSW" | docker login --username $dockerCred_USR --password-stdin
                    docker push $dockerImage:threetier-frontend
                    docker push $dockerImage:threetier-backend
                    docker image prune -a
                '''
            }
        }

        stage ('Kubernetes-Deploy') {
            steps {
                sh 'helm install fronend helm/fronend/ --namespace three-tier --create-namespace'
                sh 'helm install backend helm/backend/ --namespace three-tier --create-namespace'
                sh 'helm install database helm/database/ --namespace three-tier --create-namespace'
            }
        }
    }
}
