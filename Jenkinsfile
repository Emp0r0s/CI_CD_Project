pipeline{
    agent any
    tools{
        maven "maven"
    }

    environment {
        registry = "emporos/vproapp-cicd"
        registryCredential = "dockerhub"
    }

    stages{
        stage('build'){
            steps {
                sh 'mvn clean install -Dskiptests'
            }
            post{
                success {
                    echo 'now archiving'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        stage('Unit test'){
            steps {
                sh 'mvn test'
            }
        }

        stage('Integration test'){
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }

        stage('Code analysis with Checkstyle'){
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }

        stage('building Image'){
            steps{
                script {
                    dockerImage = docker.build registry + ":$BUILD_NUMBER"
                }
            }
        }

        stage('building Image'){
            steps{
                script {
                    docker.withRegistry ( '', registryCredential) {
                        dockerImage.push($BUILD_NUMBER)
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Remove Unused docker image') {
          steps{
            sh "docker rmi $registry:$BUILD_NUMBER"
          }
        }

        stage('CODE ANALYSIS with SONARQUBE') {

            environment {
                scannerHome = tool 'mysonarscanner4'
            }

            steps {
                withSonarQubeEnv('sonar-pro') {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile-repo \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }

                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Kubernetes Deploy') {
	        agent { label 'KOPS' }
                steps {
                    sh "helm upgrade --install --force vproifle-stack helm/vprofilecharts --set appimage=${registry}:${BUILD_NUMBER} --namespace prod"
            }
        }    
    }
}