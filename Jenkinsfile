pipeline {
    agent { label 'jenkins-jenkins-agent' }

    environment {
        IMAGE_TAG = "${env.GIT_COMMIT.take(7)}"
        IMAGE_NAME = "salehhedayati/currencyconverter"
        IMAGE = "${IMAGE_NAME}:${IMAGE_TAG}"
//         APIHOST = credentials('APIHOST')  // optional, for your tests
//         APIKEY = credentials('APIKEY')    // optional, for your tests
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-cred') // Jenkins credentials ID
        DOCKER_HOST = 'tcp://localhost:2375'
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
                container('dind') {
                    script {
                        // Short commit hash for tagging
                        sh 'git config --global --add safe.directory "$(pwd)"'
//                         def IMAGE_TAG = GIT_COMMIT.take(7)
                        def shortCommit = sh(returnStdout: true, script: "git rev-parse --short=7 HEAD").trim()
                        env.IMAGE_TAG = shortCommit // ✅ set globally
                        def latestTag = "latest"

                        withCredentials([usernamePassword(credentialsId: 'dockerhub-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                            // Wait a few seconds for Docker daemon to be fully ready
                            sh """
                                echo "Waiting for Docker daemon to start..."
                                sleep 5
                                docker info
                                echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                                docker build -t $IMAGE_NAME:$IMAGE_TAG .
                                docker tag $IMAGE_NAME:$IMAGE_TAG $IMAGE_NAME:$latestTag
                                docker push $IMAGE_NAME:$IMAGE_TAG
                                docker push $IMAGE_NAME:$latestTag
                            """
                        }
                    }
                }
            }
        }

        stage('Update Deployment Manifest') {
            steps {
                sh """
                    git config user.name "Jenkins CI"
                    git config user.email "jenkins@local"
                    git pull --rebase origin main
                    sed -i 's|image: ${IMAGE_NAME}:.*|image: ${IMAGE_NAME}:${IMAGE_TAG}|' k8s/base/deployment.yaml
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