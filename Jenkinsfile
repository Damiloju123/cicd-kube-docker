pipeline {
    agent any

    environment {
        registry = "damiloju123/vproappdock"
        registryCredential = "dockerhub"
        scannerHome = ""
    }

    stages {
        stage('BUILD') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.war', fingerprint: true
                }
            }
        }

        stage('UNIT TEST') {
            steps {
                sh 'mvn test'
            }
        }

        stage('INTEGRATION TEST') {
            steps {
                sh 'mvn verify -DskipTests'
            }
        }

        stage('CODE ANALYSIS WITH CHECKSTYLE') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }

        stage('CODE ANALYSIS WITH SONARQUBE') {
            steps {
                script {
                    def scannerHome = tool 'mysonarscanner4'
                    withSonarQubeEnv('sonar-pro') {
                        sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile " +
                                "-Dsonar.projectName=vprofile-repo " +
                                "-Dsonar.projectVersion=1.0 " +
                                "-Dsonar.sources=src/ " +
                                "-Dsonar.java.binaries=target/test-classes/ " +
                                "-Dsonar.junit.reportsPath=target/surefire-reports/ " +
                                "-Dsonar.jacoco.reportPaths=target/jacoco.exec " +
                                "-Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml"
                    }
                }
            }
            post {
                always {
                    timeout(time: 10, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }

        stage('Build App Image') {
            steps {
                script {
                    dockerImage = docker.build registry + ":V$BUILD_NUMBER"
                }
            }
        }

        stage('Upload Images') {
            steps {
                script {
                    docker.withRegistry('', registryCredential) {
                        dockerImage.push("V$BUILD_NUMBER")
                        dockerImage.push("latest")
                    }
                }
            }
        }

        stage('Remove Unused Docker Image') {
            steps {
                sh "docker rmi $registry:V$BUILD_NUMBER"
            }
        }

        stage('Kubernetes Deploy') {
            agent {
                label 'KOPS'
            }
            steps {
                sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${registry}:V${BUILD_NUMBER} --namespace prod"
            }
        }
    }
}
