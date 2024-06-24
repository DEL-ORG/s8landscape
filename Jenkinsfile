pipeline {
    agent any

     triggers {
        githubPush()
    }
    options {
        timeout(time: 1, unit: 'MINUTES') // Timeout for the entire pipeline
        buildDiscarder(logRotator(numToKeepStr: '7')) // Discard old builds to save disk space
        disableConcurrentBuilds() // Ensures that only one build can run at a time
        timestamps() // Adds timestamps to the console output
        skipDefaultCheckout() // Skips the default checkout of source code, useful if you're doing a custom checkout
        retry(3) // Automatically retries the entire pipeline up to 3 times if it fails
    }
    environment {
        DOCKER_HUB_USERNAME = "s8kevinaf02"
        landscape = "landscape-app-01"
        DOCKER_CREDENTIAL_ID = 's8kevinaf02-dockerhub-token'
    }

    parameters {
        string(name: 'BRANCH_NAME', defaultValue: 'main', description: 'Branch to build')
        string(name: 'IMAGE_NAME', defaultValue: '', description: 'Docker image name')
        string(name: 'CONTAINER_NAME', defaultValue: '', description: 'Docker container name')
        string(name: 'PORT_ON_DOCKER_HOST', defaultValue: '', description: 'Port on Docker host')
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
        stage('SonarQube analysis') {
            steps {
                script {
                    dir("${WORKSPACE}") {
                        docker.image("s8kevinaf02/s8landscape:latest").inside('-u 0:0') {
                            withSonarQubeEnv('SonarScanner') {
                                sh """
                                    ls -l 
                                    pwd
                                    sonar-scanner -v
                                    sonar-scanner
                                """
                            }
                        }
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
                        docker build -t ${env.DOCKER_HUB_USERNAME}/app-01:${BUILD_NUMBER} -f landscape.Dockerfile .
                        docker images
                    """
                }
            }
        }
        stage('Login into') {
            steps {
                script {
                    // Login to Docker Hub
                    withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIAL_ID}", 
                    usernameVariable: 'DOCKER_USERNAME', 
                    passwordVariable: 'DOCKER_PASSWORD')]) {
                        // Use Docker CLI to login
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

def customFunction() {
    sh """
        ls -l
        pwd
        uptime
    """
}

def sanity_check() {
    if (params.BRANCH_NAME.isEmpty()) {
        echo "The parameter BRANCH_NAME is not set"
        sh 'exit 2'
    }
}
