pipeline {
    agent any

    parameters {
        string(name: 'DOCKER_IMAGE_TAG', defaultValue: 'latest', description: 'Tag for the Docker image')
        booleanParam(name: 'RUN_TESTS', defaultValue: true, description: 'Run simulated frontend validations')
    }

    environment {
        DOCKER_USER = '78619'
        IMAGE_NAME  = 'smart-agri'
    }

    stages {
        stage('Parallel Build & Test') {
            parallel {
                stage('Build Local Image') {
                    steps {
                        echo "Building the application container..."
                        sh "docker build -t ${DOCKER_USER}/${IMAGE_NAME}:${params.DOCKER_IMAGE_TAG} ."
                    }
                }
                stage('Run Code Analysis') {
                    when {
                        expression { return params.RUN_TESTS }
                    }
                    steps {
                        echo "Analyzing index.html for syntactic validations..."
                        sh "grep -q 'Smart Agriculture' index.html"
                    }
                }
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh "echo ${PASS} | docker login -u ${USER} --password-stdin"
                    sh "docker push ${DOCKER_USER}/${IMAGE_NAME}:${params.DOCKER_IMAGE_TAG}"
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                echo "Updating deployment manifest with the current image tag..."
                sh "sed -i 's|image: ${DOCKER_USER}/${IMAGE_NAME}:.*|image: ${DOCKER_USER}/${IMAGE_NAME}:${params.DOCKER_IMAGE_TAG}|g' deployment.yaml"
                
                echo "Applying manifest rules to Kubernetes cluster..."
                sh "kubectl apply -f deployment.yaml"
                
                echo "Monitoring rollout status..."
                script {
                    try {
                        // Monitor the rollout for up to 60 seconds
                        sh "kubectl rollout status deployment/smart-agri-deployment --timeout=60s"
                        echo "Deployment complete and fully operational!"
                    } catch (Exception e) {
                        echo "❌ ROLLOUT FAILED DETECTED! Initiating automatic recovery rollback..."
                        sh "kubectl rollout undo deployment/smart-agri-deployment"
                        sh "kubectl rollout status deployment/smart-agri-deployment"
                        error("Pipeline stopped: Deployment was unstable and rolled back automatically.")
                    }
                }
            }
        }
    }
}
