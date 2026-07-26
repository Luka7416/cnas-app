pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        DOCKER_IMAGE = 'lucyapple/php-crud'
        K8S_NAMESPACE = 'cnas-app'
        K8S_DEPLOYMENT = 'php-crud'
        K8S_CONTAINER = 'php-crud'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate Files') {
            steps {
                sh '''
                    set -eu
                    test -f app/Dockerfile
                    test -f app/index.php
                    test -f app/db.php
                    test -f k8s/php-deployment.yaml
                '''
            }
        }

        stage('PHP Syntax Test') {
            steps {
                sh '''
                    set -eu

                    docker run --rm \
                      -v "$WORKSPACE/app:/app:ro" \
                      php:8.2-cli \
                      sh -c 'find /app -type f -name "*.php" -exec php -l {} \\;'
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -eu

                    docker build \
                      -t ${DOCKER_IMAGE}:${BUILD_NUMBER} \
                      -t ${DOCKER_IMAGE}:latest \
                      app
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    sh '''
                        set +x

                        echo "$DOCKER_TOKEN" |
                          docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin

                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                        docker push ${DOCKER_IMAGE}:latest

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig-cnas',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    sh '''
                        set -eu
                        export KUBECONFIG="$KUBECONFIG_FILE"

                        kubectl set image \
                          deployment/${K8S_DEPLOYMENT} \
                          ${K8S_CONTAINER}=${DOCKER_IMAGE}:${BUILD_NUMBER} \
                          -n ${K8S_NAMESPACE}

                        kubectl rollout status \
                          deployment/${K8S_DEPLOYMENT} \
                          -n ${K8S_NAMESPACE} \
                          --timeout=180s
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig-cnas',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    sh '''
                        set -eu
                        export KUBECONFIG="$KUBECONFIG_FILE"

                        kubectl get deployment ${K8S_DEPLOYMENT} \
                          -n ${K8S_NAMESPACE}

                        kubectl get pods \
                          -n ${K8S_NAMESPACE} \
                          -l app=php-crud \
                          -o wide

                        CURRENT_IMAGE=$(kubectl get deployment ${K8S_DEPLOYMENT} \
                          -n ${K8S_NAMESPACE} \
                          -o jsonpath='{.spec.template.spec.containers[0].image}')

                        echo "Current deployed image: $CURRENT_IMAGE"

                        test "$CURRENT_IMAGE" = \
                          "${DOCKER_IMAGE}:${BUILD_NUMBER}"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "CI/CD pipeline completed successfully."
        }

        failure {
            echo "CI/CD pipeline failed. Check the Console Output."
        }

        always {
            sh '''
                docker logout 2>/dev/null || true
            '''
        }
    }
}
