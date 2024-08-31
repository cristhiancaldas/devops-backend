pipeline {
    agent any 
    tools {
        jdk 'jdk'
        nodejs 'nodejs'
    }
   environment {
        DOCKER_HUB_LOGIN = credentials('dockerHub')
        IMAGE='app-backend'
        REGISTRY='crist'
        SCANNER_HOME = tool 'SonarQubeScanner'
    }

    stages {

        stage('Cleaning Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main',credentialsId: 'GITHUB', url: 'https://github.com/cristhiancaldas/devops-backend.git'
            }
        }

        stage('SonarQube analysis') {
            
            steps {
                withSonarQubeEnv('SonarQube') {
                      sh ''' $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=three-tier-banckend \
                        -Dsonar.projectKey=three-tier-backend '''
                }
            }
        }
        
        stage('Quality Check') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'jenkins' 
                }
            }
        }

        stage('OWASP Dependency-Check Scan') {
            steps {  

                 dependencyCheck additionalArguments: ''' 
                    -o './'
                    -s './'
                    -f 'ALL' 
                    --prettyPrint''', odcInstallation: 'DP-Check'
        
                dependencyCheckPublisher pattern: 'dependency-check-report.xml'            
            }
        }
        
        stage('Trivy File Scan') {
            steps {
                    sh 'trivy fs . > trivyfs.txt'          
            }
        }
        
        stage('build') {
            steps {
               sh 'docker build --platform linux/amd64 -t $IMAGE:${BUILD_NUMBER} .'
            }
        }
        
        stage('deploy to hub') {
            steps {
               sh '''
               docker login --username=$DOCKER_HUB_LOGIN_USR --password=$DOCKER_HUB_LOGIN_PSW
               docker tag $IMAGE:${BUILD_NUMBER} $REGISTRY/$IMAGE:${BUILD_NUMBER}
               docker push $REGISTRY/$IMAGE:${BUILD_NUMBER}
               '''
            }
        }
  
    }
}