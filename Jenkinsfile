pipeline {
  agent none

  stages {
    stage('Checkout') {
      agent { label 'build' }
      steps {
        git branch: 'main', url: 'https://github.com/<your-username>/<your-repo>.git'
      }
    }

    stage('Maven Build + Unit Test (Java 17)') {
      agent { label 'build' }
      environment {
        JAVA_HOME = "/usr/lib/jvm/java-17-openjdk-amd64"
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
      }
      steps {
        sh '''
          java -version
          mvn -version
          mvn -B clean test package
          ls -al target || true
          ls -al target/*.jar || true
        '''
        // Save the jar + Dockerfile + anything Docker needs
        stash name: 'app', includes: 'Dockerfile,target/*.jar', allowEmpty: false
      }
    }

    stage('Docker Build') {
      agent { label 'docker' }
      steps {
        // Restore files on docker agent workspace
        unstash 'app'
        sh '''
          ls -al
          ls -al target || true
          ls -al target/*.jar || true
          docker build -t petclinic:${BUILD_NUMBER} .
        '''
      }
    }
  }
}
