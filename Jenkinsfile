pipeline {
  agent any
  environment {
    DOCKERHUB_USER = 'kowshik631'
    TAG = "v${BUILD_NUMBER}"
    ENV_FILE = 'C:\\Users\\Admin\\smart-incident-management\\.env'
  }
  stages {
    stage('Checkout') {
      steps { checkout scm }
    }
    stage('Build Backend') {
      steps { dir('backend') { bat 'mvnw.cmd -B clean package -DskipTests' } }
    }
    stage('Docker Build') {
      steps {
        bat 'docker build -t %DOCKERHUB_USER%/ims-backend:%TAG% ./backend'
        bat 'docker build -t %DOCKERHUB_USER%/ims-frontend:%TAG% --build-arg VITE_API_URL=http://localhost:8088 ./frontend'
      }
    }
    stage('Docker Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'U', passwordVariable: 'P')]) {
          bat 'echo %P%| docker login -u %U% --password-stdin'
          bat 'docker push %DOCKERHUB_USER%/ims-backend:%TAG%'
          bat 'docker push %DOCKERHUB_USER%/ims-frontend:%TAG%'
        }
      }
    }
    stage('Deploy') {
      steps {
        withEnv(['DOCKER_CONFIG=C:\\Users\\Admin\\.docker']) {
          bat 'docker compose --env-file "%ENV_FILE%" up -d --build'
        }
      }
    }
  }
}
