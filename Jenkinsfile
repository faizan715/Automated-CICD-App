pipeline {
    agent any

    tools {
        maven 'Maven'
    }

    environment {
        APP_NAME = "register-app-pipeline"
        RELEASE = "1.0.0"
        DOCKER_USER = "faizan715"
        DOCKER_CRED_ID = 'docker-hub'
        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
        SONAR_HOST_URL = "http://172.31.22.57:9000"
        JFROG_URL = "http://65.0.128.88:8082/artifactory"
        NOTIFICATION_EMAIL = "mohammedfaizan2261@gmail.com"
    }

    stages {
        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
            steps {
                git branch: 'main', credentialsId: 'github-token-auth', url: 'https://github.com/faizan715/Automated-CICD-App'
            }
        }

        stage("Build & Test Application") {
            steps {
                sh "mvn clean test package"
            }
        }

        stage("SonarQube Analysis") {
            steps {
                script {
                    withSonarQubeEnv(credentialsId: 'SonarQube-token') { 
                        sh "mvn sonar:sonar -Dsonar.host.url=${SONAR_HOST_URL}"
                    }
                }    
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'SonarQube-token'
                }    
            }
        }

        stage('Artifactory Configuration') {
            steps {
                rtServer (
                    id: "jfrog-server",
                    url: "${JFROG_URL}",
                    credentialsId: "jfrog"
                )

                rtMavenDeployer (
                    id: "MAVEN_DEPLOYER",
                    serverId: "jfrog-server",
                    releaseRepo: "libs-release-local",
                    snapshotRepo: "libs-snapshot-local"
                )

                rtMavenResolver (
                    id: "MAVEN_RESOLVER",
                    serverId: "jfrog-server",
                    releaseRepo: "libs-release",
                    snapshotRepo: "libs-snapshot"
                )      
            }
        }

        stage('Deploy Artifacts') {
            steps {
                rtMavenRun (
                    tool: "Maven",
                    pom: 'webapp/pom.xml',
                    goals: 'clean install',
                    deployerId: "MAVEN_DEPLOYER",
                    resolverId: "MAVEN_RESOLVER"
                )
            }
        }

        stage('Publish Build Info') {
            steps {
                rtPublishBuildInfo (
                    serverId: "jfrog-server"
                )
            }
        }

        stage("Build & Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', DOCKER_CRED_ID) {
                        def docker_image = docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push('latest')
                    }
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                script {
                    sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image ${IMAGE_NAME}:latest --no-progress --scanners vuln --exit-code 0 --severity HIGH,CRITICAL --format table"
                }
            }
        }

        stage('Cleanup Artifacts') {
            steps {
                script {
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
                    sh "docker rmi ${IMAGE_NAME}:latest || true"
                }
            }
        }
    }

    post {
        failure {
            emailext (
                body: '''${SCRIPT, template="groovy-html.template"}''', 
                subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Failed", 
                mimeType: 'text/html',
                to: "${NOTIFICATION_EMAIL}"
            )
        }
        success {
            emailext (
                body: '''${SCRIPT, template="groovy-html.template"}''', 
                subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Successful", 
                mimeType: 'text/html',
                to: "${NOTIFICATION_EMAIL}"
            )
        }
    }
}