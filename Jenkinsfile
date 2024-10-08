def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
]
pipeline
{
    agent any
    tools
    {
        maven 'Maven-3.9.9'
        jdk 'OracleJDK11'
    }

    environment
    {
        registryCredential = 'azure-credentials' // Update with your Azure credentials ID
        appRegistry = 'vprofile.azurecr.io/vprofileappimg' // Update with your ACR login server and repository name
        vprofileRegistry = 'https://vprofile.azurecr.io'
        // nexusRegistry = 'http://74.225.230.84:8081/repository/vprofile'
        nexusRegistry = '74.225.230.84:8082'
        //nexusCredentials = credentials('nexuslogin') //This need to check
        nexusCredentials = 'nexuslogin' // Nexus credentials ID
        nexusImage = "${nexusRegistry}/vprofileappimg" // Nexus repository and image path
        RESOURCE_GROUP = 'dockerk8s' // Update with your Azure resource group
        CONTAINER_APP_NAME = 'vprofile' // Update with your Azure Container App name
        AZURE_CREDENTIALS_ID = 'AzureServicePrincipal' // Azure service principal credentials ID
        }

    stages
    {
        stage('Fetch code')
        {
            steps {
                git branch: 'docker', url: 'https://github.com/devopshydclub/vprofile-project.git'
            }
        }

        stage('Test')
        {
            steps {
                sh 'mvn test'
            }
        }

        stage('Code Analysis with Checkstyle')
        {
            steps
            {
                    sh 'mvn checkstyle:checkstyle'
            }
            post
            {
                success
                {
                    echo 'Generated Analysis Result'
                }
            }
        }

        stage('Build & SonarQube Analysis')
        {
            environment
            {
                scannerHome = tool 'sonar4.7'
            }
            steps
            {
                withSonarQubeEnv('sonar')
                {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }
            }
        }

        stage('Quality Gate') {
            steps 
            {
                timeout(time: 1, unit: 'HOURS') 
                {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build App Image') 
        {
            steps 
            {
                script 
                {
                    dockerImage = docker.build(appRegistry + ":$BUILD_NUMBER", "./Docker-files/app/multistage/")
                }
            }
        }

        stage('Upload App Image to ACR') 
        {
            steps 
            {
                script 
                {
                    docker.withRegistry(vprofileRegistry, registryCredential) 
                    {
                        dockerImage.push("$BUILD_NUMBER")
                        dockerImage.push('latest')
                    }
                }
            }
        }
        stage('Upload App Image to Nexus') 
        {
            steps 
            {
                script 
                {
                    // Login to Nexus Repository
                    withCredentials([usernamePassword(credentialsId: nexusCredentials, usernameVariable: 'NEXUS_USERNAME', passwordVariable: 'NEXUS_PASSWORD')]) {
                        sh "docker login ${nexusRegistry} -u ${NEXUS_USERNAME} -p ${NEXUS_PASSWORD}"
                    }

                    // Tag the image for Nexus
                    sh "docker tag ${appRegistry}:${BUILD_NUMBER} ${nexusImage}:${BUILD_NUMBER}"
                    sh "docker tag ${appRegistry}:latest ${nexusImage}:latest"

                    // Push the image to Nexus
                    sh "docker push ${nexusImage}:${BUILD_NUMBER}"
                    sh "docker push ${nexusImage}:latest"
                }
            }
        }
        stage('Deploy to Azure Container Apps') 
        {
            steps 
            {
                script {
                    withCredentials([azureServicePrincipal(credentialsId: AZURE_CREDENTIALS_ID)]) 
                    {
                        sh """
                        az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET --tenant $AZURE_TENANT_ID
                        az containerapp update --name ${CONTAINER_APP_NAME} --resource-group ${RESOURCE_GROUP} --image ${appRegistry}:${BUILD_NUMBER}
                        """
                    }
                }
            }
        }
    }
    post
    {
        always
        {
            echo 'Slack Notifications.'
            slackSend channel: '#jenkinscicd',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
        }
    }
}
