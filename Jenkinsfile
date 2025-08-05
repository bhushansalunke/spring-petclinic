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
        DEPLOYMENT_FILE = "deployment/deployment.yml"
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
	stage('Container Vulnerability Scan - Trivy') {
            steps {
                sh """
                    trivy image --exit-code 0 --severity HIGH,CRITICAL ${DOCKER_IMAGE}:${DOCKER_TAG} > trivy-report.txt
                """
                archiveArtifacts artifacts: 'trivy-report.txt', allowEmptyArchive: true
            }
        }


        stage('Update Deployment File') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        git config --global user.email "bhushansalunke55@gmail.com"
                        git config --global user.name "bhushansalunke"
                       
                        sed -i "s|image: .*|image: ${DOCKER_IMAGE}:${DOCKER_TAG}|" ${DEPLOYMENT_FILE}

                        git add ${DEPLOYMENT_FILE}
                        git commit -m "Update deployment image to ${DOCKER_TAG}" || echo "No changes to commit"
                        git push https://${GITHUB_TOKEN}@github.com/${GIT_REPO}.git HEAD:main
                    '''
                }
            }
        }
    }

    post {
        success {
            emailext(
                subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """<p>Build succeeded.</p>
                         <p><a href="${env.BUILD_URL}">View Build</a></p>
                         <p>See attached Dependency-Check and Trivy reports.</p>""",
                mimeType: 'text/html',
                to: 'jadhavprathamesh957@gmail.com',
                attachmentsPattern: '**/dependency-check-report.html, **/trivy-report.txt'
            )
        }

        failure {
            emailext(
                subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """<p><b>Build failed.</b></p>
                         <p><a href="${env.BUILD_URL}">View Build</a></p>
                         <p>See attached reports for error details.</p>""",
                mimeType: 'text/html',
                to: 'jadhavprathamesh957@gmail.com',
                attachmentsPattern: '**/dependency-check-report.html, **/trivy-report.txt'
            )
        }
    }
}
