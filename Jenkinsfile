pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_CREDENTIALS_ID = 'docker-hub'
        DOCKER_IMAGE = 'bhushansalunke/pet-clinic'
        DOCKER_TAG = "build-${env.BUILD_NUMBER}"
        GIT_REPO = "bhushansalunke/spring-petclinic"
    }

    stages {
        stage('Checkout Code') {
            steps {
                cleanWs()
                git branch: 'main', credentialsId: 'github-account', url: "https://github.com/${GIT_REPO}.git"
            }
        }

        stage('Build and Test') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Static Code Analysis with SonarQube') {
            steps {
                withSonarQubeEnv('Sonar-server') {
                    sh '''
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=Petclinic \
                        -Dsonar.projectName=Petclinic \
                        -Dsonar.sources=. \
                        -Dsonar.java.binaries=target/classes
                    '''
                }
            }
        }

       stage('Dependency Vulnerability Scan') {
    steps {
        withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
            dependencyCheck additionalArguments: '--scan ./ --format HTML --out . --nvdApiKey ' + NVD_API_KEY + ' --nvdValidForHours 87600 --data /var/lib/jenkins/odc-data',
                             odcInstallation: 'DP-check'
        }
        dependencyCheckPublisher pattern: '**/dependency-check-report.html'
    }
}


        stage('Docker Build and Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: DOCKER_CREDENTIALS_ID, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                        docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                        docker logout
                    '''
                }
            }
        }

        stage('Remote Deployment to Kube-VM') {
            steps {
                sshagent(['kube-ssh']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ditiss@192.168.74.225 \"
                        sed -i 's|image: bhushansalunke/pet-clinic:.*|image: ${DOCKER_IMAGE}:${DOCKER_TAG}|' ~/Desktop/Project/deployment.yml && \
                        kubectl apply -f ~/Desktop/Project/deployment.yml && \
                        kubectl rollout restart deployment petclinic-deployment
                    \"
                    """
                }
            }
        }
    }

    post {
        always {
            echo "Build finished: ${currentBuild.result}"
        }
        success {
            echo "Build and deployment succeeded!"
        }
        failure {
            echo "Build or deployment failed!"
        }
    }
}
