pipeline {
  agent none

  stages {

    stage('Checkout') {
      agent { label 'build' }
      steps {
        // ✅ FIX 1: Checkout REAL main branch (no detached HEAD)
        checkout([$class: 'GitSCM',
          branches: [[name: '*/main']],
          userRemoteConfigs: [[
            url: 'https://github.com/ChandrikaJain26/Jenkins1.git',
            credentialsId: 'github-pat'
          ]],
          extensions: [
            [$class: 'LocalBranch', localBranch: 'main'],
            [$class: 'CleanBeforeCheckout']
          ]
        ])
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
          set -eux
          java -version
          mvn -version
          mvn -B clean test package
        '''
        stash name: 'app', includes: 'Dockerfile,target/*.jar'
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
          set -eux

          echo "Starting app for functional testing on port 8081"

          nohup java -jar target/spring-petclinic-4.0.0-SNAPSHOT.jar --server.port=8081 > app.log 2>&1 &
          APP_PID=$!

          echo "APP_PID=$APP_PID"
          sleep 2

          echo "Waiting for application to start..."
          for i in {1..15}; do
            if curl -s http://localhost:8081/ >/dev/null; then
              echo "✅ Application is UP"
              break

            fi
            echo "Not up yet... retry $i"
            sleep 5
          done

          # Final check
          curl -f http://localhost:8081/ || (
            echo "❌ Application did not start. Showing logs:";
            tail -n 200 app.log;
            kill $APP_PID;
            exit 1
          )

          kill $APP_PID
        '''
      }
    }

    stage('Performance Testing') {
      agent { label 'build' }
      steps {
        sh '''
          set -eux
          nohup java -jar target/*.jar --server.port=8082 > perf.log 2>&1 &
          APP_PID=$!
          sleep 20
          ab -n 200 -c 20 http://localhost:8082/
          kill $APP_PID
        '''
      }
    }

    stage('Sonar Scan') {
      agent { label 'build' }
      steps {
        withSonarQubeEnv('SonarCloud') {
          sh '''
            set -eux
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
          set -eux
          docker build -t petclinic:${BUILD_NUMBER} .
        '''
      }
    }

    stage('Nexus Push') {
      agent { label 'docker' }
      environment {
        NEXUS_REGISTRY = "localhost:8083"
      }
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'nexus-creds',
          usernameVariable: 'NEXUS_USER',
          passwordVariable: 'NEXUS_PASS'
        )]) {
          sh '''
            set -eux
            echo "$NEXUS_PASS" | docker login $NEXUS_REGISTRY -u "$NEXUS_USER" --password-stdin
            docker tag petclinic:${BUILD_NUMBER} $NEXUS_REGISTRY/petclinic:${BUILD_NUMBER}
            docker push $NEXUS_REGISTRY/petclinic:${BUILD_NUMBER}
          '''
        }
      }
    }

    stage('Update K8s Manifest (GitOps - dsyer/petclinic)') {
      agent { label 'build' }
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'github-pat',
          usernameVariable: 'GIT_USER',
          passwordVariable: 'GIT_PAT'
        )]) {
          sh '''
            set -eux

            git checkout main

            IMAGE_TO_DEPLOY="dsyer/petclinic"
            sed -i "s|^\\( *image: *\\).*|\\1${IMAGE_TO_DEPLOY}|" k8s/petclinic.yml

            rm -f app.log perf.log || true

            # ✅ FIX 3: commit only if file actually changed
            if git diff --quiet k8s/petclinic.yml; then
              echo "No change in manifest, skipping commit"
              exit 0
            fi

            git config user.email "jenkins@local"
            git config user.name "jenkins"
            git add k8s/petclinic.yml
            git commit -m "GitOps: deploy ${IMAGE_TO_DEPLOY} (build ${BUILD_NUMBER})"

            # ✅ FIX 2: push using PAT
            git remote set-url origin https://${GIT_USER}:${GIT_PAT}@github.com/ChandrikaJain26/Jenkins1.git
            git push origin main
          '''
        }
      }
    }

  }
}
