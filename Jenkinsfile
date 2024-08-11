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
                git branch: 'main', url: 'https://github.com/asifkhazi/TWSThreeTierAppChallenge.git'
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
                helm install fronend fronend/ --namespace three-tier --create-namespace
                helm install backend backend/ --namespace three-tier --create-namespace
                helm install database database/ --namespace three-tier --create-namespace
            }
        }
    }
}
---
@Library('sharedlibrary')_
pipeline {
    agent {label 'docker'}
    environment {
        brnachName=sh(script: 'echo ${GIT_BRANCH} | sed "s#/#-#"', returnStdout: true).trim()
        repoName=sh(script: "basename -s .git ${GIT_URL}", returnStdout: true).trim()
        dockerImage="124561635468.dkr.ecr.ap-south-2.amazonaws.com/${repoName}"
        dockerTag="${branchName}${GIT_COMMIT[0..6]}"
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage ('SCM Checkout') {
            steps {
                git branch: '${GIT_BRANCH}', url: '${GIT_URL}'
            }
        }

        stage ('SonarQube Analysis') {
            steps {
                withSonarQubeEnv ('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=${repoName} \
                    -Dsonar.projectKey=${repoName}
                    '''
                }
            }
        }

        stage ('Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: false, credentialsId: 'sonarToken'
            }
        }

        stage ('Docker Build') {
            steps {
                sh 'docker build -t ${dockerImage}:${dockerTag} .'
            }
        }

        stage ('Trivvy Scan') {
            steps {
                sh 'trivy image -f json -o results-${BUILD_NUMBER} ${dockerImage}:${dockerTag}'
            }
        }

        stage ('Docker Push') {
            steps {
                sh 'echo $dockerCred_PSW | docker login --username AWS --password-stdin ${ecrAccount}'
            }
        }

        stage ('Kubernetest Deploy -Dev'){
            when {
                branch 'development'
            }
            steps {
                sh 'helm install ${repoName} -f dev.yaml'
            }
        }

        stage ('Kubernetest Deploy -master_staging') {
            when {
                branch 'master_staging'
            }
            steps {
                sh 'helm install ${reponame} -f staging.yaml'
            }
        }

        stage ('Kubernetest Deploy -PROD') {
            when {
                branch 'master'
            }
            steps {
                sh 'helm install ${reponame} -f master.yaml'
            }
        }
    }
}
---
@Library('sharedlibrary')
pipeline {
    agent {label 'docker'}
    environment {
        gitRepo=sh(script: "basename -s .git ${GIT_URL}" ,returnStdout: true).trim()
        gitBranch=sh(script: 'echo ${GIT_BRANCH} | sed "s#/#-#"' ,returnStdout: true).trim()
        dockerImage = "${ecrAccount}/${repoName}"
        dockerTag="${GIT_COMMIT[0..6]}"
        CSANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage ('SCM Checkou') {
            steps {
                git branch: ${GIT_BRANCH}, credentialsId: '', url: "${GIT_URL}"
            }
        }

         stage ('Sonarqube Analysis') {
            steps {
               withSonarQubeEnv ('sonar-server') {
                    sh '''${SCANNER_HOME}/bin/sonar-scanner
                    -Dsonar.projectName=${repoName}
                    -Dsonar.projectKey=${repoName}
                    '''
               }
            }
        }

        stage ('Docker Build') {
            steps {
                dir('./fronend') {
                    sh 'docker build -t ${dockerImage}:${dockerTag} .'
                }
                dir('./backend') {
                    sh 'docker build -t ${dockerImage}:${dockerTag} .'
                }
            }
        }
        stage ('Docker Push') {
            steps {
                sh 'echo $dockerCred_PSW | docker login --username ${dockerCred_USR} --password-stdin'
                sh 'docker push ${dockerImage}:${dockerTag}'
                sh 'docker push ${dockerImage}:${dockerTag}'
            }
        }
        stage ('K8s Deploy') {
            steps {
                sh 'helm install $chartName'
            }
        }
    }
}