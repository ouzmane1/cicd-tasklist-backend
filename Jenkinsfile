pipeline {
    agent any

    environment {
        DOCKER_IMAGE     = "ouzmane1/tasklist-backend"
        DOCKER_TAG       = "${BUILD_NUMBER}"
        SONAR_HOST_URL   = "https://sonarqube.cicd.kits.ext.educentre.fr"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                echo "Checkout du dépôt backend — build #${BUILD_NUMBER}"
            }
        }

        stage('Install') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Test') {
            steps {
                sh 'npm run test:coverage'
            }
            post {
                always {
                    junit 'reports/junit.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'SONAR_TOKEN')]) {
                    sh """
                        sonar-scanner \
                            -Dsonar.host.url=${SONAR_HOST_URL} \
                            -Dsonar.token=${SONAR_TOKEN}
                    """
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} -t ${DOCKER_IMAGE}:latest ."
            }
        }

        stage('Trivy Scan') {
            steps {
                sh """
                    trivy image \
                        --severity CRITICAL,HIGH \
                        --format table \
                        --exit-code 0 \
                        ${DOCKER_IMAGE}:${DOCKER_TAG}
                """
            }
        }

        stage('SBOM Generation') {
            steps {
                sh """
                    trivy image \
                        --format spdx-json \
                        --output sbom-spdx.json \
                        ${DOCKER_IMAGE}:${DOCKER_TAG}
                """
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'DOCKERHUB_CREDENTIALS',
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    sh """
                        echo ${DOCKER_PASSWORD} | docker login -u ${DOCKER_USERNAME} --password-stdin
                        docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                        docker push ${DOCKER_IMAGE}:latest
                    """
                }
            }
        }

    }

    post {
        always {
            archiveArtifacts artifacts: 'sbom-spdx.json', allowEmptyArchive: true
            archiveArtifacts artifacts: 'reports/junit.xml', allowEmptyArchive: true
            cleanWs()
        }
        success {
            echo "Pipeline backend terminé avec succès — image: ${DOCKER_IMAGE}:${DOCKER_TAG}"
        }
        failure {
            echo "Pipeline backend en échec — vérifiez les logs ci-dessus."
        }
    }
}
