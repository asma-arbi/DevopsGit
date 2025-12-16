pipeline {
    agent any

    tools {
        maven 'M2_HOME'
    }

    environment {
        // ---------- GIT ----------
        IMAGE_TAG = "${env.GIT_COMMIT}"

        // ---------- NEXUS DOCKER REGISTRY ----------
        // ⚠️ Port du Docker (hosted) repo, PAS le port Web Nexus
        NEXUS_REGISTRY = "192.168.33.10:8085"
        DOCKER_IMAGE = "${NEXUS_REGISTRY}/student-app:${IMAGE_TAG}"

        // ---------- JENKINS CREDENTIALS ----------
        // Credentials Jenkins de type Username/Password
        // user: admin | password: mot de passe Nexus
        DOCKER_CREDENTIALS = "nexus-docker"

        // ---------- KUBERNETES ----------
        K8S_NAMESPACE = "devops"
        KUBECONFIG = "/var/lib/jenkins/.kube/config"

        // ---------- SONARQUBE ----------
        SONAR_PROJECT_KEY = "student-app"
        SONAR_PROJECT_NAME = "Student App"
    }

    stages {

        stage('Checkout Git') {
            steps {
                git branch: 'main', url: 'https://github.com/asma-arbi/DevopsGit.git'
            }
        }

        stage('Build & Unit Tests') {
            steps {
                sh 'mvn clean verify'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh """
                        mvn sonar:sonar \
                          -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                          -Dsonar.projectName="${SONAR_PROJECT_NAME}"
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t ${DOCKER_IMAGE} -f docker/Dockerfile .
                """
            }
        }

        stage('Push Docker Image to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: DOCKER_CREDENTIALS,
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo "$DOCKER_PASS" | docker login ${NEXUS_REGISTRY} -u "$DOCKER_USER" --password-stdin
                        docker push ${DOCKER_IMAGE}
                        docker logout ${NEXUS_REGISTRY}
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                  # Create namespace if not exists
                  kubectl --kubeconfig=${KUBECONFIG} get namespace ${K8S_NAMESPACE} \
                    || kubectl --kubeconfig=${KUBECONFIG} create namespace ${K8S_NAMESPACE}

                  # Deploy MySQL
                  kubectl --kubeconfig=${KUBECONFIG} apply -f kub/mysql-deployment.yaml -n ${K8S_NAMESPACE}

                  # Wait MySQL
                  kubectl wait --for=condition=Ready pod -l app=mysql \
                    -n ${K8S_NAMESPACE} --timeout=180s

                  # Deploy Spring App
                  kubectl --kubeconfig=${KUBECONFIG} apply -f kub/spring-deployment.yaml \
                    -n ${K8S_NAMESPACE}

                  # Update image
                  kubectl --kubeconfig=${KUBECONFIG} set image deployment/student-app \
                    student-app=${DOCKER_IMAGE} -n ${K8S_NAMESPACE}

                  # Force restart pods if needed
                  kubectl --kubeconfig=${KUBECONFIG} delete pod -l app=student-app \
                    -n ${K8S_NAMESPACE} --force --grace-period=0 || true

                  # Rollout status
                  kubectl --kubeconfig=${KUBECONFIG} rollout status deployment/student-app \
                    -n ${K8S_NAMESPACE} --timeout=300s

                  # Show pods
                  kubectl --kubeconfig=${KUBECONFIG} get pods -n ${K8S_NAMESPACE}
                '''
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed!'
        }
    }
}
