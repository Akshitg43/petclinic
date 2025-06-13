pipeline {
    agent any
    tools {
        maven 'maven'
    }

    parameters {
        choice(name: 'RUN_STAGE', choices: ['all', 'build', 'test', 'sonar', 'scan'], description: 'Which stage to run')
    }

    environment{
        ACR_NAME = "terraform999"
        iMAGE_NAME = "PETCLINIC"
        BUILD_TAG = "V${BUILD_NUMBER}"

    }
    stages {
        stage('Checkout From Git') {
            steps {
                git branch: 'prod', url: 'https://github.com/bkrrajmali/enahanced-petclinc-springboot.git'
            }
        }
        stage('Maven Compile') {
            steps {
                echo "This is Maven Compile Stage"
                sh "mvn compile"
            }
        }
        stage('Maven Test') {
            steps {
                echo "This is Maven Test Stage"
                sh "mvn test"
            }
        }
        stage('File Scanning by Trivy') {
            steps {
                echo "Trivy Scanning"
                sh '''
                    mkdir -p /tmp/scan-src
                    rsync -a --no-perms --no-owner --no-group ./ /tmp/scan-src/
                    cd /tmp/scan-src
                    trivy fs --format table --output trivy-report.txt --severity HIGH,CRITICAL .
                    cp trivy-report.txt $WORKSPACE/
                '''
            }
        }
        stage('Sonar Scanning') {
            steps {
                echo "Sonar scanning"
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        sonar-scanner \
                        -Dsonar.projectKey=akscluster_petclinic-jks \
                        -Dsonar.sources=. \
                        -Dsonar.organization=akscluster \
                        -Dsonar.host.url=https://sonarcloud.io \
                        -Dsonar.java.binaries=. \
                        -Dsonar.exclusions=**/trivy-report.txt
                    '''
                }
             }
        }
         stage('Build Docker Image') {
            steps{
                echo "Docker Build"
                sh """
                    docker build -t $IMAGE_NAME:$BUILD_TAG .
                """
          }
         }   
        }
       }                        
