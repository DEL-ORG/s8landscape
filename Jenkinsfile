pipeline {
    agent any

    triggers {
        githubPush()
    }
    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
        disableConcurrentBuilds()
        timestamps()
        skipDefaultCheckout()
    }
    environment {
        DOCKER_HUB_USERNAME = "s8kevinaf02"
        DOCKER_CREDENTIAL_ID = 's8kevinaf02-dockerhub-token'
    }

    parameters {
        string(name: 'BRANCH_NAME', defaultValue: 'main', description: 'Branch to build')
        string(name: 'IMAGE_NAME', defaultValue: 'land-01', description: 'Docker image name')
        string(name: 'CONTAINER_NAME', defaultValue: 'land-01', description: 'Docker container name')
        string(name: 'PORT_ON_DOCKER_HOST', defaultValue: '2323', description: 'Port on Docker host')
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    git credentialsId: 'jenkins-ssh-agents-private-key',
                        url: 'git@github.com:DEL-ORG/s8landscape.git',
                        branch: "${params.BRANCH_NAME}"
                }
            }
        }

        stage('Sanity Check') {
            steps {
                script {
                    sanity_check()
                }
            }
        }

        stage('Verify Dockerfile Presence') {
            steps {
                script {
                    sh """
                        echo "Checking for Dockerfile in the workspace..."
                        ls -l ${WORKSPACE}
                        ls -l ${WORKSPACE}/sonar-scanner/
                        ls -l ${WORKSPACE}/landscape/
                    """
                }
            }
        }

        stage('Building Sonar Image') {
            steps {
                script {
                    dir("${WORKSPACE}/sonar-scanner") {
                        sh """
                        docker build -t ${env.DOCKER_HUB_USERNAME}/s8landscape:latest .
                        docker images
                        """
                    }
                }
            }
        }
        stage('Building Landscape Application') {
            when {
                expression {
                    params.BRANCH_NAME == 'main'
                }
            }
            steps {
                script {
                    sh """
                        docker build -t ${env.DOCKER_HUB_USERNAME}/app-01:${BUILD_NUMBER} -f ${WORKSPACE}/landscape/landscape.Dockerfile ${WORKSPACE}/landscape/
                        docker images
                    """
                }
            }
        }

        stage('Login into Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIAL_ID}", 
                    usernameVariable: 'DOCKER_USERNAME', 
                    passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh "docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD"
                    }
                }
            }
        }

        stage('Pushing into Docker Hub') {
            steps {
                script {
                    sh """
                        docker push ${env.DOCKER_HUB_USERNAME}/app-01:${BUILD_NUMBER}
                        docker push ${env.DOCKER_HUB_USERNAME}/s8landscape:latest
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Build completed successfully."
        }
        failure {
            echo "Build failed."
        }
        always {
            cleanWs()
        }
    }
}

def sanity_check() {
    if (params.BRANCH_NAME.isEmpty()) {
        echo "The parameter BRANCH_NAME is not set"
        sh 'exit 2'
    }
}
