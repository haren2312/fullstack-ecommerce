pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out the code'
                git branch: 'main', url: 'https://github.com/Harsha072/fullstack-ecommerce.git'
            }
        }
        stage('Verify Docker Setup') {
            steps {
                echo 'Verifying Docker setup...'
                script {
                    def dockerCheck = bat(script: 'docker --version', returnStatus: true)
                    if (dockerCheck != 0) {
                        error 'Docker command failed. Ensure Docker is installed and Jenkins user has access to it.'
                    }
                }
            }
        }
        
        stage('Install Dependencies - Maven') {
            steps {
                dir('backend\\spring-boot-ecommerce') {
                    echo 'Installing Maven dependencies...'
                    bat 'mvn clean install -DskipTests'
                }
            }
        }
        stage('Build JAR - Maven') {
            steps {
                dir('backend\\spring-boot-ecommerce') {
                    echo 'Building JAR file...'
                    bat 'mvn package -DskipTests'
                }
            }
        }
        stage('Build Docker Image - Spring Boot') {
            steps {
                dir('backend\\spring-boot-ecommerce') {
                    echo 'Building Docker image for Spring Boot...'
                    script {
                        def imageName = 'spring-boot-ecommerce'
                        def dockerBuild = bat(script: "docker build -t ${imageName}:${env.BUILD_NUMBER} -t ${imageName}:latest .", returnStatus: true)
                        if (dockerBuild != 0) {
                            error 'Docker image build failed for Spring Boot.'
                        }
                        echo 'Docker image for Spring Boot built successfully.'
                    }
                }
            }
        }
        stage('Deploy to ECR') {
            steps {
                echo 'Authenticating Docker to AWS ECR and pushing images...'
                script {
                    withCredentials([[
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials'
                    ]]) {
                        env.AWS_DEFAULT_REGION = 'us-east-1'

                        def ecrLogin = bat(script: 'aws ecr get-login-password --region %AWS_DEFAULT_REGION% | docker login --username AWS --password-stdin 242201280065.dkr.ecr.us-east-1.amazonaws.com', returnStatus: true)
                        if (ecrLogin != 0) {
                            error 'Failed to login to AWS ECR. Please check your credentials and region.'
                        }

                        def springBootTag = bat(script: 'docker tag spring-boot-ecommerce:latest 242201280065.dkr.ecr.us-east-1.amazonaws.com/spring-boot-ecommerce:latest', returnStatus: true)
                        if (springBootTag != 0) {
                            error 'Failed to tag Docker image for Spring Boot.'
                        }
                        def springBootPush = bat(script: 'docker push 242201280065.dkr.ecr.us-east-1.amazonaws.com/spring-boot-ecommerce:latest', returnStatus: true)
                        if (springBootPush != 0) {
                            error 'Failed to push Docker image for Spring Boot.'
                        }

                        def angularTag = bat(script: 'docker tag angular-ecommerce:latest 242201280065.dkr.ecr.us-east-1.amazonaws.com/angular-ecommerce:latest', returnStatus: true)
                        if (angularTag != 0) {
                            error 'Failed to tag Docker image for Angular.'
                        }
                        def angularPush = bat(script: 'docker push 242201280065.dkr.ecr.us-east-1.amazonaws.com/angular-ecommerce:latest', returnStatus: true)
                        if (angularPush != 0) {
                            error 'Failed to push Docker image for Angular.'
                        }

                        echo 'Docker images pushed to ECR successfully.'
                    }
                }
            }
        }

        stage('Update ECS Task Definition') {
            steps {
                echo 'Updating ECS task definition with new Docker image...'
                script {
                    def taskDefinitionName = 'my-task-family' // Replace with your task definition name
                    def newImageUri = "242201280065.dkr.ecr.us-east-1.amazonaws.com/spring-boot-ecommerce:latest"

                    // Step 2: Get the existing task definition
                    def taskDefJson = bat(script: "aws ecs describe-task-definition --task-definition ${taskDefinitionName} --region us-east-1", returnStdout: true).trim()
                    
                    // Step 3: Update the task definition with the new Docker image
                    def updatedTaskDefJson = taskDefJson.replaceFirst(/"image":\s*"[^"]+"/, "\"image\": \"${newImageUri}\"")

                    // Save the updated JSON to a file
                    writeFile file: 'task-def.json', text: updatedTaskDefJson

                    // Step 4: Register the new task definition revision
                    def registerResult = bat(script: "aws ecs register-task-definition --cli-input-json file://task-def.json --region us-east-1", returnStatus: true)
                    if (registerResult != 0) {
                        error 'Failed to register the new task definition.'
                    }

                    echo 'ECS task definition updated and registered successfully.'
                }
            }
        }
    }

    post {
        success {
            echo 'All tests, builds, and deployments completed successfully and pushed to AWS ECR!'
        }
        failure {
            echo 'Some tests, builds, or deployments failed.'
        }
    }
}
