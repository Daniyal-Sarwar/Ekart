pipeline {

    agent any 

    parameters {

        choice(name: 'ENVIRONMENT', 
        choices: ['dev', 'test', 'prod'], 
        description: 'Select the environment')
    }

    environment {
        DOCKER_REPOSITORY = 'daniyal7860/ekart'

        VERSION = "${env.BUILD_NUMBER}"
        GIT_COMMIT_HASH = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
        GIT_BRANCH = sh(script: 'git rev-parse --abbrev-ref HEAD', returnStdout: true).trim()

        BUILD_TAG = "${DOCKER_REPOSITORY}:${VERSION}-${GIT_COMMIT_HASH}-${GIT_BRANCH}-${ENVIRONMENT}"

        OCP_PROJECT = "${params.ENVIRONMENT}"
        OCP_CLUSTER_URL = "https://api.lab.ocp.lan:6443"
        DEPLOYMENT_NAME = "Ekart-${params.ENVIRONMENT}"
    }

    stages {
        stage('Initialize') {
            steps {
                script {
                        echo "Building for environment: ${params.ENVIRONMENT}"
                        echo "Docker Image Tag: ${env.BUILD_TAG}"
                        echo "OpenShift Project: ${env.OCP_PROJECT}"
                    }
                }
            }

            stage('Checkout Code') {
                steps {
                    checkout scm
                }
            }

            stage('Build Docker Image') {
                steps {
                    script {
                        sh "docker build -t ${env.BUILD_TAG} ."
                    }
                }
            }

            stage('Push Docker Image') {
                steps {
                    script {
                        withCredentials([usernamePassword(credentialsId: 'docker-registry-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                            sh "echo $DOCKER_PASS | docker login ${env.DOCKER_REPOSITORY} -u $DOCKER_USER --password-stdin"
                            sh "docker push ${env.BUILD_TAG}"
                            sh "docker logout ${env.DOCKER_REPOSITORY}"
                        }
                    }
                }
            }

            stage('Deploy to OpenShift') {
                steps {
                    script {
                        withCredentials([usernamePassword(credentialsId: 'ocp-user-pass', usernameVariable: 'OCP_USER', passwordVariable: 'OCP_PASS')]) {
                            sh "oc login ${env.OCP_CLUSTER_URL} --token=${OCP_PASS} --insecure-skip-tls-verify=true"
                            sh "oc project ${env.OCP_PROJECT}"
                            sh """
                                oc set image deployment/${env.DEPLOYMENT_NAME} ekart=${env.BUILD_TAG} --record
                                oc scale deployment/${env.DEPLOYMENT_NAME} --replicas=${getReplicaCount(params.ENVIRONMENT)}
                                oc rollout status deployment/${env.DEPLOYMENT_NAME}
                            """
                            waitForDeployment()
                            verifyDeployment()
                        }
                    }
                }
            }

            post {
                always {
                    cleanWs()
                }
                success {
                    echo "Pipeline completed successfully! Deployed to ${params.ENVIRONMENT} environment."
                }
                failure {
                    echo "Pipeline failed. Please check the logs for details."
                }
    }

    def waitForDeployment() {
    echo "Waiting for deployments to complete..."
    
    sh """
        oc rollout status deployment/${env.DEPLOYMENT_NAME} --watch=true
        oc get pods -n ${env.OCP_PROJECT} -l app=${env.DEPLOYMENT_NAME} -o jsonpath='{.items[*].metadata.name}' | xargs -n1 oc wait --for=condition=Ready=True --timeout=300s -n ${env.OCP_PROJECT} pod/
    """
}
    def verifyDeployment() {
        echo "Verifying deployment..."
        
        def podStatus = sh(script: "oc get pods -n ${env.OCP_PROJECT} -l app=${env.DEPLOYMENT_NAME} -o jsonpath='{.items[*].status.phase}'", returnStdout: true).trim()
        
        if (podStatus.contains("Running")) {
            echo "Deployment verified successfully. All pods are running."
        } else {
            error "Deployment verification failed. Some pods are not running."
        }
    }

    def getReplicaCount(environment) {
    switch(environment) {
        case 'dev':
            return '1'
        case 'test':
            return '2'
        case 'prod':
            return '3'
        default:
            return '1'
    }
}

    def cleanWs() {
        echo "Delete old docker images..."
        sh "docker image prune -f"
    }
}
}