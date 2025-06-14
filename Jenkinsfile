pipeline {
    agent any
    tools {
        maven 'maven'
    }

    parameters {
        choice(name: 'RUN_STAGE', choices: ['all', 'build', 'test', 'sonar', 'scan', 'package', 'login', 'aks', 'deploy'], description: 'Which stage to run')
    }

    environment{
        ACR_NAME = "terraform999"
        IMAGE_NAME = "petclinic"
        BUILD_TAG = "v${BUILD_NUMBER}"
        
        // ← this makes ALL kubectl use WORKSPACE/kubeconfig
        KUBECONFIG = "${env.WORKSPACE}/kubeconfig"

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
                    when { expression { params.RUN_STAGE == 'all' || params.RUN_STAGE == 'package' } }
                    steps {
                        echo "Packaging application into WAR"
                        sh 'mvn package'
    }
}

         stage('Build Docker Image') {
            when { expression { params.RUN_STAGE == 'all' || params.RUN_STAGE == 'login' } }
            steps{
                echo "Docker Build"
                sh """
                    ls -lh target/petclinic.war
                     docker build --build-arg WAR_FILE=petclinic.war -t $IMAGE_NAME:$BUILD_TAG .

                """
          }
         } 

         stage('Login to ACR') {
            when { expression { params.RUN_STAGE == 'all' || params.RUN_STAGE == 'login' } }
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

    stage('Tag and Push to ACR') {
        when { expression { params.RUN_STAGE == 'all' || params.RUN_STAGE == 'login' } }
  steps {
    echo "pushing to ACR"
    sh '''
      echo "Using tag: $IMAGE_NAME:$BUILD_TAG"
      docker tag $IMAGE_NAME:$BUILD_TAG $ACR_NAME.azurecr.io/$IMAGE_NAME:$BUILD_TAG
      docker push $ACR_NAME.azurecr.io/$IMAGE_NAME:$BUILD_TAG
    '''
  }
}

stage('Create ACR Secret in AKS') {
  when { expression { params.RUN_STAGE in ['all','deploy'] } }
  steps {
    withCredentials([
      string(credentialsId: 'AZURE_CLIENT_ID',     variable: 'AZ_CLIENT_ID'),
      string(credentialsId: 'AZURE_CLIENT_SECRET', variable: 'AZ_CLIENT_SECRET')
    ]) {
      sh '''
        echo "Current context: $(kubectl config current-context)"
        if ! kubectl get secret acr-auth &> /dev/null; then
          kubectl create secret docker-registry acr-auth \
            --docker-server=${ACR_NAME}.azurecr.io \
            --docker-username=$AZ_CLIENT_ID \
            --docker-password=$AZ_CLIENT_SECRET \
            --docker-email=Akshitg43@gmail.com
        else
          echo "acr-auth already exists"
        fi
      '''
    }
  }
}

stage('Deploy to AKS') {
  when { expression { params.RUN_STAGE in ['all','deploy'] } }
  steps {
    sh '''
      echo "Deploying with context: $(kubectl config current-context)"
      IMAGE_TAG=$(az acr repository show-tags \
        --name $ACR_NAME \
        --repository $IMAGE_NAME \
        --orderby time_desc --output tsv | head -n1)

      sed "s|__IMAGE_TAG__|$IMAGE_TAG|g" k8s/sprinboot-deployment.yaml \
        > k8s/sprinboot-deployment-final.yaml

      kubectl apply -f k8s/sprinboot-deployment-final.yaml
    '''
  }
}
                
    }
}
