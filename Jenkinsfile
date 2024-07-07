pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        THLINUX_CREDS = credentials('thlinux')
        DOCKER_CRED_ID = 'docker'
        IMAGE_NAME = 'thsre/prime'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout Git') {
            steps {
                git branch: 'main', url: 'https://github.com/thdevopssre/Boardgame.git'
            }
        }
        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        stage('Sonarqube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.java.binaries=. \
                        -Dsonar.projectKey=jenkins
                    '''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Maven Build') {
            steps {
                sh 'mvn clean install'
            }
        }
        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('Determine Docker Image Version') {
            steps {
                script {
                    def latestVersion = sh(
                        script: "curl -s https://registry.hub.docker.com/v1/repositories/${IMAGE_NAME}/tags | jq -r '.[].name' | grep -E '^v[0-9]+' | sort -V | tail -n 1",
                        returnStdout: true
                    ).trim()
                    
                    if (latestVersion) {
                        def versionParts = latestVersion.tokenize('.')
                        def majorVersion = versionParts[0].replace('v', '').toInteger()
                        def newVersion = "v${majorVersion + 1}"
                        env.NEW_IMAGE_TAG = newVersion
                    } else {
                        env.NEW_IMAGE_TAG = 'v1'
                    }
                }
            }
        }
        stage('Build and Push to Docker Hub') {
            steps {
                script {
                    withDockerRegistry(credentialsId: DOCKER_CRED_ID) {
                        sh "docker build -t ${IMAGE_NAME}:${NEW_IMAGE_TAG} ."
                        sh "docker tag ${IMAGE_NAME}:${NEW_IMAGE_TAG} ${IMAGE_NAME}:latest"
                        sh "docker push ${IMAGE_NAME}:${NEW_IMAGE_TAG}"
                        sh "docker push ${IMAGE_NAME}:latest"
                    }
                }
            }
        }
        stage('Trivy') {
            steps {
                sh "trivy image ${IMAGE_NAME}:${NEW_IMAGE_TAG} > trivy.txt"
            }
        }
        stage('Remote SSH to Server and Deploy to Container') {
            steps {
                script {
                    def remote = [:]
                    remote.name = 'thlinux'
                    remote.host = '172.16.123.135'
                    remote.allowAnyHosts = true
                    remote.user = env.THLINUX_CREDS_USR
                    remote.password = env.THLINUX_CREDS_PSW

                    sshCommand remote: remote, command: """
                        if [ \$(docker ps -a -f name=prime --format '{{.Names}}') == 'prime' ]; then
                            if [ \$(docker inspect -f '{{.State.Status}}' prime) == 'exited' ]; then
                                docker start prime
                            else
                                docker stop prime
                                docker rm prime
                                docker run -d --name prime -p 8082:8080 ${IMAGE_NAME}:${NEW_IMAGE_TAG}
                            fi
                        else
                            docker run -d --name prime -p 8082:8080 ${IMAGE_NAME}:${NEW_IMAGE_TAG}
                        fi
                    """
                }
            }
        }
    }
    post {
        always {
            sleep 5
        }
    }
}