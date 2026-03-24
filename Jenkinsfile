pipeline {
  agent none

  stages {

    stage('Checkout Code') {
      agent { label 'build' }
      steps {
        git branch: 'main',
            url: 'https://github.com/ChandrikaJain26/Jenkins1.git'
      }
    }

    stage('Maven Build & Unit Test (Java 17)') {
      agent { label 'build' }
      environment {
        JAVA_HOME = "/usr/lib/jvm/java-17-openjdk-amd64"
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
      }
      steps {
        sh '''
          java -version
          mvn -version
          mvn clean test package
        '''
      }
    }

    stage('Docker Build') {
      agent { label 'docker' }
      steps {
        sh '''
          docker build -t petclinic:${BUILD_NUMBER} .
        '''
      }
    }

  }
}
