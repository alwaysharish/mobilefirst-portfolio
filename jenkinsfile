pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        BRANCH_NAME = 'main'
        ECR_ACCOUNT = '606891810045'
        ECR_REPO = "${ECR_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com/mobilefirst/portfolio"
        IMAGE_TAG = "${BUILD_NUMBER}"
        ECR_PUBLISH_IMAGE_TAG = "${ECR_REPO}:${IMAGE_TAG}"
        GIT_CREDENTIALS = 'github_token'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Clone App Repository') {
            steps {
                git branch: "${BRANCH_NAME}",
                    credentialsId: "${GIT_CREDENTIALS}",
                    url: "https://github.com/alwaysharish/mobilefirst-portfolio.git"
            }
        }
        stage('Build Docker Image') {
            steps {
                sh """
                    echo "Building Docker image..."
                    docker build -t ${ECR_REPO}:latest .
                    docker tag ${ECR_REPO}:latest ${ECR_PUBLISH_IMAGE_TAG}
                    docker images
                """
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh """
                    echo "Logging in to AWS ECR..."
                    aws ecr get-login-password --region ${AWS_REGION} | \
                    docker login --username AWS --password-stdin ${ECR_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com

                    echo "Pushing image ${ECR_PUBLISH_IMAGE_TAG}"
                    docker push ${ECR_PUBLISH_IMAGE_TAG}
                """
            }
        }

        stage('Update Kubernetes Manifests Repo') {
            steps {
                dir('mobilefirst-manifests') {

                    git branch: 'main',
                        credentialsId: "${GIT_CREDENTIALS}",
                        url: "https://github.com/alwaysharish/mobilefirst-manifests.git"

                    sh """
                        echo "Updating image tag inside kyc/deployment-svc.yaml"

                        # Replace ONLY mobilefirst image line (safe replace)
                        sed -i 's|image: ${ECR_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com/mobilefirst:.*|image: ${ECR_PUBLISH_IMAGE_TAG}|g' portfolio/deployment-svc.yaml
                        echo "Updated file preview:"
                        cat portfolio/deployment-svc.yaml | grep image
                        git config user.email "jenkins@mobilefirst.com"
                        git config user.name "jenkins"
                        git add portfolio/deployment-svc.yaml
                        git commit -m "Update Portfolio image to ${ECR_PUBLISH_IMAGE_TAG}" || echo "No changes to commit"

                        git push https://${GIT_CREDENTIALS}@github.com/alwaysharish/mobilefirst-manifests.git main
                    """
                }
            }
        }
    }
    post {
        success {
            sh """
                echo " Pipeline succeeded!"
                echo "Image pushed: ${ECR_PUBLISH_IMAGE_TAG}"

                docker rmi ${ECR_REPO}:latest ${ECR_PUBLISH_IMAGE_TAG} || true
                docker image prune -f
            """
        }

        failure {
            sh 'echo " Pipeline failed!"'
        }
    }
}
