pipeline {
    agent {
        label 'built-in'  // Explicitly use built-in node
    }
    
    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        skipStagesAfterUnstable()
        disableConcurrentBuilds()
    }
    
    environment {
        APP_NAME = 'todo-app'
        IMAGE_TAG = "${BUILD_NUMBER}"
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_REPO = 'gpjtech045'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git credentialsId: 'my-ssh-cred', 
                url: 'git@github.com:gpj-045/cicd-end-to-end.git',
                branch: 'main'
                
                stash includes: '**', name: 'source-code'
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                unstash 'source-code'
                
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    script {
                        try {
                            sh '''
                                docker build -t ${DOCKER_REGISTRY}/${DOCKER_REPO}/${APP_NAME}:${IMAGE_TAG} .
                                docker tag ${DOCKER_REGISTRY}/${DOCKER_REPO}/${APP_NAME}:${IMAGE_TAG} ${DOCKER_REGISTRY}/${DOCKER_REPO}/${APP_NAME}:latest
                                
                                echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                                docker push ${DOCKER_REGISTRY}/${DOCKER_REPO}/${APP_NAME}:${IMAGE_TAG}
                                docker push ${DOCKER_REGISTRY}/${DOCKER_REPO}/${APP_NAME}:latest
                                
                                echo "✅ Docker build and push completed successfully"
                            '''
                        } catch (Exception e) {
                            echo "❌ Docker build failed: ${e.getMessage()}"
                            throw e
                        } finally {
                            sh 'docker logout || true'
                        }
                    }
                }
            }
        }
        
        stage('Update K8S Manifests') {
            steps {
                git credentialsId: 'my-ssh-cred', 
                url: 'git@github.com:gpj-045/cicd-demo-manifests-repo.git',
                branch: 'main'
                
                sshagent(credentials: ['my-ssh-cred']) {
                    sh '''
                        echo "📋 Current deploy.yaml:"
                        cat deploy.yaml
                        
                        echo "🔄 Updating image tag to ${IMAGE_TAG}"
                        sed -i "s|image: ${DOCKER_REPO}/${APP_NAME}:.*|image: ${DOCKER_REPO}/${APP_NAME}:${IMAGE_TAG}|" deploy.yaml
                        
                        echo "✅ Updated deploy.yaml:"
                        cat deploy.yaml
                        
                        git config user.email "tech.gjadhav045@gmail.com"
                        git config user.name "gpj-045"
                        
                        if git diff --quiet; then
                            echo "⚠️  No changes detected in deploy.yaml"
                        else
                            git add deploy.yaml
                            git commit -m "🚀 Update ${APP_NAME} image to ${IMAGE_TAG} [Build: ${BUILD_NUMBER}]"
                            git push origin main
                            echo "✅ Successfully updated manifest repository"
                        fi
                    '''
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        
        success {
            echo "✅ Pipeline completed successfully!"
            echo "🐳 Docker image: ${DOCKER_REGISTRY}/${DOCKER_REPO}/${APP_NAME}:${IMAGE_TAG}"
        }
        
        failure {
            echo "❌ Pipeline failed. Check the logs above for details."
        }
    }
}
