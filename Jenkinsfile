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
        IMAGE_TAG = "${BUILD_NUMBER}"
        DOCKER_REPO = "app-backend"
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

        stage('Checkout Code') {
            steps {
                git  branch: 'main' ,credentialsId: 'GITHUB', url: 'https://github.com/cristhiancaldas/kubernetes-Manifests-file.git'
            }
        }

        stage('Update Deployment file') {
            environment {
                GIT_REPO_NAME = "kubernetes-Manifests-file"
                GIT_USER_NAME = "cristhiancaldas"
            }
            steps {
                dir('Backend') {
                    withCredentials([gitUsernamePassword(credentialsId: 'GITHUB', variable: 'GITHUB_TOKEN')]) {
                        sh '''
                            git config user.email "c.caldas.m@gmail.com"
                            git config user.name "cristhiancaldas"
                            BUILD_NUMBER=${BUILD_NUMBER}
                            echo $BUILD_NUMBER
                            imageTag=$(grep -oP '(?<=app-backend:)[^ ]+' deployment.yaml)
                            echo $imageTag
                            sed -i "s/${DOCKER_REPO}.*/${DOCKER_REPO}:${BUILD_NUMBER}/g" deployment.yaml
                            git add deployment.yaml
                            git commit -m "Update deployment Image to version \${BUILD_NUMBER}"
                            git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
                        '''
                    }
                }
            }
        }     
  
    }
}