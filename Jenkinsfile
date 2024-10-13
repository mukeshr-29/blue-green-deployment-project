pipeline{
    agent any
    tools{
        jdk 'jdk17'
        maven 'maven'
    }
    parameters{
        choice(name: 'DEPLOY_ENV', choices: ['blue', 'green'], description: 'Choose which environment to deploy: Blue or Green')
        choice(name: 'DOCKER_TAG', choices: ['blue', 'green'], description: 'Choose the Docker image tag for the deployment')
        booleanParam(name: 'SWITCH_TRAFFIC', defaultValue: false, description: 'Switch traffic between Blue and Green')
    }
    environment{
        IMAGE_NAME = "mukeshr29/bank-app"
        TAG = "${params.DOCKER_TAG}"
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages{
        stage('git checkout'){
            steps{
                git branch: 'main', url: 'https://github.com/mukeshr-29/blue-green-deployment-project.git'
            }
        }
        stage('code compilation'){
            steps{
                sh 'mvn compile'
            }
        }
        stage('code testing'){
            steps{
                sh 'mvn test -DskipTest=true'
            }
        }
        stage('trivy fs scan'){
            steps{
                sh 'trivy fs --format table -o fs-report.html .'
            }
        }
        stage('static code analysis'){
            steps{
                script{
                    withSonarQubeEnv('sonar-server'){
                        sh'''
                            $SONAR_HOME/bin/sonar-scanner -Dsonar.projectName=blue-green \
                                -Dsonar.projectKey-blue-green \
                                -Dsonar.java.binaries=target
                        '''
                    }
                }
            }
        }
        stage('quality gate check'){
            steps{
                script{
                    timeout(time: 1, unit: 'HOURS'){
                        waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                    }
                }
            }
        }
        stage('code build'){
            steps{
                sh 'mvn package -DskipTest=true'
            }
        }
        stage('publish artifact to nexus'){
            steps{
                script{
                    withMaven(globalMavenSettingsConfig: 'settings.xml', jdk: '', maven: 'maven', mavenSettingsConfig: '', traceability: true){
                        sh 'mvn deploy -DskipTest=true'
                    }
                }
            }
        }
        stage('docker img build and tag'){
            steps{
                sh 'docker build -t ${IMAGE_NAME}:${TAG} .'
            }
        }
        stage('docker img scan'){
            steps{
                sh 'trivy image --format table -o img-report.html ${IMAGE_NAME}:${TAG}'
            }
        }
        stage('docker image push'){
            steps{
                script{
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                        sh 'docker push ${IMAGE_NAME}:${TAG}'
                    }
                }
            }
        }
        stage('Deploy MySQL Deployment and Service'){
            steps{
                script{
                    withKubeConfig(caCertificate: '', clusterName: 'my-app', contextName: '', credentialsId: 'k8s', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://768722776CA1EFF16B8E20BBB6B166F3.gr7.us-east-1.eks.amazonaws.com'){
                        sh 'kubectl apply -f mysql-ds.yml'
                    }
                }
            }
        }
        stage('Deploy SVC-APP'){
            steps{
                script{
                    withKubeConfig(caCertificate: '', clusterName: 'my-app', contextName: '', credentialsId: 'k8s', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://768722776CA1EFF16B8E20BBB6B166F3.gr7.us-east-1.eks.amazonaws.com'){
                        sh """ 
                            if ! kubectl get svc bankapp-service; then
                            kubectl apply -f bankapp-service.yml
                            fi
                        """
                    }
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    def deploymentFile = ""
                    if (params.DEPLOY_ENV == 'blue') {
                        deploymentFile = 'app-deployment-blue.yml'
                    } else {
                        deploymentFile = 'app-deployment-green.yml'
                    }

                    withKubeConfig(caCertificate: '', clusterName: 'my-app', contextName: '', credentialsId: 'k8s', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://768722776CA1EFF16B8E20BBB6B166F3.gr7.us-east-1.eks.amazonaws.com') {
                        sh "kubectl apply -f ${deploymentFile}"
                    }
                }
            }
        }
        
        stage('Switch Traffic Between Blue & Green Environment') {
            when {
                expression { return params.SWITCH_TRAFFIC }
            }
            steps {
                script {
                    def newEnv = params.DEPLOY_ENV

                    // Always switch traffic based on DEPLOY_ENV
                    withKubeConfig(caCertificate: '', clusterName: 'my-app', contextName: '', credentialsId: 'k8s', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://768722776CA1EFF16B8E20BBB6B166F3.gr7.us-east-1.eks.amazonaws.com') {
                        sh '''
                            kubectl patch service bankapp-service -p "{\\"spec\\": {\\"selector\\": {\\"app\\": \\"bankapp\\", \\"version\\": \\"''' + newEnv + '''\\"}}}"
                        '''
                    }
                    echo "Traffic has been switched to the ${newEnv} environment."
                }
            }
        }
        stage('Verify Deployment') {
            steps {
                script {
                    def verifyEnv = params.DEPLOY_ENV
                    withKubeConfig(caCertificate: '', clusterName: 'my-app', contextName: '', credentialsId: 'k8s', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://768722776CA1EFF16B8E20BBB6B166F3.gr7.us-east-1.eks.amazonaws.com') {
                        sh """
                        kubectl get pods -l version=${verifyEnv}
                        kubectl get svc bankapp-service
                        """
                    }
                }
            }
        }        
    }
}