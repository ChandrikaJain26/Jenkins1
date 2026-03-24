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
        // Save the jar + Dockerfile + anything Docker needs
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

          echo "Checking application URL"
          curl -f http://localhost:8081/ || exit 1

          echo "Functional testing passed"

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
         echo "Starting application for performance testing"

         nohup java -jar target/*.jar --server.port=8082 > perf.log 2>&1 &
         APP_PID=$!

         sleep 20

         echo "Running performance test (Apache Benchmark)"
         ab -n 200 -c 20 http://localhost:8082/ || exit 1
         
         echo "Performance testing completed"

         kill $APP_PID
       '''
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
