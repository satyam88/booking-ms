pipeline {
    agent any

    environment {
        IMAGE_NAME = "satyam88/booking-ms:dev-booking-ms-v.1.${env.BUILD_NUMBER}"
        ECR_IMAGE_NAME = "533267238276.dkr.ecr.ap-south-1.amazonaws.com/booking-ms:dev-booking-ms-v.1.${env.BUILD_NUMBER}"
        // NEXUS_IMAGE_NAME = "3.110.216.145:8085/booking-ms:dev-booking-ms-v.1.${env.BUILD_NUMBER}"
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '5', artifactNumToKeepStr: '5'))
    }

    tools {
        maven 'maven_3.9.4'
        // sonarqubeScanner 'sonarqube-scanner'
    }

    stages {
        stage('Code Compilation') {
            steps {
                echo 'Code Compilation is In Progress!'
                sh 'mvn clean compile'
                echo 'Code Compilation is Completed Successfully!'
            }
        }
        /*
        stage('Sonarqube Code Quality') {
            environment {
                scannerHome = tool 'sonarqube-scanner'
            }
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh "${scannerHome}/bin/sonar-scanner"
                    sh 'mvn sonar:sonar'
                }
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        */
        stage('Code QA Execution') {
            steps {
                echo 'JUnit Test Case Check in Progress!'
                sh 'mvn clean test'
                echo 'JUnit Test Case Check Completed!'
            }
        }
        stage('Code Package') {
            steps {
                echo 'Creating WAR Artifact'
                sh 'mvn clean package'
                echo 'Artifact Creation Completed'
            }
        }
        stage('Building & Tag Docker Image') {
            steps {
                echo "Starting Building Docker Image: ${env.IMAGE_NAME}"
                sh "docker build -t ${env.IMAGE_NAME} ."
                echo 'Docker Image Build Completed'
            }
        }
        /*
        stage('Docker Image Scanning') {
            steps {
                echo 'Docker Image Scanning Started'
                // Add actual scanning steps here
                echo 'Docker Image Scanning Completed'
            }
        }
        */
        stage('Docker Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'DOCKER_HUB_CRED', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                    echo "Pushing Docker Image to DockerHub: ${env.IMAGE_NAME}"
                    sh "docker login -u ${DOCKER_USERNAME} -p ${DOCKER_PASSWORD}"
                    sh "docker push ${env.IMAGE_NAME}"
                    echo "Docker Image Push to DockerHub Completed"
                }
            }
        }
        stage('Docker Image Push to Amazon ECR') {
            steps {
                echo "Tagging Docker Image for ECR: ${env.ECR_IMAGE_NAME}"
                sh "docker tag ${env.IMAGE_NAME} ${env.ECR_IMAGE_NAME}"
                echo "Docker Image Tagging Completed"

                withDockerRegistry([credentialsId: 'ecr:ap-south-1:ecr-credentials', url: "https://533267238276.dkr.ecr.ap-south-1.amazonaws.com"]) {
                    echo "Pushing Docker Image to ECR: ${env.ECR_IMAGE_NAME}"
                    sh "docker push ${env.ECR_IMAGE_NAME}"
                    echo "Docker Image Push to ECR Completed"
                }
            }
        }
        /*
        stage('Upload the Docker Image to Nexus') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'nexus-credentials', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                        sh 'docker login http://3.110.216.145:8085/repository/booking-ms/ -u admin -p ${PASSWORD}'
                        echo "Push Docker Image to Nexus: In Progress"
                        sh "docker tag ${env.IMAGE_NAME} ${env.NEXUS_IMAGE_NAME}"
                        sh "docker push ${env.NEXUS_IMAGE_NAME}"
                        echo "Push Docker Image to Nexus: Completed"
                    }
                }
            }
        }
        */
        stage('Delete Local Docker Images') {
            steps {
                script {
                    echo "Deleting Local Docker Images: ${env.IMAGE_NAME} ${env.ECR_IMAGE_NAME} ${env.NEXUS_IMAGE_NAME}"
                    // Ensure each image name is checked for null before attempting deletion
                    sh "docker rmi ${env.IMAGE_NAME ?: ''} ${env.ECR_IMAGE_NAME ?: ''} ${env.NEXUS_IMAGE_NAME ?: ''}"
                    echo "Local Docker Images Deletion Completed"
                }
            }
        }
        stage('Deploy app to dev env') {
            steps {
                script {
                    def yamlFile = 'kubernetes/dev/05-deployment.yaml'
                    def versionedImage = "dev-booking-v.1.${BUILD_NUMBER}"

                    // Replace <latest> with the versioned image tag in the YAML file
                    sh "sed -i 's/<latest>/${versionedImage}/g' ${yamlFile}"

                    // Deploy to Kubernetes
                    kubernetesDeploy(
                        configs: yamlFile,
                        kubeconfigId: 'my-kubeconfig',
                        kubeconfig: KUBE_CONFIG,
                        onFailure: 'abort', // Abort pipeline on deployment failure
                        showToken: true, // Display Kubernetes token for debug
                        verifySSL: false // Disable SSL verification (if needed)
                    )
                }
            }
            post {
                success {
                    echo "Deployment to dev environment completed successfully"
                }
                failure {
                    echo "Deployment to dev environment failed. Check logs for details."
                }
            }
        }
    }
}
