pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'salehhedayati/currencyconverter'
        APIHOST = credentials('APIHOST')  // optional, for your tests
        APIKEY = credentials('APIKEY')    // optional, for your tests
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-cred') // Jenkins credentials ID
        IMAGE_TAG = "" // This will be set dynamically
    }

    triggers {
        pollSCM('H/5 * * * *') // check for changes every 5 minutes
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

//         stage('Test') {
//             agent {
//                 docker {
//                     image 'python:3.11'
//                     reuseNode true
//                 }
//             } // Jenkins automatically injects the environment variables into that container
//             steps {
//                 sh '''
//                     python -m pip install --upgrade pip
//                     pip install -r requirements.txt
//                     python -m unittest discover -s tests
//                 '''
//             }
//         }

        stage('Build and Push Docker Image') {
            steps {
                script {
                    def shortCommit = sh(returnStdout: true, script: "git rev-parse --short=7 HEAD").trim()
                    def imageTag = "${shortCommit}"
                    def latestTag = "latest"

                    withCredentials([usernamePassword(credentialsId: 'dockerhub-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh """
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                            docker build -t $DOCKER_IMAGE:$imageTag .
                            docker tag $DOCKER_IMAGE:$imageTag $DOCKER_IMAGE:$latestTag
                            docker push $DOCKER_IMAGE:$imageTag
                            docker push $DOCKER_IMAGE:$latestTag
                        """
                    }
                    // Save image tag for next stage
                    env.IMAGE_TAG = imageTag
                }
            }
        }

        stage('Update Deployment Manifest') {
            steps {
                sh """
                    git config user.name "Jenkins CI"
                    git config user.email "jenkins@local"
                    git pull --rebase origin main
                    sed -i 's|image: ${DOCKER_IMAGE}:.*|image: ${DOCKER_IMAGE}:${IMAGE_TAG}|' k8s/base/deployment.yaml
                    git add k8s/base/deployment.yaml
                    git commit -m "Update image to ${IMAGE_TAG}"
git push origin main
                """
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline completed successfully. Argo CD will auto-sync deployment."
        }
        failure {
            echo "❌ Pipeline failed. Check logs for details."
        }
    }
}