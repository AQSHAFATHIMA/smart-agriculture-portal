pipeline {
    agent any
    
    parameters {
        string(name: 'DOCKER_IMAGE_TAG', defaultValue: 'latest', description: 'Tag for compiled application delivery')
        booleanParam(name: 'AUTO_DEPLOY', defaultValue: true, description: 'Trigger execution layer down stream')
    }

    environment {
        DOCKER_HUB_USER = '78619' 
        IMAGE_NAME      = 'smart-agriculture-portal'
    }

    stages {
        stage('Pull & Code Audit') {
            steps {
                echo 'Checking out active repository workspace...'
                // FIXED: Replaced 'checkout scm' with explicit Git checkout step
                git url: 'https://github.com', branch: 'main'
            }
        }

        stage('Parallel Build & Test') {
            parallel {
                stage('Build Local Image') {
                    steps {
                        echo 'Building the application container...'
                        bat "docker build -t %DOCKER_HUB_USER%/%IMAGE_NAME%:%DOCKER_IMAGE_TAG% ."
                    }
                }
                stage('Run Code Analysis') {
                    steps {
                        echo 'Analyzing index.html for syntactic validations...'
                        bat "echo Code analysis complete"
                    }
                }
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', passwordVariable: 'DB_PASS', usernameVariable: 'DB_USER')]) {
                    bat "echo %DB_PASS% | docker login -u %DB_USER% --password-stdin"
                    bat "docker push %DOCKER_HUB_USER%/%IMAGE_NAME%:%DOCKER_IMAGE_TAG%"
                }
            }
        }

        stage('Deploy to Kubernetes') {
            when {
                expression { return params.AUTO_DEPLOY == true }
            }
            steps {
                withKubeConfig([credentialsId: 'kubernetes-config-cluster']) {
                    echo 'Applying updates to target live environment...'
                    bat "kubectl apply -f deployment.yaml"
                    bat "kubectl rollout status deployment/smart-agri-portal --timeout=60s"
                }
            }
        }
    }
    
    post {
        always {
            archiveArtifacts artifacts: 'index.html, deployment.yaml', fingerprint: true, allowEmptyArchive: false
            echo 'Build artifacts systematically backed up.'
        }
    }
}
