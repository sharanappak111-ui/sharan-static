
pipeline {
    agent {
        label 'docker-agent'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    triggers {
        githubPush()
    }

    environment {
        APP_NAME = 'static-site'
        DEPLOY_PORT = '8080'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Matrix Build and Test') {
            matrix {
                axes {
                    axis {
                        name 'SERVER'
                        values 'nginx', 'apache'
                    }
                }

                stages {
                    stage('Build Image') {
                        steps {
                            script {
                                if (SERVER == 'nginx') {
                                    sh """
                                        docker build \
                                          -t ${APP_NAME}:nginx-${BUILD_NUMBER} \
                                          .
                                    """
                                } else {
                                    sh """
                                        docker build \
                                          --build-arg BASE_IMAGE=httpd:2.4-alpine \
                                          --build-arg WEB_ROOT=/usr/local/apache2/htdocs/ \
                                          -f Dockerfile.matrix \
                                          -t ${APP_NAME}:apache-${BUILD_NUMBER} \
                                          .
                                    """
                                }
                            }
                        }
                    }

                    stage('Test Image') {
                        steps {
                            script {
                                def testPort = SERVER == 'nginx' ? '18080' : '18081'
                                def imageTag = "${APP_NAME}:${SERVER}-${BUILD_NUMBER}"

                                sh """
                                    docker rm -f test-${SERVER} 2>/dev/null || true

                                    docker run -d \
                                      --name test-${SERVER} \
                                      -p ${testPort}:80 \
                                      ${imageTag}

                                    for i in \$(seq 1 15); do
                                      if curl -fsS http://localhost:${testPort}/; then
                                        break
                                      fi
                                      sleep 2
                                    done

                                    curl -fsS http://localhost:${testPort}/ \
                                      | grep 'Discover a World of Knowledge'

                                    docker rm -f test-${SERVER}
                                """
                            }
                        }
                    }
                }
            }
        }

        stage('Deploy Website') {
            steps {
                sh """
                    docker rm -f ${APP_NAME} 2>/dev/null || true

                    docker run -d \
                      --name ${APP_NAME} \
                      --restart unless-stopped \
                      -p ${DEPLOY_PORT}:80 \
                      ${APP_NAME}:nginx-${BUILD_NUMBER}

                    for i in \$(seq 1 15); do
                      if curl -fsS http://localhost:${DEPLOY_PORT}/ \
                        | grep 'Knowledge Hub'; then
                        exit 0
                      fi
                      sleep 2
                    done

                    docker logs ${APP_NAME}
                    exit 1
                """
            }
        }
    }

    post {
        success {
            slackSend(
                channel: '#devops-alerts',
                color: 'good',
                message: "SUCCESS: ${JOB_NAME} #${BUILD_NUMBER} deployed. ${BUILD_URL}"
            )
        }

        unstable {
            slackSend(
                channel: '#devops-alerts',
                color: 'warning',
                message: "UNSTABLE: ${JOB_NAME} #${BUILD_NUMBER}. ${BUILD_URL}"
            )
        }

        failure {
            slackSend(
                channel: '#devops-alerts',
                color: 'danger',
                message: "FAILED: ${JOB_NAME} #${BUILD_NUMBER}. Check ${BUILD_URL}"
            )
        }

        always {
            echo 'Pipeline execution completed.'
        }
    }
}
