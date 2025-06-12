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
                sh 'trivy fs --format table --output trivy-report.txt --severity HIGH,CRITICAL .'
            }
        }
        stage('Sonar Scanning') {
            steps {
                echo "Sonar scanning"
                withSonarQubeEnv('sonar_scanner') {
                    withCredentials([string(credentialsId: 'a7e196d0-968e-4f0a-ae11-eaba1e4e1240', variable: 'SONAR_TOKEN')]) {
                        sh '''
                            sonar-scanner \
                            -Dsonar.projectKey=akscluster_petclinic-jks \
                            -Dsonar.sources=. \
                            -Dsonar.organization=akscluster \
                            -Dsonar.host.url=https://sonarcloud.io \
                            -Dsonar.login=$SONAR_TOKEN
                            -Dsonar.java.binaries =.\
                            -Dsonar.exclusions=**/trivy-report.txt

                        '''
                    }
                }
            }
        }
    }
}
