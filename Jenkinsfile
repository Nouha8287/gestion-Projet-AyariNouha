pipeline {
    agent any

    environment {
        // >>> Remplacez par VOTRE pseudo Docker Hub
        DOCKERHUB_USER = 'nouhaayari'
        IMAGE_NAME     = "${DOCKERHUB_USER}/gestion-projet-backend"
        IMAGE_TAG      = "${BUILD_NUMBER}"
        BACKEND_DIR    = 'backend'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
    }

    // Déclenchement automatique à chaque push GitHub (webhook)
    triggers {
        githubPush()
    }

    parameters {
        booleanParam(name: 'WAIT_QUALITY_GATE', defaultValue: false,
                     description: 'Bloquer le pipeline si le Quality Gate SonarQube échoue (nécessite le webhook SonarQube -> Jenkins)')
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                dir(env.BACKEND_DIR) {
                    sh 'chmod +x mvnw'
                    sh './mvnw -B clean package -DskipTests'
                }
            }
        }

        stage('Tests') {
            steps {
                dir(env.BACKEND_DIR) {
                    sh './mvnw -B test'
                }
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: "${BACKEND_DIR}/target/surefire-reports/*.xml"
                }
            }
        }

        stage('Analyse SonarQube') {
            steps {
                dir(env.BACKEND_DIR) {
                    // 'SonarQube' = nom du serveur configuré dans Manage Jenkins > System
                    withSonarQubeEnv('SonarQube') {
                        sh './mvnw -B org.sonarsource.scanner.maven:sonar-maven-plugin:sonar'
                    }
                }
            }
        }

        stage('Quality Gate') {
            when { expression { params.WAIT_QUALITY_GATE } }
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build') {
            steps {
                dir(env.BACKEND_DIR) {
                    sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -t ${IMAGE_NAME}:latest ."
                }
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials',
                                                  usernameVariable: 'DH_USER',
                                                  passwordVariable: 'DH_PASS')]) {
                    sh 'echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin'
                    sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker push ${IMAGE_NAME}:latest"
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
            archiveArtifacts artifacts: "${BACKEND_DIR}/target/*.jar", allowEmptyArchive: true, fingerprint: true
        }
        success {
            echo "Pipeline OK : ${IMAGE_NAME}:${IMAGE_TAG} publiée sur Docker Hub"
        }
        failure {
            echo 'Pipeline en échec : consultez les logs de l\'étape en rouge.'
        }
    }
}
