pipeline {
  agent none

  stages {

    stage('Checkout') {
      agent { label 'build' }
      steps {
        git branch: 'main', url: 'https://github.com/ChandrikaJain26/Jenkins1.git'
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
        stash name: 'app', includes: 'Dockerfile,target/*.jar', allowEmpty: false
      }
    }

    stage('Functional Testing') {
      agent { label 'build' }
      environment {
        JAVA_HOME = "/usr/lib/jvm/java-17-openjdk-amd64"
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
      }
      steps {
        sh '''
          echo "Starting application for functional testing"

          nohup java -jar target/*.jar --server.port=8081 > app.log 2>&1 &
          APP_PID=$!

          sleep 20

          curl -f http://localhost:8081/ || exit 1

          kill $APP_PID
        '''
      }
    }

    stage('Performance Testing') {
      agent { label 'build' }
      environment {
        JAVA_HOME = "/usr/lib/jvm/java-17-openjdk-amd64"
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
      }
      steps {
        sh '''
          nohup java -jar target/*.jar --server.port=8082 > perf.log 2>&1 &
          APP_PID=$!

          sleep 20

          ab -n 200 -c 20 http://localhost:8082/ || exit 1

          kill $APP_PID
        '''
      }
    }

    stage('Sonar Scan') {
      agent { label 'build' }
      environment {
        JAVA_HOME = "/usr/lib/jvm/java-17-openjdk-amd64"
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
      }
      steps {
        withSonarQubeEnv('SonarCloud') {
          sh '''
            mvn -B sonar:sonar \
              -Dsonar.projectKey=jenkins1 \
              -Dsonar.organization=chandrikajain26
          '''
        }
      }
    }

    stage('Docker Build') {
      agent { label 'docker' }
      steps {
        unstash 'app'
        sh '''
          docker build -t petclinic:${BUILD_NUMBER} .
        '''
      }
    }

    stage('Nexus Push') {
      agent { label 'docker' }
      environment {
        NEXUS_REGISTRY = "localhost:8083"
        IMAGE_NAME     = "petclinic"
        IMAGE_TAG      = "${BUILD_NUMBER}"
      }
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'nexus-creds',
          usernameVariable: 'NEXUS_USER',
          passwordVariable: 'NEXUS_PASS'
        )]) {
          sh '''
            echo "$NEXUS_PASS" | docker login $NEXUS_REGISTRY -u "$NEXUS_USER" --password-stdin
            docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${NEXUS_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
            docker push ${NEXUS_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
          '''
        }
      }
    }

  }
}
