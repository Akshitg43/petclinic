pipeline {
    agent any
    tools {
        maven 'maven'
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
                withSonarQubeEnv('sonar') {
                    withCredentials([string(credentialsId: 'sonar-pw', variable: 'SONAR_TOKEN')]) {
                        sh '''
                            sonar \
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
        }
    }
}
