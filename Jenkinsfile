pipeline {
    agent any

    environment {
        IMAGE_NAME = "satyam88/booking-ms:dev-booking-ms-v.1.${env.BUILD_NUMBER}"
        ECR_IMAGE_NAME = "533267238276.dkr.ecr.ap-south-1.amazonaws.com/booking-ms:dev-booking-ms-v.1.${env.BUILD_NUMBER}"
        AWS_ACCOUNT_ID = "533267238276"
        REGION = "ap-south-1"
        ECR_URL = "${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com"
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

                withDockerRegistry([credentialsId: 'ecr:ap-south-1:ecr-credentials', url: "https://${ECR_URL}"]) {
                    echo "Pushing Docker Image to ECR: ${env.ECR_IMAGE_NAME}"
                    sh "docker push ${env.ECR_IMAGE_NAME}"
                    echo "Docker Image Push to ECR Completed"
                }
            }
        }

        stage('Delete Local Docker Images') {
            steps {
                script {
                    echo "Deleting Local Docker Images: ${env.IMAGE_NAME} ${env.ECR_IMAGE_NAME}"
                    // Ensure each image name is checked for null before attempting deletion
                    sh "docker rmi ${env.IMAGE_NAME} || true"
                    sh "docker rmi ${env.ECR_IMAGE_NAME} || true"
                    echo "Local Docker Images Deletion Completed"
                }
            }
        }

        stage('Deploy app to dev env') {
            when {
                expression {
                    currentBuild.rawBuild.branchName == 'dev'
                }
            }
            steps {
                script {
                    def devenvManifest = 'kubernetes/dev/'
                    def yamlFile = 'kubernetes/dev/05-deployment.yaml'
                    sh "sed -i 's/<latest>/dev-booking-v.1.${BUILD_NUMBER}/g' ${yamlFile}"
                    kubernetesDeploy(configs: 'kubernetes/dev/*.yaml', kubeconfigId: 'k8s-credentials')
                }
            }
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
