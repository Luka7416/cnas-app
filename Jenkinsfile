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

        SONAR_PROJECT_KEY = 'cnas-app'
        SONAR_PROJECT_NAME = 'cnas-app'
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

                    echo "Checking required project files..."

                    test -f app/Dockerfile
                    test -f app/index.php
                    test -f app/db.php
                    test -f k8s/php-deployment.yaml

                    echo "Required project files are present."
                '''
            }
        }

        stage('PHP Syntax Test') {
            steps {
                sh '''
                    set -eu

                    echo "Running PHP syntax validation..."

                    docker run --rm \
                      -v "$WORKSPACE/app:/app:ro" \
                      php:8.2-cli \
                      sh -c 'find /app -type f -name "*.php" -exec php -l {} \\;'

                    echo "PHP syntax validation completed."
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv('sonarqube') {
                        sh """
                            set -eu

                            echo "Starting SonarQube static code analysis..."

                            ${scannerHome}/bin/sonar-scanner \
                              -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                              -Dsonar.projectName=${SONAR_PROJECT_NAME} \
                              -Dsonar.sources=app \
                              -Dsonar.sourceEncoding=UTF-8

                            echo "SonarQube analysis submitted."
                        """
                    }
                }
            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        def qualityGate = waitForQualityGate()

                        echo "SonarQube Quality Gate status: ${qualityGate.status}"

                        if (qualityGate.status != 'OK') {
                            error(
                                "SonarQube Quality Gate failed: " +
                                qualityGate.status
                            )
                        }
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -eu

                    echo "Building Docker image..."

                    docker build \
                      -t ${DOCKER_IMAGE}:${BUILD_NUMBER} \
                      -t ${DOCKER_IMAGE}:latest \
                      app

                    echo "Docker image built:"
                    echo "${DOCKER_IMAGE}:${BUILD_NUMBER}"
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
                        set -eu
                        set +x

                        echo "Logging in to Docker Hub..."

                        echo "$DOCKER_TOKEN" |
                          docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin

                        echo "Pushing versioned Docker image..."
                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}

                        echo "Pushing latest Docker image..."
                        docker push ${DOCKER_IMAGE}:latest

                        docker logout

                        echo "Docker images pushed successfully."
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

                        echo "Deploying Docker image to Kubernetes..."

                        kubectl set image \
                          deployment/${K8S_DEPLOYMENT} \
                          ${K8S_CONTAINER}=${DOCKER_IMAGE}:${BUILD_NUMBER} \
                          -n ${K8S_NAMESPACE}

                        echo "Waiting for Kubernetes rolling update..."

                        kubectl rollout status \
                          deployment/${K8S_DEPLOYMENT} \
                          -n ${K8S_NAMESPACE} \
                          --timeout=180s

                        echo "Kubernetes deployment completed."
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

                        echo "Checking Kubernetes Deployment..."

                        kubectl get deployment ${K8S_DEPLOYMENT} \
                          -n ${K8S_NAMESPACE}

                        echo "Checking application Pods..."

                        kubectl get pods \
                          -n ${K8S_NAMESPACE} \
                          -l app=php-crud \
                          -o wide

                        echo "Checking deployed Docker image..."

                        CURRENT_IMAGE=$(kubectl get deployment \
                          ${K8S_DEPLOYMENT} \
                          -n ${K8S_NAMESPACE} \
                          -o jsonpath='{.spec.template.spec.containers[0].image}')

                        echo "Current deployed image: $CURRENT_IMAGE"
                        echo "Expected image: ${DOCKER_IMAGE}:${BUILD_NUMBER}"

                        test "$CURRENT_IMAGE" = \
                          "${DOCKER_IMAGE}:${BUILD_NUMBER}"

                        echo "Deployment verification successful."
                    '''
                }
            }
        }
    }

    post {

        success {
            echo "=============================================="
            echo "CI/CD + DevSecOps pipeline completed successfully."
            echo "SonarQube Quality Gate passed."
            echo "Docker image built and published."
            echo "Kubernetes deployment verified."
            echo "=============================================="
        }

        failure {
            echo "=============================================="
            echo "CI/CD pipeline FAILED."
            echo "A failed validation/security/deployment stage"
            echo "prevented the pipeline from continuing."
            echo "Check Jenkins Console Output for details."
            echo "=============================================="
        }

        always {
            sh '''
                docker logout 2>/dev/null || true
            '''
        }
    }
}
