pipeline {
    agent any
    tools {
        maven 'maven'
        sonarQubeScanner 'sonar_scanner'
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
                sh " mvn compile"
            }
        }
        stage('Maven Test') {
            steps {
                echo "This is Maven Test Stage"
                sh " mvn test"
            }
        }
        stage('File Scanning by Trivy') {
            steps {
                echo "Trivy Scanning"
                sh  'trivy fs --format table --output trivy-report.txt --severity HIGH,CRITICAL .'
            }
        }
        stage('sonar Scanning'){
            steps{
                echo "Sonar scanning"
                withSonarQubeEnv('sonar_scanner'){
                sh 'sonar-scanner -Dsonar.projectKey=akscluster_petclinic-jks -Dsonar.sources=. -Dsonar.host.url=https://sonarcloud.io/project/information?id=akscluster_petclinic-jks -Dsonar.login=34c0daf5fc82f45cf43c8ca0af170bd0f1093e08'
                }

            }
        }
    
    }

}
