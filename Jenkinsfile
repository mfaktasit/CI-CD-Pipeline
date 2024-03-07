pipeline {
    agent any

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        APP_NAME = "complete-prod-pipeline"
        RELEASE = "1.0.0"
        DOCKER_USER = "mfaktas"
        // DOCKER_PASS = 'dockerhub'
        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
        JENKINS_API_TOKEN = credentials("JENKINS_API_TOKEN")
        JAVA_HOME = tool 'Java17'
    }

    stages {
        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
            steps {
                git branch: 'main', url: 'https://github.com/mfaktasit/CI-CD-Pipeline'
            }
        }

        stage("Build Application") {
            steps {
                sh "mvn clean package"
            }
        }

        stage("Test Application") {
            steps {
                sh "mvn test"
            }
        }

        stage("Code Coverage with JaCoCo") {
            steps {
                sh "mvn jacoco:prepare-agent test jacoco:report"
                archiveArtifacts 'target/site/jacoco/index.html'
            }
        }

        stage('Build App Docker Image') {
            steps {
                echo 'Building App Image'                
                sh 'docker build --force-rm -t "$IMAGE_NAME" -f ./Dockerfile .'
                sh 'docker image ls'
            }
       }

        stage('Push Image to Dockerhub Repo') {
            steps {
                echo 'Pushing App Image to DockerHub Repo'
                withCredentials([string(credentialsId: 'DeliToken', variable: 'DOCKERHUB_TOKEN')]) {
                sh 'docker login -u $DOCKER_USER -p $DOCKERHUB_TOKEN'
                sh 'docker push "$IMAGE_NAME"'
                
            }
          }
        }

        stage("Trivy Scan") {
            steps {
                script {
                    sh 'docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image ${IMAGE_NAME} --no-progress --scanners vuln --exit-code 0 --severity HIGH,CRITICAL --format table'
                }
            }
        }

        stage('Cleanup Artifacts') {
            steps {
                script {
                    sh "docker rmi ${IMAGE_NAME}"
                }
            }
        }

        stage("Trigger CD Pipeline") {
            steps {
                script {
                    def localJenkinsUrl = 'http://localhost:8080'
                    def localJenkinsJobPath = 'job/gitops-complete-pipeline'
                    def localJenkinsToken = 'gitops-token'

                    sh "curl -v -k --user Fatih:${JENKINS_API_TOKEN} -X POST -H 'cache-control: no-cache' -H 'content-type: application/x-www-form-urlencoded' --data 'IMAGE_TAG=${IMAGE_TAG}' '${localJenkinsUrl}/${localJenkinsJobPath}/buildWithParameters?token=${localJenkinsToken}'"
                }
            }
        }
    }

post {
    always {
        emailext (
            subject: "Pipeline Status: ${BUILD_NUMBER}",
            body: '''<html>
                    <body>
                    <p>Build Status: ${BUILD_STATUS}</p>
                    <p>Build Number: ${BUILD_NUMBER}</p>
                    <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                    </body>
                    </html>''',
            to: 'mfaktasit@gmail.com',
            from: 'jenkins@noreplay',
            replyTo: 'mfaktasit@gmail.com',
            mimeType: 'text/html'
        )
        }
    }
}
