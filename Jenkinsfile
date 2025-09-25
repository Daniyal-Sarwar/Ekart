pipeline {

    agent any 

    tools {
        jdk 'jdk11'
        maven 'maven3'
    }

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
        DEPLOYMENT_NAME = "ekart-deployment"
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

            stage('Compile') {
                steps {
                    script {
                        sh "mvn clean compile"
                    }
                }
            }


            stage('Build with Maven') {
                steps {
                    script {
                        sh "mvn clean package -DskipTests"
                    }
                }
            }

            stage('Build Docker Image') {
                steps {
                    script {
                        sh "docker build -t ${env.BUILD_TAG}  -f docker/Dockerfile ."
                    }
                }
            }

            stage('Push Docker Image') {
                steps {
                    script {
                        withCredentials([usernamePassword(credentialsId: 'docker-registry-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                            sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                            sh "docker push ${env.BUILD_TAG}"
                            sh "docker logout ${env.DOCKER_REPOSITORY}"
                        }
                    }
                }
            }

            stage('Deploy to OpenShift') {
                steps {
                    script {
                        withCredentials([string(credentialsId: 'OCP_JENKINS_TOKEN', variable: 'OCP_TOKEN')]) {
                            sh "oc login --token=${OCP_TOKEN} --server=${env.OCP_CLUSTER_URL}"
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
        }

def waitForDeployment() {
        echo "Waiting for deployments to complete..."
        
        sh """
            oc rollout status deployment/${env.DEPLOYMENT_NAME} -n ${env.OCP_PROJECT} --watch=true

            ekart_deployment_PODS=\$(oc get pods -n ${env.OCP_PROJECT} -l app=ekart -o jsonpath='{.items[*].metadata.name}')

            for POD in \$ekart_deployment_PODS; do
            echo "Waiting for pod \$POD to be in 'Running' state..."
            oc wait --for=condition=Ready pod/\$POD --timeout=120s -n ${env.OCP_PROJECT}
            done

        """
    }
def verifyDeployment() {
            echo "Verifying deployment..."
            
    def podStatus = sh(script: "oc get pods -n ${env.OCP_PROJECT} -l app=ekart -o jsonpath='{.items[*].status.phase}'", returnStdout: true).trim()
            
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
            sh "docker image del \$(docker images -f 'dangling=true' -q) || true"
            sh "docker rmi -f \$(docker images | grep '${DOCKER_REPOSITORY}' | awk '{print \$3}') || true"
        }
