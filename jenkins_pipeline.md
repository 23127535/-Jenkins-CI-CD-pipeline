pipeline {
agent any

    environment {
        DOCKERHUB_USERNAME = '23127535'

        IMAGE_NAME = 'my-simple-webapp'

        DOCKER_CRED_ID = 'dockerhub-login'

        GIT_REPO = 'https://github.com/23127535/-Jenkins-CI-CD-pipeline'
    }

    stages {
        stage('1. Git pull') {
            steps {
                echo '=== Bat dau lay code tu Github ==='
                git branch: 'main', url: "${GIT_REPO}"
            }
        }

        stage('2. Build Image and publish to DockerHub') {
            steps {
                echo '=== Build Docker Image ==='
                script {
                    sh "docker build -t ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${BUILD_NUMBER} ."


                    sh "docker build -t ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:latest ."
                }

                echo '=== Login & Push to DockerHub ==='
                withCredentials([usernamePassword(credentialsId: "${DOCKER_CRED_ID}", usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh "echo $PASS | docker login -u $USER --password-stdin"

                    sh "docker push ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${BUILD_NUMBER}"
                    sh "docker push ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:latest"
                }
            }
        }

        stage('3. Deploy') {
            steps {
                echo '=== Deploying to Docker Container ==='
                script {
                    sh "docker stop web-server-container || true"
                    sh "docker rm web-server-container || true"

                    sh """
                        docker run -d \
                        -p 8090:80 \
                        --name web-server-container \
                        ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:latest
                    """
                }
            }
        }
    }
    //clean build
    post {
        always {
            echo '=== Pipeline da ket thuc ==='
            sh "docker rmi ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${BUILD_NUMBER} || true"
        }
    }

}
