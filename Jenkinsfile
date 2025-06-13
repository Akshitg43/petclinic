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

stage('Create & Login to AKS Cluster') {
  when { expression { params.RUN_STAGE == 'all' || params.RUN_STAGE == 'aks' } }
  steps {
    echo "Checking/Creating and Logging into AKS Cluster: jkspipeline"
    withCredentials([
      string(credentialsId: 'AZURE_CLIENT_ID', variable: 'AZ_CLIENT_ID'),
      string(credentialsId: 'AZURE_CLIENT_SECRET', variable: 'AZ_CLIENT_SECRET'),
      string(credentialsId: 'AZURE_TENANT_ID', variable: 'AZ_TENANT_ID'),
      string(credentialsId: 'AZURE_SUBSCRIPTION_ID', variable: 'AZ_SUBSCRIPTION_ID')
    ]) {
      sh '''
        az login --service-principal -u $AZ_CLIENT_ID -p $AZ_CLIENT_SECRET --tenant $AZ_TENANT_ID
        az account set --subscription $AZ_SUBSCRIPTION_ID

        RESOURCE_GROUP="jks"
        CLUSTER_NAME="jkspipeline"

        echo "Checking if AKS cluster exists..."
        if az aks show --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME > /dev/null 2>&1; then
          echo "Cluster '$CLUSTER_NAME' already exists."
        else
          echo "Creating AKS cluster '$CLUSTER_NAME'..."
          az aks create \
            --resource-group $RESOURCE_GROUP \
            --name $CLUSTER_NAME \
            --node-count 1 \
            --enable-addons monitoring \
            --generate-ssh-keys
        fi

        echo "Logging into AKS cluster '$CLUSTER_NAME'..."
        az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --overwrite-existing
      '''
    }
  }
}

stage('Deploy to AKS') {
  when { expression { params.RUN_STAGE == 'all' || params.RUN_STAGE == 'deploy' } }
  steps {
    echo "Deploying to AKS if not already deployed"
    withCredentials([
      string(credentialsId: 'AZURE_CLIENT_ID', variable: 'AZ_CLIENT_ID'),
      string(credentialsId: 'AZURE_CLIENT_SECRET', variable: 'AZ_CLIENT_SECRET'),
      string(credentialsId: 'AZURE_TENANT_ID', variable: 'AZ_TENANT_ID'),
      string(credentialsId: 'AZURE_SUBSCRIPTION_ID', variable: 'AZ_SUBSCRIPTION_ID')
    ]) {
      sh '''
        RESOURCE_GROUP="jks"
        CLUSTER_NAME="jkspipeline"
        ACR_NAME="terraform999"
        IMAGE_REPO="petclinic"

        echo "Logging into Azure..."
        az login --service-principal -u $AZ_CLIENT_ID -p $AZ_CLIENT_SECRET --tenant $AZ_TENANT_ID
        az account set --subscription $AZ_SUBSCRIPTION_ID

        echo "Ensuring AKS cluster context is set..."
        az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --overwrite-existing

        echo "Checking if Kubernetes deployment 'springboot-app' exists..."
        if kubectl get deployment springboot-app > /dev/null 2>&1; then
          echo "Deployment 'springboot-app' already exists. Skipping deployment."
        else
          echo "Fetching latest image tag from ACR..."
          IMAGE_TAG=$(az acr repository show-tags --name $ACR_NAME --repository $IMAGE_REPO --orderby time_desc --output tsv | head -n 1)

          if [ -z "$IMAGE_TAG" ]; then
            echo "ERROR: No image tags found in ACR for $IMAGE_REPO"
            exit 1
          fi

          echo "Injecting image tag: $IMAGE_TAG"
          sed "s|__IMAGE_TAG__|$IMAGE_TAG|g" k8s/sprinboot-deployment.yaml > k8s/sprinboot-deployment-final.yaml

          echo "Deploying application to AKS..."
          kubectl apply -f k8s/sprinboot-deployment-final.yaml
        fi
      '''
    }
  }
}
        }
       }                        
