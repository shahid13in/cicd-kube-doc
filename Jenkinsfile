pipeline {

    agent any

    tools {
        maven "maven3"
    }

    environment {
        registry = "kubeshahid/vproappdock"
        registryCredential = 'dockerhub'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main',credentialsId: 'git-cred', url: 'https://github.com/shahid13in/cicd-kube-doc.git'
            }
        }

        stage('Build') {
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

     stage('Unit Test') {
    steps {
        script {
            try {
                sh 'mvn test'
            } catch (Exception e) {
                echo "Unit tests failed. Check the Surefire reports for details."
                error("Unit tests failed")
            }
        }
    }
}

        stage('Integration Test') {
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }

        stage('Code Analysis with Checkstyle') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }

        stage('Code Analysis with SonarQube') {
            environment {
                scannerHome = tool 'sonarscanner4'
            }
            steps {
                withSonarQubeEnv('sonar-pro') {
                    sh '''${scannerHome}/bin/sonar-scanner \
                       -Dsonar.projectKey=vprofile \
                       -Dsonar.projectName=vprofile-repo \
                       -Dsonar.projectVersion=1.0 \
                       -Dsonar.sources=src/main/java \
                       -Dsonar.java.binaries=target/classes \
                       -Dsonar.junit.reportsPath=target/surefire-reports \
                       -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                       -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }

                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build App Image') {
            steps {
                script {
                    dockerImage = docker.build("${registry}:V${BUILD_NUMBER}")
                }
            }
        }

        stage('Upload Image') {
            steps {
                script {
                    docker.withRegistry('', registryCredential) {
                        dockerImage.push("V${BUILD_NUMBER}")
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Remove Unused Docker Image') {
            steps {
                sh "docker rmi ${registry}:V${BUILD_NUMBER}"
            }
        }

        stage('Kubernetes Deploy') {
            agent { label 'KOPS' }
            steps {
                sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${registry}:V${BUILD_NUMBER} --namespace prod"
            }
        }
    }
}
