pipeline {
    agent { label 'Jenkins-Agent' }

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        APP_NAME = "register-app-pipeline"
        RELEASE = "1.0.0"

        DOCKER_USER = "ashfaque9x"
        DOCKER_PASS = 'dockerhub'

        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
        IMAGE_TAG  = "${RELEASE}-${BUILD_NUMBER}"

        JENKINS_API_TOKEN = credentials("JENKINS_API_TOKEN")
    }

    stages {

        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
            steps {
                git branch: 'main',
                    credentialsId: 'github-ssh',
                    url: 'git@github.com:Ashfaque-9x/register-app.git'
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

        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv(credentialsId: 'jenkins-sonarqube-token') {
                    sh "mvn sonar:sonar"
                }
            }
        }

        stage("Quality Gate") {
            steps {
                waitForQualityGate abortPipeline: false,
                                   credentialsId: 'jenkins-sonarqube-token'
            }
        }

        stage("Build & Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('', DOCKER_PASS) {
                        def dockerImage = docker.build("${IMAGE_NAME}")
                        dockerImage.push("${IMAGE_TAG}")
                        dockerImage.push("latest")
                    }
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                sh """
                docker run --rm \
                -v /var/run/docker.sock:/var/run/docker.sock \
                aquasec/trivy image ${IMAGE_NAME}:latest \
                --no-progress \
                --scanners vuln \
                --severity HIGH,CRITICAL \
                --exit-code 0
                """
            }
        }

        stage("Cleanup Artifacts") {
            steps {
                sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
                sh "docker rmi ${IMAGE_NAME}:latest || true"
            }
        }

        stage("Trigger CD Pipeline") {
            steps {
                sh """
                curl -k -X POST \
                --user clouduser:${JENKINS_API_TOKEN} \
                --data IMAGE_TAG=${IMAGE_TAG} \
                http://ec2-13-232-128-192.ap-south-1.compute.amazonaws.com:8080/job/gitops-register-app-cd/buildWithParameters?token=gitops-token
                """
            }
        }
    }

    post {
        success {
            emailext(
                subject: "${env.JOB_NAME} - Build #${env.BUILD_NUMBER} - SUCCESS",
                body: '''${SCRIPT, template="groovy-html.template"}''',
                mimeType: 'text/html',
                to: "hm538974@gmail.com"
            )
        }

        failure {
            emailext(
                subject: "${env.JOB_NAME} - Build #${env.BUILD_NUMBER} - FAILED",
                body: '''${SCRIPT, template="groovy-html.template"}''',
                mimeType: 'text/html',
                to: "hm538974@gmail.com"
            )
        }
    }
}
