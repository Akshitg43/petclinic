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
        iMAGE_NAME = "petclinic"
        BUILD_TAG = "v${BUILD_NUMBER}"

    }
    stages {
        stage('Checkout From Git') {
            when { expression { params.RUN_STAGE == 'all' || params.RUN_STAGE == 'build' || params.RUN_STAGE == 'test' || params.RUN_STAGE == 'sonar' || params.RUN_STAGE == 'scan' } }
            steps {
                git branch: 'prod', url: 'https://github.com/bkrrajmali/enahanced-petclinc-springboot.git'
            }
        }
        stage('Maven Compile') {
            when { expression { params.RUN_STAGE == 'all' || params.RUN_STAGE == 'build' || params.RUN_STAGE == 'test' } }
            steps {
                echo "This is Maven Compile Stage"
                sh "mvn compile"
            }
        }
        stage('Maven Test') {
            when { expression { params.RUN_STAGE == 'all' || params.RUN_STAGE == 'test' } }
            steps {
                echo "This is Maven Test Stage"
                sh "mvn test"
            }
        }
        stage('File Scanning by Trivy') {
            when { expression { params.RUN_STAGE == 'all' || params.RUN_STAGE == 'scan' } }
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
            when { expression { params.RUN_STAGE == 'all' || params.RUN_STAGE == 'sonar' } }
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
                stage('Maven Package') {
                    when { expression { params.RUN_STAGE == 'all' || params.RUN_STAGE == 'build' } }
                    steps {
                        echo "Packaging application into WAR"
                        sh 'mvn package'
    }
}

         stage('Build Docker Image') {
            when { expression { params.RUN_STAGE == 'all' || params.RUN_STAGE == 'build' } }
            steps{
                echo "Docker Build"
                sh """
                    ls -lh target/petclinic.war
                     docker build --build-arg WAR_FILE=petclinic.war -t $IMAGE_NAME:$BUILD_TAG .

                """
          }
         } 

         stage('Login to ACR') {
      steps {
           echo "ACR Login"
        withCredentials([
          string(credentialsId: 'AZURE_CLIENT_ID', variable: 'AZ_CLIENT_ID'),
          string(credentialsId: 'AZURE_CLIENT_SECRET', variable: 'AZ_CLIENT_SECRET'),
          string(credentialsId: 'AZURE_TENANT_ID', variable: 'AZ_TENANT_ID')
        ]) {
          sh '''
            az login --service-principal -u $AZ_CLIENT_ID -p $AZ_CLIENT_SECRET --tenant $AZ_TENANT_ID
            az acr login --name $ACR_NAME
          '''
        }
      }
    }

        }
       }                        
