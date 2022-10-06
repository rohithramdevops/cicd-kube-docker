pipeline{
    agent any
    environment {
        registry = "rohithramdevops/vproappcicd"
        registryCredential = 'dockerhub'
    }
    stages {
      stage('BUILD') {
        steps {
          sh 'mvn clean install -DskipTests'
        }
        post {
            success {
                echo 'Now Archiving...'
                archiveArtifacts artifacts: '**/target/*.war'
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
            sh 'mvn verify -DskipUnitTests'
        }
      }
      stage('Code Analysis With checkstyle') {
        steps {
          sh 'mvn checkstyle:checkstyle'
        }
        post {
          success {
              echo 'Generated Analysis Result'
          }
        }
      }
      stage('Code Analysis with SonarQube ') {
        environment {
          scannerHome = tool 'SonarQube4.7'
        }
        steps {
          withSonarQubeEnv('SonarQubeServer') {
            sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
          }
        }
      }
      stage('Quality Gate') {
        steps {
          timeout(time: 1, unit: 'HOURS') {
            // Parameter indicates whether to set pipeline to UNSTABLE if Quality Gate fails
            // true = set pipeline to UNSTABLE, false = don't
            waitForQualityGate abortPipeline: true
          }
        }
      }
      stage('Build App Image') {
        steps {
          script {
            dockerImage = docker.build( registry + ":V$BUILD_NUMBER")
          }
        }
      }
      stage('Upload App Image') {
        steps {
          script {
            docker.withRegistry('', registryCredential) {
                dockerImage.push("V$BUILD_NUMBER")
                dockerImage.push('latest')
            }
          }
        }
      }
      stage('Remove unused Docker Image') {
        steps {
            sh " docker rmi $registry:V$BUILD_NUMBER"
        }
      }
      stage('Kubernetes Deploy') {
        agent {label 'KOPS'}
          steps {
            sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${registry}:V${BUILD_NUMBER} --namespace prod"
          }
      }
    }
}