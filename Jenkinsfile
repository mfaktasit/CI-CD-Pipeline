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

        stage("Build & Push Docker Image") {
            steps {
                script {
                    // Retrieve DockerHub token from Jenkins credentials
                    withCredentials([string(credentialsId: 'DeliToken', variable: 'DOCKERHUB_TOKEN')]) {
                        def dockerRegistryUrl = 'https://index.docker.io/v1/'

                        // Build the Docker image
                        docker.withRegistry(dockerRegistryUrl, DOCKERHUB_TOKEN) {
                            docker_image = docker.build "${IMAGE_NAME}:${IMAGE_TAG}"
                        }

                        // Push the Docker image to DockerHub
                        docker.withRegistry(dockerRegistryUrl, DOCKERHUB_TOKEN) {
                            docker_image.push()
                            docker_image.push('latest')
                        }
                    }
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                script {
                    sh 'docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image ${IMAGE_NAME}:${IMAGE_TAG} --no-progress --scanners vuln --exit-code 0 --severity HIGH,CRITICAL --format table'
                }
            }
        }

        stage('Cleanup Artifacts') {
            steps {
                script {
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker rmi ${IMAGE_NAME}:latest"
                }
            }
        }

        stage("Trigger CD Pipeline") {
            steps {
                script {
                    def localJenkinsUrl = 'http://localhost:8080'
                    def localJenkinsJobPath = 'job/gitops-complete-pipeline'
                    def localJenkinsToken = 'gitops-token'

                    sh "curl -v -k --user yasin_devops:${JENKINS_API_TOKEN} -X POST -H 'cache-control: no-cache' -H 'content-type: application/x-www-form-urlencoded' --data 'IMAGE_TAG=${IMAGE_TAG}' '${localJenkinsUrl}/${localJenkinsJobPath}/buildWithParameters?token=${localJenkinsToken}'"
                }
            }
        }
    }

    post {
        failure {
            emailext body: '${SCRIPT, template="groovy-html.template"}',
                    subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Failed",
                    mimeType: 'text/html', to: "mfaktasit@gmail.com"
        }
        success {
            emailext body: '${SCRIPT, template="groovy-html.template"}',
                    subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Successful",
                    mimeType: 'text/html', to: "mfaktasit@gmail.com"
        }
    }
}
