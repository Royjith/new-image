pipeline {
    agent { label 'node-1' } // Set the agent to use node-1

    environment {
        DOCKER_IMAGE = 'my-app'               // Docker image name
        DOCKER_TAG = 'version-1'              // Static Docker tag set to 'version-1'
        DOCKER_HUB_REPO = 'royjith/pikube'    // Docker Hub repository
        DOCKER_HUB_CREDENTIALS_ID = 'dockerhub' // Docker Hub credentials ID
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code from Git...'
                // Checkout the code from the 'main' branch
                git branch: 'main', credentialsId: 'dockerhub', url: 'https://github.com/Royjith/demo.git'
            }
        }

        stage('Build Docker Image') {
    steps {
        script {
            // Use the static Docker tag "version-1"
            def tag = "${DOCKER_TAG}"
            echo "Building Docker image with tag: ${tag}..."

            try {
                // Clean up any existing images with the same name to prevent issues
                sh """
                    docker rmi -f ${DOCKER_HUB_REPO}:${tag} || true
                    docker system prune -f
                """

                // Build the Docker image and tag it as "latest"
                echo "Building the Docker image from Dockerfile..."
                sh "docker build -f Dockerfile --no-cache -t ${DOCKER_HUB_REPO}:latest ."

                // Tag the image with the appropriate repository and tag
                echo "Tagging the Docker image with tag: ${DOCKER_HUB_REPO}:${tag}..."
                sh """
                    IMAGE_ID=\$(docker images -q ${DOCKER_HUB_REPO}:latest)
                    docker tag \$IMAGE_ID ${DOCKER_HUB_REPO}:${tag}
                """

                echo "Docker image built and tagged successfully: ${DOCKER_HUB_REPO}:${tag}"

            } catch (Exception e) {
                error "Docker build failed: ${e.message}"  // Explicitly fail if Docker build fails
                    }
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                script {
                    echo 'Running Trivy security scan on the Docker image...'

                    // Run Trivy scan for vulnerabilities in the Docker image
                    def scanResult = sh(script: "trivy image ${DOCKER_HUB_REPO}:${DOCKER_TAG}", returnStatus: true)

                    // Fail the build if vulnerabilities are found (returnStatus != 0 means issues were detected)
                    if (scanResult != 0) {
                        error 'Trivy scan found vulnerabilities in the Docker image!'
                    } else {
                        echo 'Trivy scan passed: No vulnerabilities found.'
                    }
                }
            }
        }

        stage('Push Image to DockerHub') {
            steps {
                input message: 'Approve Deployment?', ok: 'Deploy'  // Manual approval for deployment
                script {
                    echo 'Pushing Docker image to DockerHub...'

                    try {
                        // Manually login to Docker Hub using the credentials
                        withCredentials([usernamePassword(credentialsId: "${DOCKER_HUB_CREDENTIALS_ID}", usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                            sh '''
                                echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
                            '''
                        }

                        // Push the Docker image to Docker Hub with static tag "version-1"
                        sh "docker push ${DOCKER_HUB_REPO}:${DOCKER_TAG}"
                        echo "Docker image pushed successfully: ${DOCKER_HUB_REPO}:${DOCKER_TAG}"

                    } catch (Exception e) {
                        error "Docker push failed: ${e.message}"  // Explicitly fail if push fails
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()  // Clean workspace after pipeline execution
        }
        success {
            echo 'Pipeline executed successfully!'
        }
        failure {
            echo 'Pipeline failed.'
        }
    }
}
