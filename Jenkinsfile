pipeline {
    agent any
    tools {
        maven "MAVEN3"
        jdk "OracleJDK8"
    }

    environment {
        SNAP_REPO = 'vprofile-snapshot'
        SONARSERVER = 'sonarserver'
        SONARSCANNER = 'sonarscanner'
        registry = "sachinettigi/vproappdock"
        registryCredential = 'dockerhub'

    }

     stages {
        stage('Build'){
            steps {
                sh 'mvn install -DskipTests'
            }
            post {
                success {
                    echo "Now Archiving"
                    archiveArtifacts artifacts: '**/*.war'
                }
            }
        }

        stage('Test'){
            steps {
                sh 'mvn test'
            }
        }

        stage('Checkstyle Analysis'){
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
        }

        stage('Sonar Analysis') {

              environment {
             scannerHome = tool "${SONARSCANNER}"
            }

          steps {
            withSonarQubeEnv("${SONARSERVER}") {
               sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile-repo \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }
            }
        }

        stage('Quality Gates') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
               waitForQualityGate abortPipeline: true
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

        stage('Upload App Image') {
            steps {
                script {
                    docker.withRegistry('', registryCredential ) {
                        dockerImage.push("V$BUILD_NUMBER")
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Remove Unused docker image') {
            steps{
                sh "docker rmi $registry:v$BUILD_NUMBER"
            }
        }

        stage('kubernetes Deploy') {
            agent {label 'KOPS'}
              steps {
                sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${registry}:V${BUILD_NUMBER} --namespace prod"
              }
        }

    }
}